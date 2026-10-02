# CarrotTV on the Ugoos AM6B+ (SD card / USB stick)

Non-destructive: Android on the internal eMMC is not touched. Remove the card → the box boots Android again.

> Status: the image, the arm64 system and the UI were verified in an emulator. The Amlogic
> boot chain, HDMI output, GPU (Panfrost) and the infrared remote can only be verified on a
> real box — please report what you see at each step below.

## 1. Build and flash
```
./build.sh am6                                   # → CarrotTV-AM6B.img
lsblk                                            # find the card, e.g. /dev/sdX (NOT your system disk)
sudo dd if=CarrotTV-AM6B.img of=/dev/sdX bs=4M conv=fsync status=progress
```
Use a card/stick of 8 GB or more (16 GB+ recommended: Android app data lives on it). balenaEtcher works too.

## 2a. Box runs CoreELEC from internal storage (no Android)
Nothing to set up. CoreELEC's bootloader checks the USB stick for a `cfgload` file on every boot; the stick has one
that hands over to CarrotTV. **Do not use the toothpick method** on a CoreELEC box with this stick: it would run
`aml_autoscript` and replace CoreELEC's boot settings.

1. Shut CoreELEC down (Power menu → Power off), unplug power.
2. Plug the stick into the **USB 2.0** port (black). The bootloader only checks one port ("usb 0"); if CoreELEC starts instead, move the stick to the other port.
3. Plug power back in. CarrotTV boots. Without the stick, CoreELEC boots as always.

## 2b. Box runs Android: teach it once to boot from SD/USB
The factory bootloader only looks at internal storage until it is told otherwise. Any one of:

- **Toothpick method**: power off, insert the card, press and hold the reset button inside the
  AV jack with a toothpick, plug in power, keep holding ~5 s until the Ugoos logo appears, release.
- **From Android** (rooted Ugoos firmware has a terminal / ADB): with the card inserted run `reboot update`.

This runs `aml_autoscript` from the card once. It only changes the bootloader *environment* so that
SD/USB is tried before eMMC; nothing is flashed. (If the box previously ran CoreELEC this step is already done.)

## 3. First boot
- CarrotTV logo → home screen in roughly 40–60 s from a decent card (first boot is the slowest).
- The data partition grows to fill the card in the background.
- Android is prepared in the background (~1 min); "Android Apps" works after that.
- Wired Ethernet works out of the box. Wi-Fi/Bluetooth on the AM6B+ (AP6398S) may need a board-specific
  NVRAM file that Debian does not ship; tell me if `nmcli dev wifi` shows nothing and I'll add it.

## 4. Remote
The stock infrared remote is mapped in `/etc/rc_keymaps/ugoos_am6.toml`:

| Button | Action |
|---|---|
| Arrows / OK | navigate / select |
| Back | back (Esc; also "back" inside Android apps) |
| Home | close the running app, return to CarrotTV |
| Menu | CarrotTV settings |
| Vol − / Vol + / Mute | volume keys |
| Power | shut down |

If buttons do nothing: plug in a USB keyboard, press **Ctrl+Alt+F2**, log in as `carrot` (no password), run
`sudo carrot-remote-learn`, press each button and send me the `scancode = 0x…` lines.

## 5. If it does not boot
| What you see | Likely cause |
|---|---|
| Android boots as usual | step 2 not done, or card not readable in that slot — try the other USB port / SD slot |
| Ugoos logo forever / black screen | chainloaded `u-boot.ext` or device tree not liked by this board revision — tell me; a USB-TTL serial log (115200 8N1) would pinpoint it |
| CarrotTV logo, then text or black | kernel is up; GPU/HDMI issue — hold on, press Ctrl+Alt+F2 with a keyboard and check `journalctl -b -u carrot-session` |

## Layout of the card
- p1 FAT32 `CARROTBOOT`: `aml_autoscript`, `s905_autoscript`, `boot.scr`, `uEnv.txt` (kernel command line — editable),
  `u-boot.ext`, `zImage`, `uInitrd`, `dtb/amlogic/meson-g12b-ugoos-am6.dtb`, `live/filesystem.squashfs`
- p2 ext4 `persistence`: everything you change (settings, Wi-Fi passwords, Android apps and data)

Boot files come from the Armbian-for-Amlogic project (ophub), which lists the *UGOOS AM6 Plus* with exactly this
device tree and u-boot (`u-boot-gtkingpro.bin`, pinned by SHA-256 in `boot/am6/u-boot.ext.sha256`).
