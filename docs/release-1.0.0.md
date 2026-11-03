# Sky Tracker 1.0.0

1.0.0 reduces the tracking profiles to three and adds a new `correlation`
tracker. Several 0.2.x profile names change meaning or are refused, so read
[Upgrading from 0.2.x](#upgrading-from-02x) before you update.

## Profiles

| Profile | What it is | Targets |
|---|---|---|
| `default` | Grey template tracker tuned for constrained CPUs. The recommended starting point. | One or several |
| `correlation` | New. Old-tracker, a correlation-filter tracker, and NanoTrack, a 1.8 MB neural tracker, run on every frame; an arbiter reports one box. `pure` runs old-tracker alone. | One |
| `legacy-adaptive` | Template tracker with correlation assist (strict response checks and camera-motion compensation) and scheduled foreground box fitting. | One or several |

`pi4-target`, `raspi4-target` and `pi-target` remain aliases for `default`.

Accuracy and speed for each profile are in the [1.0.0 benchmarks](benchmarks.md).
On the public drone-detection dataset (224 clips), `correlation` keeps the
centre within 20 px on 0.903 of labelled frames, against 0.770 for `default`.
On our 18 field clips, `default` leads with 0.692 against 0.653.

## The correlation profile

`correlation` runs old-tracker and NanoTrack on every frame. Where they agree it
reports old-tracker's box; where they disagree it picks one from appearance,
contrast, motion and both trackers' scores. Once one tracker has clearly won for
ten frames while the two are on different objects, it restarts the other. The
reported box is steadied, so it does not jump between the two trackers' boxes.

- It tracks one target: `Tracker` accepts it, `MultiTracker` and Android do not.
- NanoTrack needs a build against OpenCV 4.7+ with the `dnn` module, and its two
  model files (Apache 2.0, shipped with the SDK). Where the build lacks `dnn` or
  cannot find the models, `Tracker("correlation")` raises an error that says so.
- `pure` (CLI `--pure`, Python `pure=True`, C++ `Tracker("correlation", true)`,
  Node `pure: true`) runs old-tracker alone. It needs neither `dnn` nor the
  models. Its confidence is 1 while tracking and 0 once lost. It refuses a start
  box so close to the frame edge that its tracking rectangle does not fit.
- CLI only: `--profile correlation --pure --old-tracker-motion` adds a layer for
  moving cameras. It rejects a box that moves with the background, coasts on the
  prediction and re-locks on a moving blob.

## Command line

- `--help` now lists the input, target and output options.
- New options: `--pure`, the `--arbiter-*` options for `correlation`, and the
  `--old-tracker-motion*` options. `--help` lists them with their defaults.

## Upgrading from 0.2.x

| 0.2.x profile | 1.0.0 |
|---|---|
| `legacy` | `default`: identical output |
| `adaptive-quality` | `legacy-adaptive`: identical output |
| `default` | No exact equivalent. Choose `default` (grey template tracker) or `legacy-adaptive` (adds correlation assist and foreground box fitting) |
| `adaptive` | `legacy-adaptive` is closest: the same box fitting with stricter correlation checks |
| `correlation` | `correlation` is now a different tracker; see above |
| `correlation-quality`, `fast-sky`, `birds`, `close-pass`, `missile`, `scale-target`, `mosse` | Refused |

`default` is the grey template tracker that 0.2.x called `legacy`. On all 242
benchmark clips, `default` and `legacy-adaptive` give byte-identical output to
0.2.2 `legacy` and `adaptive-quality`.

Python constructors and result objects, Node calls, C++ SDK calls and tracker
CSV columns are unchanged, apart from the new `pure` argument. Update the Node
package together with the 1.0.0 CLI runtime: the npm package runs an external
executable and does not update it.

## Packaging

- The Linux wheels (x86_64 and aarch64, for Raspberry Pi and Jetson Orin) and
  the macOS wheels include NanoTrack: OpenCV `dnn` and the two model files.
  `sky_tracker.has_nanotrack` says whether a build has it.
- The Jetson C++ package (JetPack 4.6, for Jetson Nano and TX2) ships its own
  OpenCV 4.12 with `dnn` in `opencv/`, and its CLI uses it. JetPack 4's OpenCV
  4.1.1 is too old for NanoTrack. Applications link the bundled copy.
- The library no longer uses `std::filesystem`, so it builds with JetPack 4's
  GCC 7.
- The NanoTrack models install to `share/sky_tracker/models/nanotrack` and sit
  beside the CLI as `sky_tracker_models/nanotrack`. Set
  `SKY_TRACKER_NANOTRACK_MODELS` to use another directory.
- When the library is built with NanoTrack, `find_package(sky_tracker)` also
  requires OpenCV's `dnn` module.
- Shared-library builds carry SOVERSION 1.

## Validation

On macOS (Apple M3 Pro, OpenCV 4.14):

- Builds with and without OpenCV `dnn` pass all 12 test suites.
- The Python and Node smoke tests track 30/30 frames with each profile.
- The benchmark figures were measured on this build.

On Linux aarch64, in Docker on Apple silicon:

- The Python 3.13 wheel (manylinux 2.27/2.28) builds and contains NanoTrack and
  its models.
- The Jetson package, built on Ubuntu 18.04 with GCC 7.5 and OpenCV 4.12, passes
  all 12 test suites, NanoTrack cases included. Its CLI starts with only the
  OpenCV inside the package.

Nothing has yet run on a Raspberry Pi or Jetson device, and Windows, x86_64
Linux and Android builds of 1.0.0 have not been validated.
