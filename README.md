# SmartPi-Kodi

Ready-to-flash **Kodi media center** image for the **Yumi SmartPi One** (Allwinner H3).
The board boots straight into Kodi — no desktop, no login prompt — the way LibreELEC
does on its supported hardware. LibreELEC has no SmartPi One target, so this image
rebuilds the same experience on top of Debian.

## Install

1. Download the image for your preferred base from the
   [latest release](https://github.com/Yumi-Lab/SmartPi-Kodi/releases/latest)
   (`.img.7z` multi-part archives — The Unarchiver on macOS and 7-Zip on Windows
   open `.001`/`.002` parts natively).
2. Extract the `.img` and verify it against the published `.img.sha256`.
3. Flash it to an SD card (8 GB minimum):

   ```bash
   diskutil list && diskutil unmountDisk /dev/diskN
   sudo dd if=<image>.img of=/dev/rdiskN bs=4m status=progress
   diskutil eject /dev/diskN
   ```

   [SmartPi-Imager](https://github.com/Yumi-Lab/SmartPi-imager) works too.
4. Insert the card, connect HDMI and power. The first boot expands the root
   filesystem, then Kodi starts on the HDMI output.

Default account: `pi` / `yumi`.

## Variants

| Release asset | Base OS | Notes |
|---|---|---|
| `*-armbian-smartpi1-trixie` | Armbian — Debian 13 « trixie » | recommended |
| `*-armbian-smartpi1-forky` | Armbian — Debian 14 « forky » (testing) | newer Kodi/Mesa, moving target |
| `*-dietpi-smartpi1-trixie` | DietPi on Debian 13 « trixie » | lightest footprint, DietPi tooling |
| `*-dietpi-smartpi1-forky` | DietPi on Debian 14 « forky » (testing) | newer Kodi/Mesa, moving target |

All variants ship the same Kodi setup; they differ only in the base image underneath.

## How it works

Two-layer build, same chain as every Yumi OS image:

```
Layer 1  Yumi-Lab/SmartPi-armbian (or DietPi-SmartPi)  →  bare OS image
Layer 2  this repo — CustomPiOS chroot                 →  Kodi image
```

The `kodi` module does all the work:

- installs Kodi from the Debian repositories (`kodi`,
  `kodi-peripheral-joystick`, `kodi-inputstream-adaptive`) plus a minimal X
  stack — Debian builds `kodi.bin` for armhf against X11 + GLES, so the image
  runs Kodi under `xinit` with the modesetting driver and glamor on the lima
  GPU; no desktop environment is installed
- renders `kodi.service`, which takes over tty1 at boot and runs Kodi as the
  `pi` user (`Conflicts=getty@tty1.service`, PAM login session)
- adds `pi` to the `video`, `render`, `input`, `audio` and `tty` groups and
  allows the X server to start from a systemd service (`Xwrapper.config`)

Module chains per variant (`config/<type>/default`):

```
armbian : base(udev_fix,armbian(armbian_net,kodi))
dietpi  : base(udev_fix,kodi)        # DietPi handles network + resize itself
```

## Status

- Every image is built in CI, `e2fsck`-checked and structure smoke-tested
  (Kodi binaries, service wiring, user groups) before it reaches a release.
- Not yet validated on hardware: HDMI-CEC, audio output selection, and video
  playback performance. The Debian Kodi build decodes video in software unless
  it can reach the cedrus V4L2 decoder; 1080p H.264 on an H3 is the practical
  ceiling either way.

## Development

- `develop` branch, push → `BuildImages.yml` builds all four variants as
  90-day artifacts.
- Release: run the `Release` workflow with a `X.Y.Z` version — it merges
  `develop` into `main`, builds the four variants and uploads them as
  split 7z assets with `.sha256` checksums.
- `ShellCheck.yml` lints every module and config on push.

Build engine: [CustomPiOS-Yumi](https://github.com/Yumi-Lab/CustomPiOS-Yumi) `1.5.0`.

## License

GPLv3 — same as the CustomPiOS build chain this repo derives from.
