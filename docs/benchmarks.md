# Sky Tracker 1.0.0 Benchmarks

Every number here comes from the 1.0.0 command-line tracker, started headless
from the first labelled box of each clip with one profile and no other options.

## Accuracy

The metrics are computed per clip and then averaged over clips, so every clip
counts equally:

- **Centre within 10 / 20 / 40 px**: share of labelled frames where the
  reported centre lies within that distance of the label's centre. Frames the
  tracker reports as lost count as misses.
- **Wrong while tracking**: of the frames where the tracker claims the target,
  the share more than 40 px from the label.
- **Mean IoU** and **IoU ≥ 0.5**: overlap between the reported box and the
  labelled box, with lost frames counting as 0.

### Public drone-detection dataset: 224 clips

The drone-detection dataset by Svanström et al.: 224 single-target clips, 112
visible (640×512) and 112 infrared (320×256), 14 per modality and class
(airplane, bird, drone, helicopter), with every frame labelled.

| Profile | Within 10 px | Within 20 px | Within 40 px | Wrong while tracking | Mean IoU | IoU ≥ 0.5 |
|---|---|---|---|---|---|---|
| `default` | 0.736 | 0.770 | 0.817 | 0.133 | 0.351 | 0.309 |
| `correlation` | **0.865** | **0.903** | **0.922** | **0.055** | **0.464** | **0.469** |
| `correlation`, pure | 0.807 | 0.851 | 0.880 | 0.090 | 0.431 | 0.429 |
| `legacy-adaptive` | 0.786 | 0.831 | 0.864 | 0.093 | 0.363 | 0.309 |

`correlation` is ahead of `default` on every metric here. The 95% interval for
its "within 20 px" gain is +0.09 to +0.18. Its arbitration model was fitted on
disagreement frames from these clips, which flatters it slightly. Pure refuses
to start on one clip, whose target sits at the frame edge; that clip counts as
missed.

### Field footage: 18 clips

Our own collection: FPV and ground-camera footage of quadcopters, fixed-wing RC
models, light aircraft, an airliner, birds and a rally car. Labels are sparse:
225 hand-labelled frames, about 12 per clip.

| Profile | Within 10 px | Within 20 px | Within 40 px | Wrong while tracking | Mean IoU |
|---|---|---|---|---|---|
| `default` | **0.590** | **0.692** | **0.730** | **0.168** | 0.333 |
| `correlation` | 0.527 | 0.653 | 0.715 | 0.250 | **0.424** |
| `correlation`, pure | 0.494 | 0.602 | 0.677 | 0.237 | 0.364 |
| `legacy-adaptive` | 0.558 | 0.626 | 0.696 | 0.194 | 0.299 |

Here `default` leads on centre accuracy and `correlation` fits the box best.
With 18 clips, the gap between them is not significant (within 20 px: −0.04,
95% interval −0.18 to +0.09).

Which profile wins depends on the footage. Run the candidates on labelled
frames of your own camera before you choose.

## Speed

Frames per second through the whole command-line pipeline (video decode and
tracking, one target, no display or output files). Each figure is the median of
three runs on one clip per resolution, rounded down to two significant figures.
They were measured on an Apple M3 Pro (11-core CPU) with macOS 26 and OpenCV
4.14, with no GPU use. Runs varied by about ±15%.

| Profile | 1920×1080 | 1280×720 | 640×512 |
|---|---|---|---|
| `default` | 150 | 410 | 1,000 |
| `correlation` | 56 | 67 | 100 |
| `correlation`, pure | 170 | 220 | 1,000 |
| `legacy-adaptive` | 75 | 57 | 290 |

Speed depends on the clip as well as the frame size: `legacy-adaptive` is slower
on the 720p clip than on the 1080p one.
`correlation` runs NanoTrack, a 1.8 MB neural network, on the CPU every frame.

These are desktop figures. Raspberry Pi and Jetson throughput has not been
measured for 1.0.0; measure it on the target device at the intended resolution.
Expect `correlation` to be much slower there than the other profiles. NanoTrack
takes about 10 ms per frame on one M3 Pro core, and more cores barely help,
whereas pure takes 0.5 ms. Pi and Jetson CPU cores are several times slower.
Where `correlation` cannot keep up, `pure` and `default` usually can.
