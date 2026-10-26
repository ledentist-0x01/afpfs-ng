# afpfs-ng — modernized build for 64-bit Linux and Apple Time Capsule

A bug-fixed, modernized build of **afpfs-ng 0.8.1**, the Linux client for the
Apple Filing Protocol (AFP). This fork exists so that an Apple Time Capsule
(and other AFP servers) can be mounted on current Debian/Ubuntu systems without
the SMB/CIFS breakage that makes file managers unusable.

> **Why this exists:** The Time Capsule only speaks SMB1. The common fstab
> recipe forces `vers=3.0` (or relies on SMB2/3 negotiation), which the
> Time Capsule cannot answer, producing
> `CIFS: VFS: \\192.168.1.33 SMB response too short (35 bytes)` in dmesg and
> freezing every file manager that touches the mount. AFP is the native protocol
> of the Time Capsule and works reliably.

## Improvements over afpfs-ng 0.8.1

### Build fixes (GCC 14 / modern toolchain)
- Added missing `#include`s (`codepage.h`, `forklist.h`, `midlevel.h`,
  `afp_replies.h`, `afp.h`, `libgen.h`, `sys/time.h`, `<stdlib.h>`,
  `<string.h>`).
- Fixed implicit conversions that GCC 14 turns into errors:
  - `afp_closedt()` now takes `unsigned short *` (was called with a value).
  - `cleartxt_passwd()` signature corrected to 4 arguments.
  - `appledouble_write()` prototype added to `resource.h`.
  - `set_nonunix_perms()`, `remove_opened_fork()`, `test_urls()`, etc.
- Fixed 64-bit pointer truncation: `read()` buffer and `dsi_getstatus_reply`
  pointer arithmetic used `(unsigned int)` casts of pointers, causing `EFAULT`
  on 64-bit systems. This was the actual reason GetStatus always timed out.
- Fixed `pthread_t` / `full_url` (tentative definition) for `-fno-common`.
- `snprintf` size arguments corrected (`PATH_MAX` vs. `AFP_MAX_PATH`).

### Runtime fixes
- **GetStatus now succeeds** on real AFP servers (Time Capsule / netatalk)
  after the 64-bit pointer fix.
- **Non-`FIXED` volumes are accepted** — the Time Capsule reports a
  non-`AFP_VOL_FIXED` signature; the old code refused to mount it.
- **`allow_other` is passed automatically** when running as root, so the FUSE
  mount is accessible under a systemd autofs mountpoint.
- Removed the volume-password / fixed-directory hard failures that aborted
  mounts on Time Capsule volumes.

## Building

```sh
sudo apt-get install build-essential libfuse-dev libgcrypt20-dev \
     libgmp-dev libreadline-dev libncurses-dev dpkg-dev debhelper
./configure --prefix=/usr/local
make
sudo make install
sudo ldconfig
```

Debian packaging (optional):

```sh
dpkg-buildpackage -b -uc -us
sudo dpkg -i ../libafpclient0_*.deb ../afpfs-ng-utils_*.deb ../afpfs-ng_*.deb
```

## Mounting an Apple Time Capsule

`/etc/fstab`:

```
afp://admin:YOURPASSWORD@192.168.1.33/AirPort-Laufwerk /mnt/timecapsule afpfs noauto,x-systemd.automount,x-systemd.mount-timeout=30,_netdev 0 0
```

Install a mount helper (`/sbin/mount.afpfs`):

```sh
#!/bin/sh
spec="$1"; dir="$2"
exec /usr/local/bin/mount_afp "$spec" "$dir"
```

Then:

```sh
sudo systemctl daemon-reload
sudo systemctl start mnt-timecapsule.automount
ls /mnt/timecapsule   # triggers the mount
```

## License
afpfs-ng is released under the GPL. See `COPYING`.

## Credits
Original afpfs-ng by Alex deVries and contributors (0.8.1, 2008-2012).
Modernized for 64-bit Linux and Apple Time Capsule by ledentist (2026).