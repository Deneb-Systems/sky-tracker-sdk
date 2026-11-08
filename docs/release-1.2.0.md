# Sky Tracker 1.2.0

1.2.0 adds `MotionTracker`, which finds and follows moving objects with no
bounding box to start from, and its `bright-diff` mode is free: it runs without a
licence key. The `correlation` profile is faster and lighter, re-locks a target
that left its window and fits its box tighter, and Python 3.14 wheels are new
too. Everything else is unchanged, and code written for 1.1 runs
as before.

## MotionTracker

Until now every target started from a box you supplied. `MotionTracker` finds
whatever moves in each frame and follows each object with its own stable ID:
drones and birds against the sky, or meteors. It is the motion pipeline behind
the command line's motion mode, now in the C++ and Python SDKs.

```python
tracker = sky_tracker.MotionTracker(mode="bright-diff", moving_only=True)
for track in tracker.update(frame, dt=1 / fps):
    print(track.id, track.center(), track.speed)
```

- Three ways to find motion: `background` (background subtraction, tolerates
  slow light changes), `diff` (frame differencing) and `bright-diff` (what got
  brighter since the last frame, which puts a meteor's box on its head rather
  than its trail).
- `moving_only` drops objects that have not moved 3 px, such as twinkling stars
  and sensor noise.
- `region` limits detection to part of the frame, for example the sky above the
  horizon.
- Each object reports its box, centre, velocity and speed in pixels per second,
  how long it has been followed and whether this frame's position is a
  prediction.
- `reset()` starts again after a scene cut; IDs keep counting up.

It needs a static or slowly moving camera, and it finds objects by motion only:
it does not recognise what they are.

## Free: bright-diff

`MotionTracker` in `bright-diff` mode needs no licence key, for any use. It finds
what got brighter on a still camera, which is what meteors, fireballs and other
sky events need; [Meteor Cut](https://github.com/Deneb-Systems/meteor-cut), the
open-source meteor highlights tool, is built on it. The `background` and `diff`
modes, `Tracker`, `MultiTracker` and the command line still need a key, as
before.

## Faster correlation tracker

The `correlation` profile's update takes a third to three quarters less time,
and holds hundreds of megabytes less memory, with the same tracking:

| Clip | Window | 1.1.1 | 1.2.0 |
|---|---|---|---|
| 1920×1080, 55 px target | 256 | 1.83 ms | 1.15 ms |
| 1920×1080, 22 px target | 128 | 0.53 ms | 0.27 ms |
| 1280×720, 80 px target | 256 | 1.72 ms | 1.15 ms |
| 640×512, 26 px target | 128 | 0.38 ms | 0.26 ms |
| 320×256, 10 px target | 128 | 0.35 ms | 0.25 ms |

Update time per frame on one Apple M3 Pro core, both builds timed in the same
session, median of three runs over 300 frames, without video decoding. The
absolute figures move with the machine's state (an earlier session measured
both builds about 1.8x slower); the saving, a third to a half, holds. Three
things changed:

- The tracker no longer copies every frame into its 256-frame history (about
  530 MB at 1080p, and a 2 MB allocation per frame); the history is kept only
  for the temporal averaging modes, which are off by default.
- The debug views (pattern, mask, correlation) are rendered only when asked for.
- The filter's Hann support is applied in the frequency domain as an exact
  3-tap convolution instead of an inverse and a forward DFT, and the remaining
  per-pixel loops use OpenCV's vectorised spectrum operations.

The last change uses the periodic form of the Hann window for the support (the
input window is unchanged), so boxes can differ from 1.1.1 by a fraction of a
pixel. On the 224 public benchmark clips the difference is within noise (within
20 px: 0.849 against 0.851, 95% interval −0.005 to +0.001; mean IoU 0.430
against 0.431), and so on the 17 field clips (0.606 against 0.618, interval
−0.049 to +0.018). `correlation --ml` and the motion layer share the gain.

## Correlation tracker: re-lock and a tighter box

Two changes to what the `correlation` profile reports, both measured on the
224 public clips and the 17 field clips (`run_drone_benchmark.py`):

- **A wider search while weak.** When the tracker has lost its target for four
  frames, it also correlates its filter at the eight window positions around
  the search window and re-locks on a neighbour whose peak clears the usual
  return threshold. Before, a target that left the window was gone until it
  happened to come back. Public clips, both changes together against 1.1.1:
  within 20 px 0.864 against 0.851 (95% interval +0.001 to +0.028), within
  40 px 0.895 against 0.880, frames claimed 0.980 against 0.962, wrong frames
  unchanged. Field clips: within 10 px +0.010, the rest within noise. It costs eight extra correlations on a weak frame and nothing
  otherwise. `--correlation-weak-search N` sets the delay (0 turns it off),
  `--correlation-weak-search-every N` the cadence.
- **Object box margin.** The measured object extent gets 4 px plus 10% per axis
  instead of a fixed 8 px, which made boxes 1.3x too wide on small targets.
  Public clips, both changes together: mean IoU 0.453 against 0.431, boxes
  with IoU over 0.5 0.485 against 0.429; field clips (larger targets) within noise. `--correlation-object-margin`
  and `--correlation-object-margin-ratio` set both terms; `8` and `0` restore
  the old box.

- **A PSR floor.** The tracker's adaptive threshold cycles between a few units
  and the target's own PSR every few frames, so a peak of 6 or 7, which is what
  empty sky gives, was accepted whenever the cycle was low, and a faint or
  vanished target was followed as "tracking" instead of being reported weak.
  Peaks below PSR 6 are now never accepted. Public clips: wrong frames 0.089
  against 0.090 (significant), the rest unchanged; field clips: within 20 px
  0.650 against 0.598 and wrong frames 0.184 against 0.237, because the weak
  state now starts the neighbour search. `--correlation-min-psr X` sets it, 0
  restores the old behaviour.

Also new, off by default because they measured neutral or did not help on the
clips they were built for: `--correlation-support-scale X` confines the filter
to X tracking rectangles, and `--correlation-distractor` prefers the response
peak nearest the prediction when two are close.

Scale estimation is in as an option, off by default: `--correlation-scale-step X`
(for example 1.05) keeps the learned filter at its model size, cuts the window
at the current scale, and every `--correlation-scale-every` accepted frames
also correlates the window one step larger and smaller, moving to a scale whose
peak wins by `--correlation-scale-margin`. It follows a zoom in a controlled
test and helps the field clips whose target grows (video_1 within 20 px 0.25 to
0.58, video_8 0.33 to 0.42, video_74 0.27 to 0.33), but the peak comparison is
noisy on small targets and drifts on steady ones: on the 224 public clips it
costs 0.011 within 20 px and 0.018 mean IoU, and over all 20 field clips the
gain is within noise. Guards (`--correlation-scale-min-side`,
`--correlation-scale-confirm`, `--correlation-scale-grow-only`) remove the
public loss and the field gain together, or worse. Use it for footage of
approaching targets; a proper scale filter is the next step.

Re-acquisition after the target leaves the frame is in as an option, off by
default: `--correlation-reacquire N`. While the target is held fully inside
the frame the tracker keeps a template of it; when the target is followed out
of the frame, or lost within half a search window of the border, the tracker
no longer gives up after 128 frames, and from the Nth lost frame on it matches
the template over the whole frame every `--correlation-reacquire-every` frames
(normalised correlation of at least `--correlation-reacquire-score`, 0.6,
on a frame downscaled to about 640 px), re-locking where the filter confirms
the match on two frames in a row. After the target was followed out of the
frame only this search brings it back; the usual return and the neighbouring
windows would latch onto whatever is at the border. With N = 5: public clips
within 20 px +0.003 (significant), wrong frames -0.001; field clips within
20 px +0.022, wrong +0.021 (n.s.). It re-finds the jet in video_14 (0.50 to
0.70) and the glider in flight_5 (0.25 to 0.50); in video_31 the box had
already latched onto buildings before it left the frame, so it re-finds the
buildings. The search costs one downscaled whole-frame match every few lost
frames.

And
`--correlation-predict` searches at the position the smoothed velocity predicts
and coasts along it while weak (`--correlation-velocity-smoothing`,
`--correlation-coast-damping`).

`correlation --ml` and `--correlation-motion` run the same filter and inherit
these changes; their numbers were not re-measured.

## Python 3.14

Wheels are now built for Python 3.10 to 3.14 on every platform.

C++: `sky_tracker::sdk::MotionTracker` with `MotionOptions` and `MotionTrack`
in `sdk.hpp`. Python: `sky_tracker.MotionTracker` and `MotionTrack`. See the
quickstarts for every option.

## Compatibility

1.2.0 is a drop-in update for 1.x: every existing type and function is
unchanged, and your licence key covers it.
