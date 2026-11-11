# mtgpu-chromium

Chromium **157.0.8103.38** (release channel) built from source on the M1000
laptop (aarch64, 12× Cortex-A78, KWin Wayland), with EDR/HDR verified against
KWin's real `wp_color_management_v1` headroom.

## Why this build exists

The pre-installed Google Chrome is the same upstream version, but on this
machine HDR playback was reported as not showing real EDR.  This project builds
the same source locally so the composition path can be instrumented and
verified, and ships it as a separate package that cannot disturb the system
browser.

## What was verified

With `CHROME_EDR_DEBUG=1` (a diagnostics patch in
`mtgpu-kde/tools/chromium-hdr`):

```
[EDR] swap headroom=3.83197 sdr_nits=83
[EDR] quad src={primaries:BT2020, transfer:PQ (white:203 nits)} is_video=1
      disp_headroom=3.83197  targeted_headroom=3.83197
```

Chromium 157 already consumes the compositor's headroom
(`targeted == display`), so the historical "Chromium uses a hard-coded
headroom" problem no longer exists in this version.  What this package adds is
the verified build, the diagnostics, and a launcher that always runs on
Wayland (where `wp_color_management_v1` exists).

## Install

```sh
sudo apt install ./mtgpu-chromium_157.0.8103.38_arm64.deb
mtgpu-chromium https://example.com
```

- Installs under `/opt/mtgpu-chromium`, launcher `/usr/bin/mtgpu-chromium`,
  desktop entry `Chromium (mtgpu, HDR)`.
- Does not modify or conflict with `google-chrome-stable` at runtime; the
  package declares `Conflicts:` only to prevent file collisions if the
  distribution's package is ever repackaged into `/opt`.

## Caveats

- **HEVC is not decoded** by this build (`ffmpeg_branding="Chrome"`); `hdr-bars.mp4`
  (HEVC) therefore does not play.  Use a VP9/AV1 PQ clip, e.g.
  `ffmpeg -i hdr-bars.mp4 -c:v libvpx-vp9 -pix_fmt yuv420p10le -color_trc smpte2084 ...`.
- Qt file dialogs are disabled (`use_qt6=false`): Chromium only takes Qt from its
  Debian sysroot, which has no Qt6.  File dialogs fall back to xdg-desktop-portal.
- Screenshots of HDR video come out black (hardware overlay planes are not
  captured); that is a capture limitation, not an EDR failure.

## Hardware scope

This build was validated on the **aibook OLED panel** (no backlight): the
compositor can pin the panel to full drive for free, keep the 500 nit peak
available and move only the SDR white point (200 nit at 100 %, 83 nit at 40 %),
so headroom is 2.5–6.0 and EDR always reaches the panel peak.

On a backlit LCD the equivalent setup has to **boost the backlight to maximum
while EDR/HDR is active**, which measurably increases power draw and battery
wear.  Such an implementation must save the user's backlight level on entering
EDR and restore it when EDR/HDR is switched off (and after a compositor
restart).  The Chromium patch here consumes `luminances.max`, i.e. the peak the
compositor reports as currently achievable, so it follows the boosted level on
an LCD and the pinned level on the OLED without extra logic.
