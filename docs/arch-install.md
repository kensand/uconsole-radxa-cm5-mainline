# Arch Linux ARM install — uConsole + Radxa CM5 (7.2.9 mainline, no WM)

Switching the microSD base from the Debian/RX image to **Arch Linux ARM** on a
**blank microSD card**, with the **7.2.9-dirty mainline kernel** and **Radxa's
U-Boot SPL** (written from the stock CLI image). Follows the repo's recommended
Arch path
([kernel-guide.md](kernel-guide.md#step-9--install-the-arch-linux-arm-rootfs))
with two deliberate deviations, called out where they happen:

1. **Blank SD card; Radxa U-Boot SPL written from the stock image.** The CM5
   will not boot an SD without U-Boot in the raw first 16 MiB, so we write the
   guide's Step 8 (first 16 MiB of `radxa-cm5-rpi-cm4-io_bookworm_cli_b3.output.img.xz`,
   which is on the host at `~/Downloads/`). No preservation needed because the
   card starts empty.
2. **Install the 7.2.9 mainline kernel, not the guide's 6.1 build.** We already
   built it and it drives the TXW500170B0-BL panel + AIO v2. No dev-null2019
   overlays — the AIO DTB bakes all of that in.

> **Rollback first.** Before running anything below, the Debian state must be
> backed up (it is: `~/Work/uconsole-backups/uconsole-debian-7.2.9-backup.tar.gz`,
> and the full procedure is in
> [rx-image-session-log.md §8b](rx-image-session-log.md#8b-rollback-how-to-get-back-to-this-debian-state)).
> Do not proceed to Step 2 until that tarball is confirmed on the host.

**Host tools** (this PC): `parted mkfs.ext4 bsdtar wget dd xz rsync` are enough.
The guide's `sgdisk` is not installed; `parted` is used for the GPT label instead.

---

## Step 1 — Identify the SD card (do not skip)

```bash
lsblk
# confirm which /dev/sdX is the uConsole's card (by size, and that it currently
# holds the Debian system: mmcblk-style partitions, the RX labels, etc.)
export SD=/dev/sdX          # EDIT THIS to your real device
echo "target: $SD"
```

Everything below wipes and rewrites `$SD`. **If you get the device wrong you
destroy that disk.** Double-check against `lsblk` and `lsblk -o NAME,SIZE,LABEL`.

## Step 2 — (blank card: nothing to preserve)

We start from a fresh card, so there is no U-Boot to save. (If you *had* chosen
the in-place-card approach instead, you would have preserved the first 16 MiB
before wiping — see the git history of this doc.)

## Step 3 — Partition the card (GPT: 16 MiB gap, /boot, /root)

```bash
sudo wipefs -a "$SD"
sudo parted -s "$SD" mklabel gpt
sudo parted -s "$SD" mkpart boot ext4 16MiB 528MiB
sudo parted -s "$SD" mkpart root ext4 528MiB 100%

sudo mkfs.ext4 -L BOOT "${SD}1"
sudo mkfs.ext4 -L ROOT "${SD}2"

sudo mount "${SD}2" /mnt
sudo mkdir -p /mnt/boot
sudo mount "${SD}1" /mnt/boot
```

## Step 4 — Write U-Boot SPL to the raw start (from the Radxa image)

The first 16 MiB of the Radxa CLI image contains the RK3588S ROM → U-Boot SPL
→ U-Boot proper chain. Write it to the raw card (offset 0, does not touch the
GPT that landed at 16 MiB):

```bash
RADXA_IMG=~/Downloads/radxa-cm5-rpi-cm4-io_bookworm_cli_b3.output.img.xz
ls -lh "$RADXA_IMG"    # confirm it exists (690M)
xzcat "$RADXA_IMG" | sudo dd of="$SD" bs=512 count=32768 conv=notrunc status=progress
sync
```

> `count=32768` × 512 B = 16 MiB. This is byte-identical to the guide's Step 8.
> If you don't have that image on hand, any Radxa CM5 b3 image works — only the
> first 16 MiB is used.

## Step 5 — Install the Arch Linux ARM rootfs

```bash
cd /tmp
wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
sudo bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt
```

Set `fstab` by UUID (root = `${SD}2`, boot = `${SD}1`):

```bash
ROOT_UUID=$(sudo blkid -s UUID -o value "${SD}2")
BOOT_UUID=$(sudo blkid -s UUID -o value "${SD}1")
sudo tee /mnt/etc/fstab <<EOF
UUID=${ROOT_UUID}  /      ext4  rw,noatime  0 1
UUID=${BOOT_UUID}  /boot  ext4  rw,noatime  0 2
EOF
```

## Step 6 — Install the 7.2.9 kernel, DTB, and modules

The artifacts are on this host at `~/Work/uconsole-cm5-7.2.9/` (built from this
repo, see [rx-image-session-log.md §1](rx-image-session-log.md#1-build-on-the-x8664-host)).

```bash
A=~/Work/uconsole-cm5-7.2.9

# kernel image
sudo cp "$A/Image-7.2" /mnt/boot/Image

# the AIO DTB (and the base DTB as a fallback) — we use prebuilt DTBs, no overlays
sudo cp "$A/rk3588s-radxa-cm5-uconsole-aio.dtb" \
       "$A/rk3588s-radxa-cm5-uconsole.dtb" /mnt/boot/

# modules (staged tarball: contains lib/modules/7.2.9-dirty)
mkdir -p /tmp/mods && tar xf "$A/modules-7.2.tar.gz" -C /tmp/mods
sudo rsync -a /tmp/mods/lib/modules/7.2.9-dirty /mnt/usr/lib/modules/
```

**Fallback if the card doesn't get past U-Boot at all** (SPL didn't take, or the
raw write landed wrong): re-run Step 4. The Radxa image's first 16 MiB is the
known-good bootloader for this exact board.

## Step 7 — extlinux.conf

The U-Boot on this card reads `extlinux/extlinux.conf` from the **boot partition**
(that's how the RX image was configured). Write it for the 7.2.9 kernel:

```bash
sudo mkdir -p /mnt/boot/extlinux
sudo tee /mnt/boot/extlinux/extlinux.conf <<'EOF'
timeout 10
default arch-72-aio

label arch-72-aio
  menu label Arch mainline 7.2.9 (AIO v2)
  kernel /Image
  fdt /rk3588s-radxa-cm5-uconsole-aio.dtb
  append root=PARTUUID=__ROOT_PARTUUID__ rootfstype=ext4 rootwait rw console=tty1 consoleblank=0 loglevel=7 panic=10

label arch-72-base
  menu label Arch mainline 7.2.9 (base DTB, serial console)
  kernel /Image
  fdt /rk3588s-radxa-cm5-uconsole.dtb
  append root=PARTUUID=__ROOT_PARTUUID__ rootfstype=ext4 rootwait rw console=ttyS4,1500000 console=tty1 consoleblank=0 loglevel=7 panic=10
EOF
```

Substitute the new card's root **PARTUUID** (not the old Debian one — it's a fresh
partition):

```bash
NEW_ROOT_PARTUUID=$(sudo blkid -s PARTUUID -o value "${SD}2")
echo "$NEW_ROOT_PARTUUID"
sudo sed -i "s/__ROOT_PARTUUID__/$NEW_ROOT_PARTUUID/" /mnt/boot/extlinux/extlinux.conf
cat /mnt/boot/extlinux/extlinux.conf
```

> **Note on `console=ttyS4` in the base entry:** the *base* DTB keeps UART4 as a
> serial console; the AIO DTB disables it (it's the amplifier-enable line). This
> matches the Debian setup. If your preserved U-Boot expects a different
> `extlinux` path or syntax, check the RX image's original file for reference —
> it's in the backup tarball at `boot/extlinux/extlinux.conf`.

## Step 8 — display-kick service (REQUIRED)

Without this the first DSI enable at boot wedges → grey screen.

```bash
R=/home/kenny/git/uconsole-radxa-cm5-mainline/runtime
sudo install -m 644 "$R/display-kick.service" /mnt/etc/systemd/system/
# enable it by creating the symlink (can't systemctl on a chroot-less mount)
sudo mkdir -p /mnt/etc/systemd/system/multi-user.target.wants
sudo ln -s /etc/systemd/system/display-kick.service \
  /mnt/etc/systemd/system/multi-user.target.wants/display-kick.service
```

## Step 9 — Unmount and boot

```bash
sudo umount /mnt/boot
sudo umount /mnt
sync
```

Insert the SD into the uConsole (HDMI **unplugged**, so the console goes to the
panel upright — see the rotation note in
[rx-image-session-log.md §7](rx-image-session-log.md#7-the-rotated-console-gotcha-and-its-one-line-fix))
and cold-start. Login: **`alarm` / `alarm`** (root: `root` / `root`).

## Step 10 — Verify

```bash
uname -r                       # 7.2.9-dirty
dmesg | grep cwu50             # panel: TXW500170B0-BL (new)
dmesg | grep -i ocp8178        # OCP8178 one-wire backlight
ip -br link                    # eth0/end0 up
lsusb                          # keyboard/trackball hub
```

## Step 11 — Post-boot (on the device, via SSH)

Minimal, no WM (per request). Add only what's needed:

```bash
# basic tooling
sudo pacman -Syu
sudo pacman -S --needed git base networkmanager iwd  # iwd if you use WiFi

# GPU: panthor driver is built into the 7.2.9 kernel; Mesa for userspace
sudo pacman -S --needed mesa
# (no compositor yet — that's a later step when you want one; sway/labwc both
#  read rotation=90 from the DTB and rotate automatically)
```

**AIO v2** (when ready): the AIO DTB already enables GPS (UART2 → `/dev/ttyS2`)
and RTC (I2C7). The `aio` rail tool + gpsd + rtl-sdr are the same as guide step 8.

---

## If it doesn't boot

- **Black + no SSH:** the base-DTB entry (`arch-72-base`) carries a serial
  console (`console=ttyS4,1500000`) — a 1.5 Mbps adapter on PIN_23/19/25 shows
  the panic. Usual suspect: wrong `root=PARTUUID` in Step 7.
- **Boots but panel dark/grey:** warm-reboot rather than cold (known MIPI
  timing quirk), confirm `display-kick` is present (`systemctl cat display-kick`),
  check `dmesg | grep -iE 'panel|dsi|ocp'`.
- **U-Boot itself doesn't run:** the SPL write (Step 4) didn't take → re-run it,
  and confirm `count=32768` bs=512 landed at offset 0 on the right `$SD`.
- **Give up entirely:** restore the Debian state from the backup
  ([rx-image-session-log.md §8b](rx-image-session-log.md#8b-rollback-how-to-get-back-to-this-debian-state)) —
  re-flash the RX image or unpack the tarball onto a fresh partition layout.
