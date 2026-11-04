# Changelog

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
