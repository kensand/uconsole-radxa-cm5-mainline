# Arch Linux ARM install — uConsole + Radxa CM5 (7.2.9 mainline, no WM)

**What actually worked (2026-10-10).** Switching the microSD base to Arch Linux
ARM, keeping the **7.2.9-dirty mainline kernel** from this repo. Final working
method:

1. **Flash the full RX image** to the SD (proven-good SPL + GPT + U-Boot + a
   booting Debian rootfs — all verbatim from the image).
2. **Offline swap** (uConsole **powered off**, SD in the host's reader): wipe
   the ext4 root partition's *contents* and copy the Arch rootfs in. Never touch
   the live system.
3. Put the **7.2.9 kernel + AIO DTB** in the rootfs's `/boot`, add an
   `extlinux.conf` entry with **`root=PARTUUID=`** (not `root=UUID=`).
4. Boot.

> ## Why the earlier approaches failed (lessons, 2026-10-10)
>
> - **Hand-assembled boot layout (blank card + `parted` GPT + `dd` first 16 MiB
>   SPL): unreliable.** The RX/Radxa image's on-disk GPT is *not* a standard
>   GPT-at-offset-0 (its header LBA fields are scrambled/relocated), and U-Boot
>   does not find the boot partition from a naive `parted` table. Do not
>   hand-replicate it. **Flash the whole image instead.**
> - **Live `rsync --delete` swap of the rootfs on a running system: breaks it.**
>   Replacing `/usr` (libs, binaries) out from under a live Debian system leaves
>   it unable to run most commands (`head`, `ls`, `cat` → "required file not
>   found"). **Do the swap offline** (uConsole off, SD in reader).
> - **`root=UUID=` failed with "cannot open root device".** The UUID was correct
>   (`blkid` confirmed it), but the 7.2.9 kernel couldn't resolve it at boot.
>   **`root=PARTUUID=` works** — it's what the original Debian 7.2.9 setup used,
>   and it's resolved from the GPT by the block layer (more robust than scanning
>   for a filesystem UUID). Use PARTUUID.

> **Rollback:** the Debian state is backed up at
> `~/Work/uconsole-backups/uconsole-debian-7.2.9-backup.tar.gz` (procedure in
> [rx-image-session-log.md §8b](rx-image-session-log.md#8b-rollback-how-to-get-back-to-this-debian-state)).
> The simplest full rollback is re-flashing the RX image (Phase A).

**Host tools:** `dd xz tar lsblk blkid mount find`. No `bsdtar` needed (GNU `tar`
handles the `.tar.gz`).

---

## Phase A — Flash the full RX image (proven-good boot)

### A1. Identify the SD card
```bash
lsblk
export SD=/dev/sdX        # EDIT: the uConsole card, by size. Confirm before wiping.
echo "target: $SD"
```

### A2. Wipe + write the whole image
```bash
IMG=~/Downloads/ClockworkRadxa-Debian-12-Bookworm-lite-6.1.84.img.xz
ls -lh "$IMG"
sudo umount "$SD"* 2>/dev/null
sudo dd if=/dev/zero of="$SD" bs=1M count=8 status=none
sync
xzcat "$IMG" | sudo dd of="$SD" bs=4M status=progress
sync
sudo partprobe "$SD" 2>/dev/null || true
lsblk "$SD"
```

### A3. Boot it (HDMI in) and confirm
Insert SD, cold-start, log in (`clockwork`). Confirm:
```bash
uname -r                 # 6.1.84+ (RX kernel) — EXPECTED
cat /etc/os-release | head -1   # Debian
```
**Don't proceed until it boots.** This is the known-good baseline.

---

## Phase B — Offline swap to Arch (uConsole OFF, SD in the reader)

The RX image's real layout (verified with `blkid`):

| part | size | fs | role |
|---|---|---|---|
| `sda1` | 16M | vfat "config" | U-Boot env |
| `sda2` | 300M | vfat "efi" | **empty** (not used for the kernel) |
| `sda3` | 476G | ext4 "rootfs" | **Debian root + `/boot`** (kernel, DTBs, extlinux.conf) |

Only **`sda3`** matters for the OS swap. U-Boot reads `extlinux.conf` from the
ext4 rootfs's `/boot/extlinux/`.

### B1. Get the Arch rootfs on the host
```bash
cd /tmp && rm -rf archroot && mkdir archroot
wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
tar xzf ArchLinuxARM-aarch64-latest.tar.gz -C /tmp/archroot
ls /tmp/archroot        # bin etc lib usr ...
```

### B2. Mount the rootfs
```bash
export SD=/dev/sda
sudo umount /mnt/boot 2>/dev/null
sudo umount /mnt 2>/dev/null
sudo mount "${SD}3" /mnt
ls -la /mnt/boot/extlinux/extlinux.conf   # must show the RX conf
```

### B3. Wipe the rootfs contents, copy Arch in (zsh-safe)
```bash
# bulletproof wipe (no glob-abort in zsh): removes every top-level entry once
sudo find /mnt -mindepth 1 -maxdepth 1 -exec rm -rf {} +
ls -la /mnt                       # now just . and ..
sudo cp -a /tmp/archroot/. /mnt/
cat /mnt/etc/os-release | head -1   # NAME="Arch Linux ARM"
```

### B4. /boot with the 7.2.9 kernel + DTBs + modules
```bash
A=~/Work/uconsole-cm5-7.2.9
sudo mkdir -p /mnt/boot/extlinux
sudo cp "$A/Image-7.2" /mnt/boot/Image-7.2
sudo cp "$A/rk3588s-radxa-cm5-uconsole-aio.dtb" /mnt/boot/
sudo cp "$A/rk3588s-radxa-cm5-uconsole.dtb" /mnt/boot/
sudo mkdir -p /tmp/mods && sudo tar xf "$A/modules-7.2.tar.gz" -C /tmp/mods
sudo rsync -a /tmp/mods/lib/modules/7.2.9-dirty /mnt/usr/lib/modules/
ls -l /mnt/boot/Image-7.2 /mnt/boot/*.dtb
```

> The Arch rootfs ships its own `/boot` (stock 7.1.6 `Image` + `initramfs-linux.img`);
> we leave those and just add our `Image-7.2`. `extlinux.conf` points at `Image-7.2`.
> The 7.2.9 kernel has ext4/mmc/usb built-in, so it boots **without an initramfs**.

### B5. extlinux.conf — **use PARTUUID, not UUID**
```bash
ROOT_PARTUUID=$(sudo blkid -s PARTUUID -o value "${SD}3")   # ef199cad-...
echo "root PARTUUID: $ROOT_PARTUUID"
sudo tee /mnt/boot/extlinux/extlinux.conf <<EOF
default arch-72-aio
menu title U-Boot menu
prompt 1
timeout 10

label arch-72-aio
        menu label Arch mainline 7.2.9 (AIO v2)
        linux /boot/Image-7.2
        fdt /boot/rk3588s-radxa-cm5-uconsole-aio.dtb
        append root=PARTUUID=$ROOT_PARTUUID rootfstype=ext4 rootwait rw console=ttyFIQ0,1500000n8 console=tty1 consoleblank=0 loglevel=7 panic=10
EOF
cat /mnt/boot/extlinux/extlinux.conf
```

> **Why PARTUUID:** `root=UUID=<fs-uuid>` failed at boot with "cannot open root
> device" even though the UUID was correct. `root=PARTUUID=` (the GPT partition
> UUID) works — it's what the original Debian 7.2.9 setup used.

### B6. display-kick (REQUIRED — or first DSI enable wedges → grey/black screen)
```bash
R=/home/kenny/git/uconsole-radxa-cm5-mainline/runtime
sudo install -m 644 "$R/display-kick.service" /mnt/etc/systemd/system/
sudo mkdir -p /mnt/etc/systemd/system/multi-user.target.wants
# enable by relative symlink (resolves inside the rootfs, on host or at boot):
cd /mnt/etc/systemd/system/multi-user.target.wants
sudo ln -sf ../display-kick.service display-kick.service
cd -
ls -l /mnt/etc/systemd/system/multi-user.target.wants/display-kick.service
```

### B7. Unmount + boot
```bash
sudo umount /mnt; sync
```
Insert SD, **HDMI unplugged** (console → panel, upright), cold-start.
Login **`alarm`/`alarm`** (or `root`, no password).

### B8. Verify
```bash
uname -r                      # 7.2.9-dirty
head -1 /etc/os-release       # Arch Linux ARM
dmesg | grep cwu50            # panel: TXW500170B0-BL (new)
dmesg | grep -i ocp8178
lsblk -d -o NAME,SIZE,TRAN    # check NVMe/eMMC enumeration under 7.2.9
dmesg | grep -iE 'nvme|pcie|LTSSM|Link'
```

### B9. Post-boot
```bash
# pacman 7 sandboxes its downloads with Landlock; our 7.2.9 kernel does NOT
# enable CONFIG_SECURITY_LANDLOCK, so pacman hard-fails with
#   "restricting filesystem access failed because Landlock is not supported"
#   "switching to sandbox user 'alpm' failed"
# Fix: DisableSandbox in the GLOBAL section of /etc/pacman.conf (top of file,
# BEFORE any [repo]/[aur] header — under a section it is "not recognized").
grep -q '^DisableSandbox' /etc/pacman.conf || sed -i '1i DisableSandbox' /etc/pacman.conf
pacman -Syu
pacman -S --needed git base networkmanager mesa   # no WM (per request)
# GPU: panthor is built into the 7.2.9 kernel; mesa gives userspace.
```

> **Why not just enable Landlock in the kernel?** `CONFIG_SECURITY_LANDLOCK=y`
> is the "proper" fix (it's a one-line config), and a Landlock-enabled Image
> builds and *boots*. But the incremental rebuild was the step that correlated
> with a broken boot during this session, so the safe call was to keep the
> known-good kernel and disable pacman's sandbox in userspace instead. If you
> ever rebuild the kernel from scratch (full build script, not a hand `make`),
> add `CONFIG_SECURITY_LANDLOCK=y` and drop the `DisableSandbox` line.

---

## Troubleshooting (what we actually hit)

| Symptom | Cause | Fix |
|---|---|---|
| `bsdtar: command not found` | Debian RX has GNU tar, not libarchive | use `tar xzf` |
| zsh: `no matches found: /mnt/.*` — `rm -rf` didn't run | zsh aborts the whole line if a glob matches nothing | `sudo find /mnt -mindepth 1 -maxdepth 1 -exec rm -rf {} +` |
| `sudo find/ls` returned nothing on the host | the agent's `sudo` had no TTY for the password | run the `sudo` reads yourself; or mount is world-readable so plain `ls /mnt` works |
| `/mnt/boot` looked empty | the FAT `sda2` was mounted over the ext4 rootfs's `boot/` dir | `sudo umount /mnt/boot` first; the real `/boot` is on `sda3` |
| Live swap: `head/clear/pgrep: cannot execute` | replaced `/usr` under a running system | **swap offline** (uConsole off) |
| "cannot open root device" (UUID) | 7.2.9 kernel couldn't resolve `root=UUID=` | use `root=PARTUUID=` |
| backlight on, black screen, fan 100%, boot stalls | kernel hung (usually the root-device issue above) | fix `root=PARTUUID=`; if still stuck, boot with HDMI in and read the console |
| `pacman`: "Landlock is not supported by the kernel" / "switching to sandbox user 'alpm' failed" | pacman 7 sandboxes downloads with Landlock; kernel lacks `CONFIG_SECURITY_LANDLOCK` | add `DisableSandbox` to the **global** section of `/etc/pacman.conf` (top of file, before any `[repo]`/`[aur]` — under a section it's "not recognized"). Do **not** delete the `alpm` user (that breaks it differently). |

## If it doesn't boot
- **U-Boot loads, no kernel / `=>` prompt:** `extlinux.conf` label/path wrong —
  check `linux /boot/Image-7.2` and `fdt /boot/...aio.dtb` exist on the card.
- **"cannot open root device":** use `root=PARTUUID=` (B5); confirm the
  PARTUUID with `blkid` on the reader.
- **Boots, panel dark/grey:** warm-reboot (MIPI timing); confirm `display-kick`
  is present + enabled (B6).
- **Give up:** re-flash the RX image (Phase A) — full rollback to Debian.
