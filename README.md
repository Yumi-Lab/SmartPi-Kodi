# SmartPi-Kodi

Ready-to-flash **Kodi media center** image for the **Yumi SmartPi One** (Allwinner H3).
The board boots straight into Kodi — no desktop, no login prompt — the way LibreELEC
does on its supported hardware. LibreELEC has no SmartPi One target, so this image
rebuilds the same experience on top of Debian.

## Install

1. Download the `.img.xz` for your preferred base from the
   [latest release](https://github.com/Yumi-Lab/SmartPi-Kodi/releases/latest),
   and check it against the published `.img.xz.sha256` if you want to be sure.
2. Flash it to an SD card (8 GB minimum) —
   [SmartPi-Imager](https://github.com/Yumi-Lab/SmartPi-imager) and USBImager
   flash the `.img.xz` directly, no extraction needed. Or by hand:

   ```bash
   diskutil list && diskutil unmountDisk /dev/diskN
   xz -dc <image>.img.xz | sudo dd of=/dev/rdiskN bs=4m status=progress
   diskutil eject /dev/diskN
   ```

3. Insert the card, connect HDMI and power. The first boot expands the root
   filesystem, then Kodi starts on the HDMI output.

Should an image ever outgrow GitHub's 2 GiB asset limit, the release falls
back to a split 7z (`.7z.001`/`.7z.002`) that The Unarchiver (macOS) and
7-Zip (Windows) reassemble natively.

Default account: `pi` / `yumi`.

## Variants

| Release asset | Base OS | Notes |
|---|---|---|
| `*-armbian-smartpi1-trixie-debian13` | Armbian — Debian 13 « trixie » | recommended |
| `*-armbian-smartpi1-forky-debian14` | Armbian — Debian 14 « forky » (testing) | newer Kodi/Mesa, moving target |
| `*-dietpi-smartpi1-trixie-debian13` | DietPi on Debian 13 « trixie » | lightest footprint, DietPi tooling |
| `*-dietpi-smartpi1-forky-debian14` | DietPi on Debian 14 « forky » (testing) | newer Kodi/Mesa, moving target |

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
  `.img.xz` assets with `.sha256` checksums (split 7z fallback past 2 GiB).
- `ShellCheck.yml` lints every module and config on push.

Build engine: [CustomPiOS-Yumi](https://github.com/Yumi-Lab/CustomPiOS-Yumi) `1.5.0`.

## License

GPLv3 — same as the CustomPiOS build chain this repo derives from.
