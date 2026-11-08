# Changelog

## v1.2.0 - 2026-11-08

- `MotionTracker` (Python `sky_tracker.MotionTracker`) finds and follows moving
  objects with no bbox to start from, each with a stable ID: drones and birds
  against the sky, or meteors. Modes `background`, `diff` and `bright-diff`,
  plus `moving_only`, `region` and `box_padding`. See the
  [Python quickstart](docs/quickstart-python.md) and the
  [1.2.0 release notes](docs/release-1.2.0.md).
- `bright-diff` mode is free: it needs no licence key, for any use. Everything
  else still needs one.
- `correlation`: the update takes a third to three quarters less time and holds
  hundreds of megabytes less memory at 1080p. The tracker re-locks a target
  that left its window, never accepts a weak peak (PSR below 6), and fits its
  box tighter. CLI `--correlation-reacquire N` (off by default) finds the
  target again after it left the frame.
- Python 3.14 wheels.

## v1.1.1 - 2026-11-06

- Free 7-day trial keys for the full SDK, from
  [sky-tracker.deneb-systems.com/trial](https://sky-tracker.deneb-systems.com/trial/).
  A trial key (`SKT2.`) is confirmed with the licence server over HTTPS when a
  tracker starts, so it needs an internet connection; earlier versions refuse
  it. Bought and student keys stay fully offline. See the
  [1.1.1 release notes](docs/release-1.1.1.md).
- CLI: meteor tracking on a static camera with `--mode bright-diff`,
  `--moving-only`, `--box-padding` and `--target-end-when-still`.

## v1.1.0 - 2026-11-04

- `correlation` is now the improved correlation-filter tracker alone (1.0.0's
  `pure`), with no neural network. `ml` (CLI `--ml`, Python `ml=True`, C++
  `TrackerOptions::ml`, Node `ml: true`) adds NanoTrack and reproduces 1.0.0's
  `correlation`. See the [1.1.0 release notes](docs/release-1.1.0.md).
- `ml_every` (CLI `--ml-every`, Node and C++ `mlEvery`) runs NanoTrack on every
  Nth frame while the two trackers agree. The default is every frame.
- `pure` is gone; `--pure` is refused with a pointer to the new setting.
- CLI: the correlation tracker's options are `--correlation-*` (were
  `--old-tracker-*`). The motion layer for moving cameras (`--correlation-motion`)
  makes fewer wrong claims, treats a box held against the frame border as lost,
  and can estimate camera motion around the target only. `--output`, `--csv` and
  `--json` refuse to overwrite the source video.
- Desktop demo: redesigned, smoother, skips black openings, and saves on macOS.

## v1.0.0 - 2026-11-03

- Three profiles: `default`, `correlation` and `legacy-adaptive`. See the
  [1.0.0 release notes](docs/release-1.0.0.md) for how 0.2.x names map.
- New `correlation`: an improved correlation-filter tracker arbitrated with
  NanoTrack, a 1.8 MB neural tracker; one target. `pure` runs the correlation-filter
  tracker alone, with no neural network.
- `default` is the grey template tracker that 0.2.x called `legacy`.
- `legacy-adaptive` is the tracker that 0.2.x called `adaptive-quality`.
- 0.2.x `legacy`, `adaptive`, `correlation-quality`, `adaptive-quality`, `fast-sky`,
  `birds`, `close-pass`, `missile`, `scale-target` and `mosse` are refused.
- CLI: complete `--help`, `--pure`, arbiter tuning, and a motion-aware layer for
  moving cameras (with `--pure`; see `--help`).
- Published [benchmarks](docs/benchmarks.md): accuracy on 224 public dataset clips
  and 18 field clips, and speed per profile.

## v0.2.2 - 2026-09-12

- Fix Python wheel builds for Linux x86_64 and ARM64 against pinned OpenCV 4.12.0.
- Bundle required OpenCV libraries and license notices into manylinux wheels.
- Retain existing manylinux glibc 2.28 baseline.

## v0.2.1 - 2026-08-26

- Fix macOS builds and tracking differences between C++ toolchains.
- macOS CLI and Python wheel builds explicitly use OpenCV 4.
- Correlation initialization draws training offsets in fixed X/Y order for consistency across compilers.
- Fix small target search window clamp bounds on macOS.

## v0.2.0 - 2026-08-25

- Default profile updated to combine correlation identity tracking with scheduled foreground-box fitting and smoothed geometry.
- Automatic correlation search window sizing (64 px for tiny targets, 128 px through 80 px, 256 px above 80 px).
- Added `legacy` profile to retain v0.1.12 default behavior.
- MultiTracker with identity and bounding box ownership protection during close passes.
- Opt-in `correlation-quality` and `adaptive-quality` profiles.
- Native C++ SDK distribution and quickstart documentation.
- Standalone machine-ID utility and perpetual license support.

## v0.1.0 - 2026-05-08

- Initial evaluation release
- Python SDK: `pip install sky-tracker-sdk`
- Node.js SDK: `npm install @sky-tracker/node`
- Offline Ed25519 licence enforcement - no cloud dependency
- `.lic` file support: drop next to script, no env var needed
- Profiles: `default`, `birds`, `missile`, `pi4-target`
- CSV telemetry: `frame`, `state`, `cx`, `cy`, `confidence`, `speed`, `reason`
- Platforms: Windows x64 verified
