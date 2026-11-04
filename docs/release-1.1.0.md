# Sky Tracker 1.1.0

1.1.0 makes `correlation` the correlation tracker on its own and moves
NanoTrack behind a new `ml` option. Tracking is unchanged: every 1.0.0 setting
has a 1.1.0 equivalent with identical output. Some names change, so read
[Upgrading from 1.0.0](#upgrading-from-100) before you update.

## Profiles

| Profile | What it is | Targets |
|---|---|---|
| `default` | Grey template tracker tuned for constrained CPUs. | One or several |
| `correlation` | An improved correlation-filter tracker, with no neural network. Add `ml` for NanoTrack's second opinion. | One |
| `legacy-adaptive` | Template tracker with correlation assist and scheduled foreground box fitting. | One or several |

Accuracy and speed for each setting are in the [benchmarks](benchmarks.md).

## The ml option

`ml` adds NanoTrack, a 1.8 MB neural tracker, to `correlation`, with an arbiter
that reports one box. This is exactly 1.0.0's `correlation`.

- CLI `--profile correlation --ml`, Python `Tracker("correlation", ml=True)`,
  C++ `Tracker("correlation", options)` with `sdk::TrackerOptions::ml` set,
  Node `{ profile: "correlation", ml: true }`.
- `ml_every` (CLI `--ml-every N`, C++ `mlEvery`, Node `mlEvery`) runs NanoTrack
  on every Nth frame while the two trackers agree, and on every frame while they
  disagree or the target is lost. The default is 1, every frame. At N = 3 the
  averages over the public clips hold and the update time roughly halves, but on
  fast footage the box can jump between checks, so raise it only where `ml`
  cannot keep up and check the result on your own footage.
- `ml` needs a build with OpenCV `dnn` and the two model files, as before. Plain
  `correlation` needs neither. `ml` with another profile is refused.

## Command line

- The correlation tracker's options are now `--correlation-*` (1.0.0:
  `--old-tracker-*`), for example `--correlation-motion` for the layer for
  moving cameras. The CSV reasons follow: `pick-correlation` and
  `+restart-correlation`.
- The motion layer (`--correlation-motion`, off by default) makes fewer wrong
  claims: on our labelled field clips, wrong-while-tracking frames fall by
  0.043 and centre-within-20-px rises by 0.025 against 1.0.0's layer.
  - A box held against the frame border for 5 frames counts as lost
    (`--correlation-motion-edge-frames`).
  - A coast that leaves the frame ends as lost.
  - `--correlation-motion-camera-local X` estimates the camera motion only
    around the target, which helps on fisheye FPV footage.
- `--output`, `--csv` and `--json` refuse a path that names the source video,
  instead of overwriting it.

## Upgrading from 1.0.0

| 1.0.0 | 1.1.0 |
|---|---|
| `correlation` | `correlation` with `ml` (`--ml`, `ml=True`, `TrackerOptions::ml`, `ml: true`): identical output |
| `correlation`, pure (`--pure`, `pure=True`, `Tracker("correlation", true)`, `pure: true`) | `correlation`: identical output |
| `--old-tracker-*` options | `--correlation-*` options |
| `default`, `legacy-adaptive` | Unchanged |

- `--pure` is refused with a message that names the new setting.
- Python: `pure=` is gone, and `ml` and `ml_every` are keyword-only, so a
  1.0.0 call with a positional `pure` fails instead of silently changing
  meaning.
- C++: the `Tracker(profile, bool pure)` constructor is replaced by
  `Tracker(profile, const TrackerOptions&)`.
- Node: `pure` is ignored (plain `correlation` already runs without
  NanoTrack). Update the Node package together with the 1.1.0 CLI runtime,
  which is the first to accept `ml`.

Result objects, CSV columns, licences and the other profiles are unchanged.
1.x licence keys work with 1.1.0.

## Desktop demo

The free demo app is redesigned:

- It offers Correlation (selected first), Correlation + ML and Default.
- Tracking and video encoding run on their own threads, paced to the clip, so
  the window no longer stutters.
- Leading black frames are skipped.
- Saving the clip works on macOS.

## Validation

On macOS (Apple M3 Pro, OpenCV 4.14):

- All 12 test suites pass.
- On 241 of the 242 benchmark clips (the 242nd clip's video has changed
  since), `correlation` gives output identical to 1.0.0 `correlation` pure,
  and `correlation --ml` the same boxes, states and scores as 1.0.0
  `correlation`; only the renamed CSV reasons differ.
- The Python and Node smoke tests pass with every profile.
- The macOS and Linux (x86_64 and arm64) demo packages track ten seconds of a
  test clip with each of their three settings, none lost. The Windows package
  is cross-compiled from the same source and was not run for this release.

Nothing has yet run on a Raspberry Pi or Jetson device, and Windows, x86_64
Linux and Android builds of 1.1.0 have not been validated on hardware.
