# `rmdir("/.oldproc") => Device or resource busy` with `rootfs("/")` + a private procfs

## Summary

A container built with `Container::new()` (which unshares PID and mounts a private
`/proc`) **and** `rootfs("/")` fails deterministically during setup with:

```
rmdir("/.oldproc") => Device or resource busy (os error 16)
```

(In 1.7.1 the path is `/.oldproc-<uuid>`; in 1.7.0 it's `/.oldproc`. Same bug.)

It is **not** a race and **not** new to 1.7.1 — reproduced 3/3 on both 1.7.0 and
1.7.1. It only bites when the **host `/proc` has a submount**, e.g.
`/proc/sys/fs/binfmt_misc` (present on most desktop/systemd and many Gentoo hosts):

```
$ mount | grep ' /proc'
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
binfmt_misc on /proc/sys/fs/binfmt_misc type binfmt_misc (...)
```

## Root cause

`runc/unshare.rs`:

- Setup binds the host `/proc` to the placeholder **recursively**:
  ```rust
  sys::mount("/proc", &oldproc, MsFlags::MS_BIND | MsFlags::MS_REC)?;   // ~line 114
  ```
  `MS_REC` pulls every `/proc` submount (`.../binfmt_misc`, etc.) into `.oldproc`.
- Teardown (after the fresh procfs is mounted) does a **lazy** unmount then an
  **immediate** rmdir:
  ```rust
  sys::unmount(&oldproc)?;   // umount2(MNT_DETACH) — lazy/recursive but deferred
  sys::rmdir(&oldproc)?;     // EBUSY: .oldproc/sys/fs/binfmt_misc still attached
  ```
  `MNT_DETACH` schedules the subtree to detach once unbusy, but does not complete
  synchronously. The `binfmt_misc` submount is still mounted under
  `.oldproc/sys/fs/`, so `.oldproc` is non-empty and `rmdir` returns `EBUSY`.

Contrast `.oldrootfs` (the `pivot_root` `put_old`, line ~84-85): it has *many*
submounts yet `unmount` + `rmdir` works, because the kernel detaches the
`pivot_root` old-root cleanly. The hand-rolled recursive `/proc` bind doesn't get
that treatment.

## Why `rootfs("/")` specifically

`rootfs("/")` binds the host filesystem, so the container's `/proc` (bound to
`.oldproc`) is the host `/proc` *with* its submounts. A container built with
`rootdir(<prepared_dir>)` (a chroot into a stage tree) does not reproduce this —
which is why e.g. crossdev-stages' `rootdir()` sandbox works while a
`rootfs("/")` container on the same host fails.

## Reproduce

```rust
let status = Container::new()        // PID ns + private procfs
    .rootfs("/")?
    .devfsmount("/dev")
    .command("/bin/true")
    .status()?;                       // -> rmdir("/.oldproc") EBUSY before the command runs
```
On a host where `/proc/sys/fs/binfmt_misc` is mounted.

## Fix

Make the placeholder `rmdir` **non-fatal** (`mount2()`):

```rust
  sys::unmount(&oldproc)?;          // umount2(MNT_DETACH) — lazy/async
- sys::rmdir(&oldproc)?;
+ let _ = sys::rmdir(&oldproc);     // best-effort: see below
```

`unmount` is a lazy `umount2(MNT_DETACH)`, so when the recursive `/proc` bind
carries submounts the placeholder can still be busy and the rmdir races the async
detach (EBUSY). The leftover empty `.oldproc` dir lives in the ephemeral container
mount namespace and is harmless. Failing the whole container on it is wrong.

**Why not just bind non-recursively** (drop `MS_REC`)? Tried first — it doesn't
work: `mount("/proc", ".oldproc", MS_BIND)` without `MS_REC` returns **EINVAL**
(you can't non-recursively bind `/proc`). So the recursive bind is required, and
the fix has to be at teardown.

Alternative considered: recursively unmount the submounts synchronously before the
rmdir — more code, and the submounts (e.g. a locked `binfmt_misc`) may not be
synchronously unmountable from inside the userns. The best-effort rmdir is the
minimal correct fix.

## Status

Fixed in this fork (`runc/unshare.rs` `mount2()`). Verified end-to-end: a
`Container::new().rootfs("/").runctl(RootdirRW)` build container now starts and
runs real package builds on a host with `/proc/sys/fs/binfmt_misc` mounted.

Found while using hakoniwa as the unprivileged-build backend for a Portage package
manager (`em`): the build container must `rootfs("/")` to reach the host toolchain.
