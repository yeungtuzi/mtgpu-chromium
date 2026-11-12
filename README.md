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

## Two features of this build

1. **Native EDR support for the aibook panel (500 nit peak).**  The build reads
   the real headroom the compositor advertises and, with
   `patches/chromium-157-edr-panel-peak.patch` (in `mtgpu-kde`), takes the panel
   peak from `luminances.max` instead of the brightness-scaled
   `target_luminance`.  Measured: `targeted_headroom == display_headroom`, with
   the 500 nit peak reachable.  *HDR playback is still not usable end to end on
   this machine* — that needs the vendor to finish the GPU driver and panel
   parameter work (HDR output path), not just the client side.
2. **Hardware-accelerated HEVC decoding for mtgpu.**  Chromium's bundled ffmpeg
   ships with `CONFIG_HEVC_DEMUXER/PARSER/DECODER 0` on Linux, so neither
   Chromium nor Google Chrome can open HEVC at all on this platform (verified:
   `DEMUXER_ERROR_NO_SUPPORTED_STREAMS`, Chrome 157 included).  Our HEVC variant
   rebuilds that ffmpeg with the HEVC demuxer/parser enabled, so HEVC reaches the
   Mthreads VPU VA-API decoder (measured 11x realtime on 1080p10 PQ vs 2.9x in
   software).

### Highlight roll-off (BT.2390)

The compositor advertises a **fixed 1000 nit HDR target** to clients (independent
of the brightness slider) and renders into the real 500 nit panel peak with an
ITU-R BT.2390 EETF (129-entry PQ lookup table).  500 nit content maps to ~338 nit
and 1000 nit content to 500 nit, so the top two bars stay distinguishable instead
of both clipping.

Verified in the self-built Chromium with `hdr-bars-vp9.webm`: Chromium reports
headroom 1.00 at 100 % and 2.30 at 40 % (both = the same 1000 nit target), and
KWin's video pipeline logs `tonemap=1 maxWhite=1000 maxOut=500` in both cases;
the four bars are distinct around 50 % brightness.

Known limitation: the HDR clip still follows the brightness slider somewhat --
at 100 % the SDR white reaches the panel peak, so a 500 nit highlight coincides
with white and the last two bars merge; near 0 % the highlights dim too.  The
EETF itself is fixed; the remaining dependence comes from the SDR composition
blend.  This is close to macOS EDR behaviour and is accepted for now.

**Recommended brightness for HDR playback: 40-50 % (~200-250 nit SDR white).**
Below that the highlights look dim; above it the 500 nit bar climbs into the
1000 nit bar and they merge near 100 %.  This is the known brightness dependence
described above, not intended behaviour.

### HEVC hardware decode: verified

The bundled ffmpeg is rebuilt with `CONFIG_HEVC_DEMUXER=1` plus
`libavformat/hevc.c` / `hevcdec.c`; the parser used for the VA-API path is
Chromium's own `media::H265Parser`, so `CONFIG_HEVC_PARSER`/`CONFIG_HEVC_DECODER`
stay off.

Verified on the M1000 (`mthreads-gmi -q`, `Utilization -> Decoder`):

| flags | decoder util | playback |
|---|---|---|
| default | 0.0 % | no decoder available (build has no software HEVC) |
| `--enable-features=VaapiVideoDecoder,VaapiIgnoreDriverChecks` | **3.2-6.3 %** | HEVC plays, VPU active |

So on Linux the VA-API decoder is gated behind those two features; the
`mtgpu-chromium` launcher/desktop entry must pass them for HEVC to work.
