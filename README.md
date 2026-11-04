# Sky Tracker SDK

Sky Tracker is a CPU-friendly SDK for following small, fast-moving targets in
video. The caller selects each target with an initial bounding box; Sky Tracker
then maintains its position, identity, motion state, and visible box over time.

Current version: 1.1.0. Coming from 1.0.0, read the
[1.1.0 release notes](docs/release-1.1.0.md): `pure` is gone and NanoTrack moved
behind `ml`. Coming from 0.2.x, read the [1.0.0 release notes](docs/release-1.0.0.md)
too: profile names changed.

Website, licences and the free demo app:
[sky-tracker.deneb-systems.com](https://sky-tracker.deneb-systems.com).

It is a selected-target tracker, not a general object detector. Use your own
detector, telemetry, or an interactive ROI to provide the initial
`(x, y, width, height)` box.

## Try it without writing code

The free demo app opens any video and tracks ten seconds with Correlation,
Correlation + ML or Default:
[macOS](https://github.com/Deneb-Systems/sky-tracker-sdk/releases/latest/download/Sky-Tracker-Demo-macOS.dmg),
[Windows](https://github.com/Deneb-Systems/sky-tracker-sdk/releases/latest/download/Sky-Tracker-Demo-Windows-x64.zip),
[Linux x86_64](https://github.com/Deneb-Systems/sky-tracker-sdk/releases/latest/download/Sky-Tracker-Demo-Linux-x86_64.AppImage),
[Linux ARM64](https://github.com/Deneb-Systems/sky-tracker-sdk/releases/latest/download/Sky-Tracker-Demo-Linux-arm64.AppImage).
It is not code-signed yet, so macOS and Windows ask before the first run.

## Install

Python 3.10 or later:

```bash
pip install sky-tracker-sdk opencv-python
```

The Node.js wrapper is also available:

```bash
npm install @sky-tracker/node
```

## Licence setup

A valid licence is required when a tracker is created. Buy one at
[sky-tracker.deneb-systems.com](https://sky-tracker.deneb-systems.com/#pricing):
the key arrives by email right after payment and works on every device, with no
machine ID to send.

Set the licence key in the environment:

```powershell
$env:SKY_TRACKER_LICENSE_KEY = "<your-token>"
```

```bash
export SKY_TRACKER_LICENSE_KEY="<your-token>"
```

Alternatively, place `sky_tracker.lic` beside your script or point
`SKY_TRACKER_LICENSE_FILE` at it.

## Single-target quick start

```python
import cv2
import sky_tracker

capture = cv2.VideoCapture("video.mp4")
fps = capture.get(cv2.CAP_PROP_FPS) or 30.0
dt = 1.0 / fps

ok, frame = capture.read()
if not ok:
    raise RuntimeError("could not read video")

tracker = sky_tracker.Tracker("default")
tracker.lock(frame, bbox=(469, 409, 26, 38))

while True:
    result = tracker.update(frame, dt)
    print(result.state, result.center(), result.bbox(), result.confidence)

    ok, frame = capture.read()
    if not ok:
        break

capture.release()
```

`tracker.initialize(frame, bbox)` is an alias for `tracker.lock(frame, bbox)`.
Pass `dt` in seconds when processing a video file; omit it for live input when
wall-clock timing is appropriate.

## Multi-target quick start

Use `MultiTracker` when the caller selects several targets. Initial bbox order
defines stable `target_id` values: the first box is target 1, the second is
target 2, and so on.

```python
multi = sky_tracker.MultiTracker("default")
multi.lock(
    frame,
    [
        (469, 409, 26, 38),
        (520, 390, 24, 34),
    ],
)

for result in multi.update(frame, dt):
    print(
        result.target_id,
        result.state,
        result.bbox(),
        result.interacting,
        result.prediction_only,
    )
```

When selected targets pass close to one another, the multi-tracker protects
identity and bbox ownership. It can freeze model or geometry learning during an
ambiguous interaction and coast briefly from motion instead of assigning both
tracks to the same observation. The result exposes this state through
`interacting`, `suppressed`, `suppressed_by_id`,
`geometry_ownership_constrained`, and `prediction_only`.

## Profiles

| Profile | What it is | Targets |
| ------- | ---------- | ------- |
| `default` | Grey template tracker tuned for constrained CPUs. The recommended starting point. | One or several |
| `correlation` | An improved correlation-filter tracker, with no neural network. `ml=True` adds NanoTrack, a 1.8 MB neural tracker, and an arbiter that reports one box. | One (`Tracker` only) |
| `legacy-adaptive` | Template tracker with correlation assist (strict checks, camera-motion compensation) and scheduled foreground box fitting. | One or several |

```python
tracker = sky_tracker.Tracker("correlation")                        # correlation filter alone
tracker = sky_tracker.Tracker("correlation", ml=True)               # plus NanoTrack, every frame
tracker = sky_tracker.Tracker("correlation", ml=True, ml_every=3)   # NanoTrack every 3rd agreeing frame
```

The Linux (x86_64 and aarch64) and macOS wheels and the Jetson C++ package
include NanoTrack; `sky_tracker.has_nanotrack` says whether a build has it.
Without it, `ml=True` raises and plain `correlation` still works. On a
Raspberry Pi or Jetson, NanoTrack is the slow part: measure `ml=True` on the
device, and raise `ml_every` or use plain `correlation` or `default` if it
cannot keep up. `ml_every` above 1 saves time but can let the box jump on fast
footage.

Which profile wins depends on the footage:

| Centre within 20 px of the label | 224 public dataset clips | 18 field clips |
| -------------------------------- | ------------------------ | -------------- |
| `default`                        | 0.770                    | **0.692**      |
| `correlation`, `ml=True`         | **0.903**                | 0.653          |
| `correlation`                    | 0.851                    | 0.602          |
| `legacy-adaptive`                | 0.831                    | 0.626          |

Start with `default`, and compare profiles on labelled frames of your own
footage before changing a production configuration. Full results, metric
definitions and speed are in [benchmarks](docs/benchmarks.md).

## Python examples

### Interactive single target

Select one object, optionally starting later in the video, and save an annotated
MP4 with a zoom inset:

```bash
python examples/track_video.py video.mp4 --start-seconds 10 --profile default
python examples/track_video.py video.mp4 --profile correlation          # correlation filter alone
python examples/track_video.py video.mp4 --profile correlation --ml     # plus NanoTrack
```

Source: [`examples/track_video.py`](examples/track_video.py)

### Interactive multiple targets

Select targets in stable ID order. After selecting at least two boxes, press
Esc on the next selection dialog to start tracking. `MultiTracker` accepts
`default` and `legacy-adaptive`; `correlation` tracks one target:

```bash
python examples/track_multi_video.py video.mp4 --profile default
```

Source: [`examples/track_multi_video.py`](examples/track_multi_video.py)

### Headless bbox and telemetry

Use a known bbox without opening a selection window. CSV telemetry is always
written; annotated video is optional:

```bash
python examples/track_from_bbox.py video.mp4 \
  --bbox 469,409,26,38 \
  --start-seconds 10 \
  --csv tracks.csv \
  --output tracked.mp4
```

Source: [`examples/track_from_bbox.py`](examples/track_from_bbox.py)

### Machine ID

Only for keys issued to a single machine before 1.0; keys bought on the website
work everywhere.

```bash
python examples/machine_id.py
python examples/machine_id.py --raw
```

Source: [`examples/machine_id.py`](examples/machine_id.py)

Every example supports `--help`.

## Common result fields

| Field                                            | Meaning                                                              |
| ------------------------------------------------ | -------------------------------------------------------------------- |
| `state`, `lost`                                  | Current lifecycle state and whether the target is lost.              |
| `cx`, `cy`, `center()`                           | Target centre in pixels.                                             |
| `bbox_x`, `bbox_y`, `bbox_w`, `bbox_h`, `bbox()` | Visible target box.                                                  |
| `confidence`                                     | Tracker confidence from 0 to 1.                                      |
| `speed`                                          | Estimated image-plane speed in pixels per second.                    |
| `reason`                                         | Decision label useful for telemetry and debugging.                   |
| `prediction_only`                                | The output is a tentative motion coast, not an accepted observation. |
| `annotate()`                                     | A copy of the current frame with the SDK overlay.                    |

## Platforms

| Platform               | Status                                          |
| ---------------------- | ----------------------------------------------- |
| macOS (Apple silicon)  | 1.1.0 built, tested and benchmarked             |
| Windows x64            | Verified for 0.2.2; 1.x not yet validated       |
| Linux x86_64           | Verified for 0.2.2; 1.x not yet validated       |
| Raspberry Pi 4 (ARM64) | Verified for 0.2.2; 1.0.0 aarch64 wheel built and checked in Docker, not yet on a Pi |
| Jetson (JetPack 4.6)   | 1.0.0 C++ package built and tested in Docker, not yet on a Jetson |

## Documentation

- [Python quickstart](docs/quickstart-python.md)
- [Node.js quickstart](docs/quickstart-node.md)
- [C++ quickstart](docs/quickstart-cpp.md)
- [Benchmarks](docs/benchmarks.md)
- [1.1.0 release notes](docs/release-1.1.0.md)
- [1.0.0 release notes](docs/release-1.0.0.md)
- [0.2.2 release notes](docs/release-0.2.2.md)
