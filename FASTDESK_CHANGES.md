# Fastdesk changes

Base: RustDesk `master` at commit `eefd22b2057ba057305b10a6b5bf93c79b686eb9`.

Local branch: `fastdesk/low-latency-60fps`.

Upstream remote: `https://github.com/rustdesk/rustdesk.git` (`upstream`).

## Goals

Fastdesk is a Windows-first RustDesk fork focused on a stable low-latency 60 FPS target and selectable host audio quality.

No RustDesk protocol fields, server identifiers, package IDs, or rendezvous behavior were changed. A Fastdesk build remains wire-compatible with ordinary RustDesk peers, although the complete fixed-FPS behavior is obtained when Fastdesk runs on the host and viewer.

## Fixed 60 FPS mode

Config key: `fastdesk-fixed-60-fps`.

The option is enabled by default. Only the explicit value `N` disables it. The desktop setting is shown as **Fastdesk fixed 60 FPS**.

Changes:

- The viewer no longer calculates an adaptive cap from decode speed and queue length (`decode_fps * 0.8` for relayed links or `decode_fps * 0.9` for direct links).
- The viewer sends a constant `custom_fps = 60` target instead of stepping the host down to values such as 24–28 FPS.
- The host-side video QoS ignores adaptive FPS caps and network-delay FPS reduction while Fastdesk mode is enabled.
- Existing adaptive bitrate/quality logic remains active. Congestion is therefore handled by reducing video quality/bitrate instead of deliberately lowering the requested FPS.
- Changing image quality no longer resets the session to 30 FPS.
- The compressed-frame decode queue is reduced from 120 frames to 8 frames in Fastdesk mode. At 60 FPS this bounds that queue to roughly 133 ms. RustDesk's existing `force_push` overflow path discards stale queued data and requests a refresh/keyframe rather than allowing a multi-second backlog.

This is a hard software target, not a physical guarantee. Actual displayed FPS can still be below 60 when capture, encode, network, decode, GPU presentation, or the remote machine cannot sustain it. RustDesk may also avoid transmitting identical frames when the screen is literally unchanged; Fastdesk removes adaptive FPS throttling but does not intentionally waste bandwidth on endless duplicate frames.

## Selectable audio bitrate

Config key: `audio-bitrate-kbps`.

Desktop host settings provide these values:

- 64 kbps
- 96 kbps
- 128 kbps (default)
- 192 kbps
- 256 kbps

The value is validated in the Rust audio service. Invalid or missing values fall back to 128 kbps.

RustDesk already uses Opus `LowDelay` with 10 ms audio blocks. Fastdesk now calls `Encoder::set_bitrate(Bitrate::Bits(...))` immediately after every Opus encoder is created, on both CPAL/Windows and PulseAudio/Android paths. Changing the option restarts only the audio service so the new bitrate is applied without restarting the whole application.

The selected Opus bitrate is a target bitrate; Opus retains its normal variable-bitrate behavior.

## Changed files

- `src/client/io_loop.rs` — disables viewer adaptive-FPS feedback and uses the short Fastdesk decode queue.
- `src/server/video_qos.rs` — fixes host capture/encode scheduling target at 60 while retaining bitrate adaptation.
- `src/ui_session_interface.rs` — prevents quality changes from resetting FPS to 30.
- `src/server/audio_service.rs` — validates and applies the Opus bitrate.
- `src/flutter_ffi.rs` — restarts the audio service after bitrate changes.
- `flutter/lib/consts.dart` — Fastdesk config keys.
- `flutter/lib/desktop/pages/desktop_setting_page.dart` — fixed-60 toggle and audio bitrate selector.

## Windows build

Prerequisites:

1. Visual Studio 2022 with Desktop development with C++ and the Windows SDK.
2. Rust stable with Cargo and `rustfmt`.
3. Flutter with Windows desktop support enabled.
4. Python 3.
5. vcpkg, with `VCPKG_ROOT` set.

From PowerShell:

```powershell
cd E:\Dev\Fastdesk
git submodule update --init --recursive

$env:VCPKG_ROOT = 'E:\Dev\vcpkg' # adjust to the real path
& "$env:VCPKG_ROOT\vcpkg.exe" install `
  libvpx:x64-windows-static `
  libyuv:x64-windows-static `
  opus:x64-windows-static `
  aom:x64-windows-static

python build.py --flutter --skip-portable-pack
```

For the hardware-codec/VRAM build normally desired for Fastdesk:

```powershell
python build.py --flutter --hwcodec --vram --skip-portable-pack
```

The unpacked Windows Flutter build is produced under:

```text
E:\Dev\Fastdesk\flutter\build\windows\x64\runner\Release\
```

To build the installer/portable package, omit `--skip-portable-pack` after installing the Python requirements used by `libs/portable`.

## Verification performed in this environment

- Repository cloned recursively and moved to branch `fastdesk/low-latency-60fps`.
- `origin` renamed to `upstream`.
- Pinned `magnum-opus` commit `588c6e1f9ed50c3a01fa64f3bd3e7cdb0378a114` verified to expose `Bitrate::Bits` and `Encoder::set_bitrate`.
- `git diff --check` passed.
- `python build.py --help` passed.

Rust, Dart, and Flutter toolchains were not installed or available in `PATH` on this machine, so a full `cargo check`, `dart format`, and Windows build were not run here.

## Known limitations / next validation

- The audio bitrate selector is implemented in the desktop Flutter settings UI; mobile UI was intentionally left unchanged for the Windows-first fork.
- The 8-frame compressed queue is intentionally aggressive. Test VP9/AV1/H.264/H.265 hardware decoding on a deliberately overloaded client and tune to 12 frames if refresh/keyframe requests become too frequent.
- Measure capture FPS, encoded FPS, decode queue depth, end-to-end input latency, and bitrate separately. The quality monitor's FPS alone does not prove end-to-end latency.
- Branding is deliberately minimal. Internal RustDesk protocol names, executable/package identifiers, services, and server compatibility were not mass-renamed.
