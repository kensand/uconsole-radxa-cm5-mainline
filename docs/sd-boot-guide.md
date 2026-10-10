# From a dead uConsole to a working system on microSD — full guide

Start: a uConsole with a Radxa CM5, a **TXW500170B0-BL** panel, the HackerGadgets
CM5 adapter and **AIO v2** in the mPCIe slot, and a 10 Ah single-cell battery.
Nothing on any storage device works yet. End: Debian 13 + mainline kernel 7.2.9
(build in this repo), panel, keyboard, USB, Ethernet, AIO v2 (GPS, RTC, SDR,
rail switching), GPU acceleration, battery status.

This guide is written for **starting on microSD only** (no eMMC, no NVMe).
Everything else in this repo assumes a working system to keep as fallback;
here the fallback is *the same SD card*, and the order of operations exists
because of that.

> **Keep a working boot entry at all times.** Before every risky step you
> should have at least one kernel that boots. On an SD-only setup that means:
> Radxa's BSP kernel (from the image) stays in `extlinux.conf` forever, next
> to ours. If ours breaks you flip one line and reboot — no reflashing.

## What you need

| Item | Notes |
|---|---|
| uConsole + CM5 + HackerGadgets adapter | panel TXW500170B0-BL (new, Dec 2025+) or original — both work |
| AIO v2 in the mPCIe slot | optional at first boot, but assumed below |
| Battery installed | a sagging/empty battery masquerades as a broken device tree (see [battery.md](battery.md#power-supply-and-battery) — *grey panel and dead keyboard on every kernel you try = check the battery first*) |
| Host PC (Linux) with USB ports | to write the SD card and copy files |
| microSD, **≥ 32 GB**, class 10 or better | the image is ~4 GB written, the kernel build needs headroom later |
| A **good card reader** | a flaky reader mid-`dd` bricks the boot partition and you reflash anyway |
| 5 V / 3 A supply | the uConsole wants 12 V/2 A on its barrel jack; a 5 V USB supply is enough for flashing and bring-up, the eMMC/Radxa docs say 12 V for sustained loads |

Host packages (Debian/Ubuntu names): `git rsync bc bison flex libssl-dev
libelf-dev dwarves python3 aarch64-linux-gnu-gcc` (cross-compile; skip on an
ARM host) — only needed if you build the kernel yourself; step 3 of this guide
uses the build produced from this repo.

---

## Step 1 — Flash Radxa's Debian image to the microSD

Radxa ships a CM5 image based on Debian 12 (bookworm). It gives us: a boot
loader, an emergency kernel (their 6.1.x BSP — no display driver for our
panel, but it boots, gives SSH, and can repair our broken entry), and a Debian
system to upgrade to 13 later.

1. Get the image. From Radxa's `radxa-cm5-images` release assets (GitHub),
   take **`radxa-cm5-rpi-cm4-io_bookworm_cli_b3.output.img.xz`** — the
   flashable disk image (the `_kde_` one is the same plus a desktop you'd be
   upgrading anyway; `rootfs.tar.xz`/`seed.tar.xz` are build artifacts, not
   flashable). Verify the **SHA512** (not sha256) against the release's
   `.sha512sum` file before flashing:

   ```bash
   echo "<sha512 from .sha512sum>  radxa-cm5-rpi-cm4-io_bookworm_cli_b3.output.img.xz" \
     | sha512sum -c -
   ```

2. Write it. Identify the card (`lsblk` — do not overwrite the wrong disk):

   ```bash
   sudo xzcat radxa-cm5-rpi-cm4-io_bookworm_cli_b3.output.img.xz \
     | sudo dd of=/dev/sdX oflag=direct conv=fsync bs=4M status=progress
   sync
   ```

   (~690 MB compressed, ~4 GB written.) Wait for a *clean* finish. If the
   connection drops or errors appear, reseat the card or use a different
   reader and reflash.

3. **First boot from the SD, in the uConsole.** Power off (battery out, wait a
   minute, battery back in — a warm module or weak battery gives grey display
   and dead keyboard), plug in power, insert SD, power on.

   Expected: **no picture on the LCD** (the BSP kernel does not drive our
   panel — that is normal and the reason this repo exists), status LED
   heartbeats, and within a minute an IP on the network. The image's default
   credentials are on Radxa's page (commonly `user`/`1234` or on a card in
   the box — check the image's README).

4. Verify you are actually in:

   ```bash
   ssh user@<ip>
   uname -r          # 6.1.x-rk2410 (Radxa BSP) — that's expected, not ours yet
   ip -br link       # end0/end1 up with an address (RTL8211F works in the BSP)
   ```

If there is no IP after ~5 minutes: a CP2102N/FT232H/CH343P USB-serial to
UART4 at **1,500,000 baud** (PIN_23=TX→RX, PIN_19=RX→TX, PIN_25=GND; PL2303
and CH340 cannot do 1.5 Mbps — see README "Serial console") will show where
it died. The BSP entry's console is `ttyFIQ0` at 115200 — the FIQ console
has its own UART path and does not go to those pins — so on the BSP kernel
the serial adapter may show nothing useful; the LCD (dead) and the network
are your diagnostics, and a rescue SD (same image, second card) is the fix
path.

**Milestone 1: SSH works from the Radxa BSP kernel.** This is your emergency
system. Do not delete it, ever.

---

## Step 2 — Get the mainline kernel onto the card

Two ways; pick one.

### Option A — deploy the build from this repo (recommended)

Build on an x86_64 host with `aarch64-linux-gnu-gcc` (this repo's
`kernel/build-uconsole-kernel.sh` does the whole thing; it produced
**7.2.9-dirty** with the AIO DTBs in one clean run). The three artifacts are:

```
Image-7.2                                      # 42 MB kernel
rk3588s-radxa-cm5-uconsole-aio.dtb             # ← the one to boot (AIO v2)
modules-7.2.tar.gz                             # full module tree
```

(If you build from this repo's sources yourself, all three land in
`~/Work/uconsole-cm5-7.2.9/` — or wherever you staged them. The DTB name in
the deploy steps below is the *AIO* one; the base `...-uconsole.dtb` goes in
too, as a fallback if you ever remove the AIO.)

```bash
# on the host
scp Image-7.2 rk3588s-radxa-cm5-uconsole.dtb rk3588s-radxa-cm5-uconsole-aio.dtb \
    modules-7.2.tar.gz user@uconsole:/tmp/
```

### Option B — build on the device itself

Native build takes about an hour on the CM5 (16 cores) and 12 GB of
headroom; the image needs ~8 GB free. Install
`build-essential bc bison flex libssl-dev libelf-dev dwarves python3 rsync
git`, clone the kernel, follow README steps 2–3, then `STAGING=/root/staging
KDIR=$PWD ~/build-uconsole-kernel.sh`. You then have the files in-tree
(`arch/arm64/boot/Image`, `arch/arm64/boot/dts/rockchip/*.dtb`) instead of in
`/tmp`.

---

## Step 3 — Deploy (three artifacts, together, or it fails silently)

**A stale or mismatched DTB boots the kernel with the old device tree** —
the panel silently does the old thing. Copy Image, DTB *and* modules of the
same build, together.

```bash
# on the uConsole (as user, via SSH)
KREL=$(zcat /tmp/none 2>/dev/null; echo 7.2.9-dirty)   # from the build: cat include/config/kernel.release
sudo cp /tmp/Image-7.2 /boot/
sudo mkdir -p /boot/dtb-7.2
sudo cp /tmp/rk3588s-radxa-cm5-uconsole*.dtb /boot/dtb-7.2/

# DEBIAN TRAP (docs/debian.md): never `tar -C /` a modules tarball whose
# archive has lib/ as a directory — it replaces the /lib -> usr/lib symlink
# and the system stops booting. Use rsync/cp:
sudo rsync -a --no-o --no-g <staging>/lib/modules/7.2.9-dirty /lib/modules/
# for the tarball, extract to /tmp first, then:
#   sudo rsync -a --no-o --no-g /tmp/lib/modules/7.2.9-dirty /lib/modules/
```

Note the release name carries `-dirty` (patched tree, expected — it is what
`uname -r` will print). `/boot` on the SD may be small (512 MB) — Image is 42
MB, no problem.

**Which DTB is which, one subtle point:** the *base* DTB (`...-uconsole.dtb`)
keeps the UART4 serial console declared in its `chosen` node and a plain
on/off backlight. The *AIO* DTB (`...-uconsole-aio.dtb`) — which includes the
mainboard include — **disables UART4** (the pin becomes the amplifier-enable
line, held low, to kill the AW8110 hiss on battery), switches the backlight
to the OCP8178 32-level driver, and adds the AIO's GPS/RTC/fan nodes. You
boot the AIO DTB; the base one is the fallback if the AIO is ever removed.
Consequence for recovery: with the AIO DTB there is **no serial console at
all** — SSH is the recovery path, and the serial-console advice in the
recovery section applies to the Radxa BSP entry, not ours.

Get the root PARTUUID (the SD is `mmcblk0` while it is the boot device; if
you later move to eMMC the number changes):

```bash
lsblk -o NAME,PARTUUID
# the ext4 root partition's PARTUUID is what goes into the boot entry
```

---

## Step 4 — Add the boot entries

Edit `/boot/extlinux/extlinux.conf` (Debian: `sudo nano ...`; it is the file
U-Boot reads directly). **Keep Radxa's existing entry intact.** Add two labels
— plain DTB (fallback, keeps the serial console) and AIO DTB (the one you
boot):

```
label uconsole-72-mainline
    menu label uConsole mainline 7.2.9 (base DTB, serial console, amp hiss)
    linux /boot/Image-7.2
    fdt /boot/dtb-7.2/rk3588s-radxa-cm5-uconsole.dtb
    append root=PARTUUID=<YOUR-PARTUUID> rootfstype=ext4 rootwait rw console=ttyS4,1500000 console=tty1 consoleblank=0 loglevel=7 panic=10

label uconsole-72-aio
    menu label uConsole mainline 7.2.9 (AIO v2, no serial console)
    linux /boot/Image-7.2
    fdt /boot/dtb-7.2/rk3588s-radxa-cm5-uconsole-aio.dtb
    append root=PARTUUID=<YOUR-PARTUUID> rootfstype=ext4 rootwait rw console=tty1 consoleblank=0 loglevel=7 panic=10
```

Rules that have cost evenings (all from this repo's write-ups):

- **`root=PARTUUID=`, never `root=UUID=`** — no initramfs in this kernel, and
  a bare kernel cannot resolve filesystem UUIDs.
- **No `earlycon=`.** It touches UART4 before its clock is ungated and
  hard-hangs the boot with *no output at all*.
- **`fdt`, not `fdtdir`** — Radxa's entries use `fdtdir /usr/lib/linux-image-...`
  (Debian layout); a standalone DTB file is `fdt /boot/dtb-7.2/...`.
- **No `console=ttyS4,1500000` in the AIO entry** — the AIO DTS disables
  UART4 outright and hogs the pin low as PA_EN (the mainboard's amplifier
  enable — with a console on it the AW8110 hisses on battery), so the AIO DTB
  has no serial console at all; SSH is the recovery path. The **base entry
  keeps it**: the base DTS still declares `serial4:1500000n8`, so the base
  DTB *is* your serial-console fallback (1.5 Mbps adapter, see the recovery
  section) — at the cost of amplifier hiss on battery.
- Leave `default` on Radxa's entry for the *first* mainline boot: at the U-Boot
  menu, pick our label manually. Once ours has booted twice cleanly, set
  `default uconsole-72-aio`.

### The apt trap (do this before any `apt` run)

`u-boot-menu` (and Radxa's `rsetup` trigger) regenerates this file and **drops
any entry it did not write** — ours. Install the guard *before* upgrading:

```bash
sudo cp /boot/extlinux/extlinux.conf /root/extlinux.conf.mainline
sudo cp runtime/debian/99-restore-extlinux /etc/apt/apt.conf.d/
sudo apt-mark hold u-boot-menu u-boot-radxa-cm5-rpi-cm4-io
```

(Refresh `/root/extlinux.conf.mainline` after every *intentional* change to
the menu. Details in [debian.md](debian.md#boot-entries-and-the-apt-hook).)

---

## Step 5 — First boot of the mainline kernel (and the grey-screen fix)

Install the runtime files **before** the first boot (SSH is still up):

```bash
# REQUIRED — the first DSI enable at boot wedges the controller (mainline
# dw-mipi-dsi2 bug). This one-shot service cycles the panel once at boot.
# Without it: backlight on, screen uniform grey.
sudo cp runtime/display-kick.service /etc/systemd/system/
sudo systemctl enable display-kick.service

# full power-off (cuts the AXP rails, not just the CM5) + watchdog bounds
sudo cp runtime/axp-off.sh /usr/lib/systemd/system-shutdown/
sudo chmod +x /usr/lib/systemd/system-shutdown/axp-off.sh
sudo mkdir -p /etc/systemd/system.conf.d
sudo cp runtime/reboot-watchdog.conf /etc/systemd/system.conf.d/
```

Now the real moment — a **true cold start**: power off, battery out, **wait a
full minute**, battery back in, power on. (A warm module or a near-empty
battery gives grey display + dead keyboard and nothing else changes.)

**Selecting the entry.** On many of these images the U-Boot menu prints to the
**FIQ/UART serial console, not HDMI** — on HDMI the boot goes straight black →
the `default` entry, with no menu to pick from (mashing keys does nothing). If
you have no serial adapter, the reliable way to boot a new entry is to make it
the default:

```bash
sed -i 's/^default l0/default uconsole-72-aio/' /boot/extlinux/extlinux.conf
```

then cold-start. (A real menu appears only on the serial console. See the
[RX-image session log](rx-image-session-log.md#5-the-uboot-menu-is-not-on-hdmi-set-default-instead)
for the full story.) If you *do* have the menu, select **`uConsole mainline
7.2.9 (AIO v2)`**.

Expected: boot log on the LCD (console=tty1, rotated correctly by fbcon),
login prompt `uconsole login:`, then:

```bash
uname -r                       # 7.2.9-dirty
dmesg | grep cwu50             # panel: TXW500170B0-BL (new)   ← panel detected
dmesg | grep -i ocp8178        # OCP8178 one-wire backlight, 31 levels
ls /sys/class/backlight/       # backlight, max_brightness = 31
ip -br link                    # end0 up, has an IP
lsusb                          # keyboard/trackball hub present
glxinfo | grep -i renderer     # after step 7: Mali-G610 (Panfrost), NOT llvmpipe
```

If the console text is **rotated 90°** but you're watching over **HDMI**: that's
fbcon bound to the HDMI framebuffer (no panel rotation), not a real problem —
**unplug the HDMI and reboot** and the on-panel console is upright. The DSI
panel is correct; under sway/labwc rotation is automatic anyway. See the
[session log](rx-image-session-log.md#7-the-rotated-console-gotcha-and-its-one-line-fix).

If the screen is grey: the kick service didn't run, or it's the warm-start
trap — re-run a true cold start. If nothing at all and no IP: boot the
**base-DTB entry** and read the panic on a 1.5 Mbps serial adapter (it
keeps the UART4 console; see the recovery section below). A dead
`root=PARTUUID` is the usual suspect; fix it from there or from the BSP
entry/rescue SD.

**Milestone 2: mainline kernel on the panel.** If you are reading this on the
uConsole's own screen, you are done with the hard part.

---

## Step 6 — Upgrade Debian 12 → 13 (Trixie)

Why: the GPU (panthor + panfrost) needs **Mesa 25**, the `aio` tool needs
**libgpiod v2** — both arrive with Trixie. (Bookworm boots the kernel fine;
the desktop just stays on software rendering.)

```bash
# 1. sources: bookworm -> trixie, remove bookworm-backports and any
#    third-party source (e.g. a Meshtastic repo) first
sudo sed -i 's/bookworm/trixie/g; s/bookworm-backports/trixie-backports/g' \
    /etc/apt/sources.list /etc/apt/sources.list.d/* 2>/dev/null
sudo apt update
sudo apt upgrade --without-new-pkgs
# 2. answer N to config prompts for zramswap, plymouthd.conf, chrony.conf
sudo apt full-upgrade
# 3. the apt hook from step 4 restores extlinux.conf if it was clobbered
cat /boot/extlinux/extlinux.conf        # our labels still there?
```

After the upgrade: reboot (cold start), verify `uname -r` is still ours and
the panel is still alive. `needrestart` will nag about a "newer" kernel —
silence it:

```bash
echo '$nrconf{kernelhints} = 0;' | sudo tee /etc/needrestart/conf.d/no-kernel-hint.conf
```

If you keep the Radxa desktop packages, the BSP leftovers in
[debian.md](debian.md#radxa-leftovers) will bite (GDM dies on a diverted
`gdm.schemas`); for the CLI image you skipped that step and owe none of it.

---

## Step 7 — Desktop and GPU

Pick one (both need Mesa 25 from Trixie and the panthor driver, which is
built into this kernel):

**sway** (no display manager):

```bash
sudo apt install -y sway foot wofi
mkdir -p ~/.config/sway
cat > ~/.config/sway/config << 'EOF'
set $mod Mod1
output DSI-1 transform 90
input type:keyboard xkb_layout us
exec swayidle -w timeout 120 'swaymsg "output DSI-1 power off"' resume 'swaymsg "output DSI-1 power on"'
EOF
# start from .bash_profile on tty1 (no display manager)
```

**labwc** instead: `sudo apt install -y labwc`, and `cp runtime/labwc-autostart
~/.config/labwc/autostart` (it carries the rotation sway gets from the DTB
for free).

**GNOME** (`gnome-core` + GDM) works too — see [debian.md](debian.md#desktop);
clear Radxa leftovers first if present, and **disable suspend** (suspend-to-RAM
does not resume on this board; a masked `sleep.target` et al. is the
permanent fix — the GNOME section has the exact commands).

Verify hardware acceleration:

```bash
sudo apt install -y mesa-utils
glxinfo | grep -i renderer      # want "Mali-G610 ... (Panfrost)", not llvmpipe
```

---

## Step 8 — AIO v2: rails, GPS, RTC, SDR

(Only if the AIO v2 is in the slot — the AIO DTB already enabled its UART2
GPS and I2C7 RTC at kernel level.)

**Rails** (the `aio` tool drives the four enable lines):

```bash
sudo apt install -y python3-libgpiod
sudo install -m 755 runtime/aio-v2/aio /usr/local/bin/aio
aio status                        # usb on, sdr on at reset (pull-up!) — normal
sudo cp runtime/aio-v2/aio-rails.service /etc/systemd/system/
sudo systemctl enable aio-rails.service    # sets usb on, sdr off after boot
```

The service guards itself: it only runs when the AIO's RTC answers on I2C7,
so it is inert if you swap in a different slot board later.

**RTC** (should already be present from the AIO DTB):

```bash
ls -l /dev/rtc0                   # rtc-pcf85063 registered
timedatectl | grep -i synchronized
sudo hwclock -w && sudo hwclock -r
```

Fit a **CR1220 coin cell** on the AIO or it only keeps time while powered.
If Radxa's `fake-hwclock` package is present, purge/mask it.

**GPS** (receiver on `/dev/ttyS2`, rail-switched by gpsd):

```bash
sudo apt install -y gpsd gpsd-clients
echo 'DEVICES="/dev/ttyS2"' | sudo tee /etc/default/gpsd
sudo mkdir -p /etc/systemd/system/gpsd.service.d
sudo cp runtime/aio-v2/gpsd-rail.conf /etc/systemd/system/gpsd.service.d/rail.conf
sudo systemctl daemon-reload && sudo systemctl enable --now gpsd
sleep 90; cgps                    # a fix takes 1-2 minutes outdoors / window
```

**RTL-SDR** (USB, behind the AIO hub):

```bash
sudo apt install -y rtl-sdr
echo 'blacklist dvb_usb_rtl28xxu' | sudo tee /etc/modprobe.d/rtl-sdr-blacklist.conf
sudo aio sdr on
rtl_test -s 2048000               # clean, no dropped samples
```

**LoRa** — as delivered there is **no MISO path** on the CM5; it needs the
one-wire fix (wire on the AIO from the SX1262's MISO pad to test pad TP2)
plus the `aio-lora` DTB. That is a soldering job and a separate boot entry;
see [aio-v2.md](aio-v2.md#the-one-wire-fix) when you get there.

---

## Step 9 — Battery, WiFi, and finishing

**Battery status** (the AXP228 fuel gauge; the AXP patch in this repo makes
it report *Full* once charge terminates instead of *Charging* forever):

```bash
upower -d | less                  # capacity, state, and (after the patch) "Full"
sudo i2cget -f -y 9 0x34 0x36     # AXP228 register 0x36: PWRON hold = hard poweroff
```

Install this fork's battery units if you want the automatic behaviour (AXP
tuning for a 10 Ah cell, clean poweroff at 3.50 V, minute-by-minute log):
follow the README "Quick install of the runtime pieces" — edit the capacity
bytes in `axp-battery-config.service` for your cell first.

**WiFi** — no onboard radio on the CM5. An RTL8812AU dongle (firmware
`firmware-realtek`) works with the in-tree `rtw88_8812au` this build enables;
RTW88 8821CU/8822BU/CU/8723DU are in as modules too.

**Shutdown**: the power-button hold is a hard AXP rail cut (works even when
frozen, and is the way out of a hang — but it is not a clean shutdown).
Normal poweroff goes through `runtime/axp-off.sh` (step 5) and the reboot
hang is bounded by `reboot-watchdog.conf`.

---

## Recovery — how to get back when something breaks

Order of operations, cheapest first:

1. **Wrong entry / broken entry in `extlinux.conf`** → boot the *other*
   working entry (BSP or base-DTB), fix the file over SSH. The apt hook
   (step 4) also restores it from `/root/extlinux.conf.mainline`.
2. **`extlinux.conf` gone or U-Boot bricked** → boot from the **base-DTB
   entry** if any entry still works; otherwise a rescue SD (step 1 image,
   written to a second card) mounts our root partition (`/dev/mmcblk0p3` —
   check with `blkid`) and rewrites the file.
3. **Kernel won't boot** → boot the **base-DTB entry**: it carries
   `console=ttyS4,1500000`, so a 1.5 Mbps adapter (PIN_23=TX→RX,
   PIN_19=RX→TX, PIN_25=GND; CP2102N/FT232H/CH343P only) shows where it
   panics. The usual suspects: `root=PARTUUID` (wrong value) or a module
   mismatch (modules dir for a different kernel release than the Image —
   the build prints which release the modules are for). The Radxa BSP entry
   is the next fallback if even the base entry won't come up.
4. **Completely dark, no IP, LED dead or static** → power/battery first
   ([battery.md](battery.md#power-supply-and-battery)), then a full reflash
   (step 1). Nothing here survives a reflash; the SD *is* the system, and a
   tarball of `/boot /lib/modules /etc /root` on a USB stick before risky
   steps makes step 4 a non-event.
5. **Keyboard gone after a reboot, `lsusb` shows `1eaf:0003 Maple DFU`** →
   the keyboard MCU is in its bootloader; a cold start brings it back
   ([debian.md](debian.md#keyboard-controller-in-dfu-mode)).

## What this setup does not have (yet)

- **Audio** — no DAC on the CM5; the AIO DTB silences the amplifier hiss but
  there is no output. (PWM audio is hardware-possible, no mainline driver.)
- **LoRa over the air** — needs the TP2 wire; the chip initialises without
  it only as far as the bus allows.
- **Suspend-to-RAM** — does not resume; keep the sleep targets masked.
- **eMMC/NVMe** — the DTBs carry PCIe and eMMC nodes and they are harmless
  with nothing attached; when you add the NVMe board it works without
  changes (Gen2 x1, `CONFIG_BLK_DEV_NVME=y` in-tree).

---

*Verified 2026-10-10: kernel 7.2.9 built from this repo on a clean
`linux-7.2.y` clone, all three DTBs and the module tree produced in one
run, zero build errors. Hardware: TXW500170B0-BL panel, AIO v2.*
