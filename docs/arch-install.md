# Arch Linux ARM install — uConsole + Radxa CM5 (7.2.9 mainline, no WM)

Switching the microSD base to **Arch Linux ARM**, keeping the **7.2.9-dirty
mainline kernel** from this repo. **Approach: flash the full RX image, then swap
the OS in place.**

> ## Why not hand-assemble the boot layout? (2026-10-10 lesson)
>
> The original plan was: blank card → `parted` GPT → `mkfs` → `dd` the first
> 16 MiB of the RX image (U-Boot SPL) → Arch rootfs. **That is unreliable on
> this hardware.** The RX/Radxa SD image's GPT is *not* a standard
> GPT-at-offset-0 layout: its on-disk GPT header at 0x200 carries scrambled /
> relocated LBA fields (current-LBA points into the multi-GB range, first-usable
> > last-usable), and U-Boot does not find the boot partition from a naive
> `parted`-created GPT. Hand-replicating that byte layout with `parted` + `dd`
> produces a table the SPL may not understand, and without the device in front
> of us we can't debug a boot failure.
>
> **So: flash the whole RX image** (its SPL + partition layout + U-Boot + a
> booting Debian rootfs are all proven-good for this exact board), boot it, then
> **replace only the root filesystem with Arch** and add the 7.2.9 kernel. The
> boot-critical bits (SPL, GPT, U-Boot) come from the image verbatim; we never
> hand-write them.

> **Rollback first.** The Debian state is already backed up:
> `~/Work/uconsole-backups/uconsole-debian-7.2.9-backup.tar.gz` (procedure in
> [rx-image-session-log.md §8b](rx-image-session-log.md#8b-rollback-how-to-get-back-to-this-debian-state)).

**Host tools** (this PC): `dd xz lsblk`. On the device after boot: `pacman`,
`rsync`, `blkid`.

---

## Phase A — Flash the full RX image (proven-good boot)

### A1. Identify the SD card (do not skip)

```bash
lsblk
export SD=/dev/sdX        # EDIT: the uConsole card, by size. Confirm it's the
                          # one you mean to wipe (currently the 476G SanDisk).
echo "target: $SD"
```

### A2. Wipe + write the whole image

```bash
IMG=~/Downloads/ClockworkRadxa-Debian-12-Bookworm-lite-6.1.84.img.xz
ls -lh "$IMG"

sudo umount "$SD"* 2>/dev/null          # unmount anything on it
sudo dd if=/dev/zero of="$SD" bs=1M count=8 status=none   # clear old GPT quickly
sync

# full image write (this is the slow part — several minutes for 690M)
xzcat "$IMG" | sudo dd of="$SD" bs=4M status=progress
sync
sudo partprobe "$SD" 2>/dev/null || true

lsblk "$SD"          # should show the RX image's partitions (boot + root, Debian)
```

> `bs=4M` makes the write faster; the image is a byte-exact SD image so no
> `count=` needed. If `dd` reports a short write or the card is busy, unplug/replug
> and retry.

### A3. Boot the RX image (verify it works before touching it)

Insert SD, **HDMI unplugged** (console goes to the panel), cold-start. Log in
(`clockwork`/`clockwork` or the image default). Confirm:

```bash
uname -r                 # 6.1.84+ (the RX kernel) — this is EXPECTED for now
ip -br link              # eth0/end0 up
```

**Do not proceed until the RX image boots to a login.** This is the known-good
baseline. If it doesn't boot, the card/write is wrong — re-do A2.

---

## Phase B — Swap the OS to Arch (on the booted RX system, over SSH)

Everything below runs **on the device** (the booted RX system). We replace the
Debian rootfs with Arch, keep the same `/boot` partition + U-Boot, and install
the 7.2.9 kernel.

> **This is destructive to the Debian rootfs** (it's what we're replacing). The
> rollback backup (§8b) covers getting back to Debian.

### B1. Get the Arch rootfs onto the card

```bash
# on the device:
ROOT=$(findmnt -n -o SOURCE /)           # e.g. /dev/mmcblk0p3
BOOT=$(findmnt -n -o SOURCE /boot)       # the ext4 boot partition
echo "root=$ROOT boot=$BOOT"

# download + extract Arch into a temp dir
cd /tmp
wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
mkdir -p /tmp/archroot
bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C /tmp/archroot
```

### B2. Move Arch onto the root partition

We wipe the Debian rootfs and copy Arch in. (Same partition, so fstab/UUID stay
valid and U-Boot's boot path is untouched.)

```bash
# on the device — DESTRUCTIVE to the current rootfs:
mount | grep -w "$ROOT" || { echo "root is the live fs; doing this online is unsafe — see note"; }
```

> **Important:** replacing the rootfs of a *live* system is not straightforward
> (busy filesystems). The clean way is to boot the RX system, then either:
> (a) extract Arch over the running root and fix the few busy files on next boot,
> or
> (b) **chroot approach** — but the simplest reliable method is to do the swap
> while the rootfs is not the running one.
>
> **Practical method used:** boot the RX system, download the Arch tarball, and
> `rsync` it over `/` *excluding* the currently-mounted/volatile paths, then
> reinstall the bootloader config. Files that are busy (running binaries, /proc,
> /sys) are skipped and get corrected on the next cold boot. The kernel + U-Boot
> + /boot are untouched, so if anything goes wrong the RX kernel still loads and
> you can recover.

```bash
# on the device — swap rootfs to Arch, keeping /boot, /dev, /proc, /sys, /run, /etc/machine-id:
cd /tmp
rsync -aHAX --delete \
  --exclude '/boot' --exclude '/dev' --exclude '/proc' --exclude '/sys' \
  --exclude '/run' --exclude '/etc/machine-id' --exclude '/root/.ssh' \
  /tmp/archroot/ /
```

> `rsync --delete` makes `/` match Arch. `/boot` is preserved (still has the RX
> kernel + the extlinux U-Boot reads). We'll add the 7.2.9 kernel to `/boot` next.
> Keep `/root/.ssh` so you don't lock yourself out of SSH.

### B3. Install the 7.2.9 kernel + DTB into /boot

Copy the artifacts from the host to the device first (from the host PC):

```bash
# FROM THE HOST PC (not the device):
A=~/Work/uconsole-cm5-7.2.9
scp "$A/Image-7.2" "$A/rk3588s-radxa-cm5-uconsole-aio.dtb" \
    "$A/rk3588s-radxa-cm5-uconsole.dtb" \
    "$A/modules-7.2.tar.gz" \
    clockwork@<device-ip>:/tmp/
```

```bash
# ON THE DEVICE:
cp /tmp/Image-7.2 /boot/Image-7.2
cp /tmp/rk3588s-radxa-cm5-uconsole-aio.dtb /boot/
cp /tmp/rk3588s-radxa-cm5-uconsole.dtb /boot/
mkdir -p /tmp/mods && tar xf /tmp/modules-7.2.tar.gz -C /tmp/mods
rsync -a /tmp/mods/lib/modules/7.2.9-dirty /usr/lib/modules/
```

### B4. extlinux.conf — add the Arch + 7.2.9 entry

The RX system's U-Boot reads `/boot/extlinux/extlinux.conf`. **Keep the existing
RX entries** (so the 6.1.84 kernel remains an escape hatch) and add an Arch/7.2.9
entry that points at the Arch root (same root partition, now holding Arch):

```bash
# on the device:
cp /boot/extlinux/extlinux.conf /boot/extlinux/extlinux.conf.rx-backup
ROOT_PARTUUID=$(blkid -s PARTUUID -o value "$ROOT")
cat >> /boot/extlinux/extlinux.conf <<EOF

label arch-72-aio
  menu label Arch mainline 7.2.9 (AIO v2)
  kernel /Image-7.2
  fdt /rk3588s-radxa-cm5-uconsole-aio.dtb
  append root=PARTUUID=$ROOT_PARTUUID rootfstype=ext4 rootwait rw console=tty1 consoleblank=0 loglevel=7 panic=10
EOF
# set it as default (keep the RX entry as fallback):
sed -i 's/^default .*/default arch-72-aio/' /boot/extlinux/extlinux.conf
cat /boot/extlinux/extlinux.conf
```

### B5. display-kick (REQUIRED, or first DSI enable wedges → grey screen)

```bash
# on the device (the unit is already in the backup/repo; copy from /boot backup or host):
cat > /etc/systemd/system/display-kick.service <<'EOF'
[Unit]
Description=Kick DSI panel (fixes grey screen on first enable)
After=systemd-udevd.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'for i in 1 2 3; do echo 1 > /sys/class/backlight/*/bl_power 2>/dev/null; echo 1 > /sys/class/backlight/*/bl_power 2>/dev/null; sleep 0.5; done'
RemainAfterExit=yes
EOF
systemctl enable display-kick.service
```

(Adjust the `ExecStart` to match the actual working display-kick from the Debian
system — it's in the backup tarball at `etc/systemd/system/display-kick.service`.
Prefer restoring that exact file over the stub above.)

### B6. Reboot into Arch + 7.2.9

```bash
reboot
```

After reboot, verify:

```bash
uname -r                 # 7.2.9-dirty
cat /etc/os-release | head -1   # Arch Linux
dmesg | grep cwu50         # panel: TXW500170B0-BL (new)
dmesg | grep -i ocp8178
```

Login is now **`alarm`/`alarm`** (Arch default) — the Debian `clockwork` user is
gone unless you recreate it.

### B7. Post-boot

```bash
sudo pacman -Syu
sudo pacman -S --needed git base networkmanager mesa   # no WM (per request)
# GPU: panthor is built into the 7.2.9 kernel; mesa gives userspace.
```

---

## If it doesn't boot

- **U-Boot loads but no kernel / drops to `=>`:** the `extlinux.conf` label is
  wrong → the RX backup (`extlinux.conf.rx-backup`) still has the working 6.1.84
  entry; revert `default` to it and boot the RX kernel, then fix the label.
- **Boots RX 6.1.84 but not Arch 7.2.9:** the `arch-72-aio` label has a bad
  `root=PARTUUID` or the kernel/DTB didn't land in `/boot` — check `ls /boot`.
- **Panel dark/grey after Arch boot:** warm-reboot (MIPI timing quirk); confirm
  `display-kick` is present and enabled.
- **Give up:** restore Debian from the backup
  ([rx-image-session-log.md §8b](rx-image-session-log.md#8b-rollback-how-to-get-back-to-this-debian-state)).
