# Sky Tracker SDK — C++ Quickstart

Native C++ integration. The library is the tracker itself; the Python and Node
SDKs are bindings over this same code.

**Prerequisites**

- A C++17 compiler and CMake 3.16 or later
- A licence key (see step 2)

You do not need OpenCV: the library carries its own.

---

The SDK version is 1.1.1. Every profile tracks exactly as it does in the CLI and the Python SDK.

## 1. Download the package

Pick yours from the [latest release](https://github.com/Deneb-Systems/sky-tracker-sdk/releases/latest):

| Platform | Package | Needs |
|---|---|---|
| Windows x64 | [`sky-tracker-cpp-windows-x64.zip`](https://github.com/Deneb-Systems/sky-tracker-sdk/releases/latest/download/sky-tracker-cpp-windows-x64.zip) | Visual Studio 2019 or 2022 |
| macOS, Apple silicon | [`sky-tracker-cpp-macos-arm64.tar.gz`](https://github.com/Deneb-Systems/sky-tracker-sdk/releases/latest/download/sky-tracker-cpp-macos-arm64.tar.gz) | macOS 13 or later |
| Linux x86_64 | [`sky-tracker-cpp-linux-x86_64.tar.gz`](https://github.com/Deneb-Systems/sky-tracker-sdk/releases/latest/download/sky-tracker-cpp-linux-x86_64.tar.gz) | glibc 2.31+ (Ubuntu 20.04, Debian 11, RHEL 9 or later) |
| Linux ARM64 | [`sky-tracker-cpp-linux-arm64.tar.gz`](https://github.com/Deneb-Systems/sky-tracker-sdk/releases/latest/download/sky-tracker-cpp-linux-arm64.tar.gz) | glibc 2.27+ (Ubuntu 18.04 or later, Jetson JetPack 4.6 or later, Raspberry Pi OS 64-bit) |

```
include/sky_tracker/sdk.hpp           the entire public API
lib/libsky_tracker.so.1               the library (macOS: lib/libsky_tracker.1.dylib,
                                      Windows: bin/sky_tracker.dll, lib/sky_tracker.lib,
                                      and sky_trackerd.dll/.lib for Debug builds)
lib/cmake/sky_tracker/                CMake package config
share/sky_tracker/models/nanotrack/   NanoTrack models for the correlation profile with ml
examples/cpp/                         two compiling examples
```

The library is a shared library with OpenCV 4.12 linked inside, and it exports
only the `sky_tracker::sdk` API. `sdk.hpp` carries no OpenCV types. So your
application needs no OpenCV, and if it uses one of its own, any version, the two
do not clash. The package's `README.md` covers building the examples and shipping
the library with your program.

---

## 2. Get a licence

Buy a licence at [sky-tracker.deneb-systems.com](https://sky-tracker.deneb-systems.com/#pricing).
The key arrives by email right after payment, as text and as a `sky_tracker.lic`
file, and works on every device you run the SDK on: there is no machine ID to
send. Students get a [free one-year licence](https://sky-tracker.deneb-systems.com/student/).
To try the tracker first, use the free demo app from the website, or get a
[free 7-day trial key](https://sky-tracker.deneb-systems.com/trial/) for the full SDK.

`sky_tracker::sdk::machineId()` still prints this machine's ID, for keys we
issued to a single unit before 1.0.

---

## 3. Install your licence

Any one of these:

```bash
export SKY_TRACKER_LICENSE_KEY=<your-token>
export SKY_TRACKER_LICENSE_FILE=/path/to/sky_tracker.lic
```

…or place `sky_tracker.lic` next to your executable, in the working directory,
or at `~/.sky_tracker/licence`. Validation of bought and student licences is fully
offline — nothing contacts a network at runtime. A 7-day trial key (it starts
with `SKT2.`) is the exception: when a tracker starts, the SDK confirms it with
the licence server over HTTPS, so a trial needs an internet connection, and
1.1.1 or later.

Call `validateLicense()` at startup to fail fast with a clear message:

```cpp
try {
    const auto info = sky_tracker::sdk::validateLicense();
    std::printf("licensed to %s until %s\n",
                info.customer.c_str(), info.expiresAt.c_str());
} catch (const sky_tracker::sdk::Error& e) {
    std::fprintf(stderr, "licence: %s\n", e.what());
    return 1;
}
```

---

## 4. Build against it

```cmake
cmake_minimum_required(VERSION 3.16)
project(my_app LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)

find_package(sky_tracker REQUIRED)

add_executable(my_app main.cpp)
target_link_libraries(my_app PRIVATE sky_tracker::sky_tracker)
```

```bash
cmake -S . -B build -DCMAKE_PREFIX_PATH=/path/to/sky-tracker
cmake --build build
```

---

## 5. Track a target

```cpp
#include <sky_tracker/sdk.hpp>

namespace st = sky_tracker::sdk;

st::Tracker tracker("default");

// Point at your capture buffer. No copy is taken for Gray8 or Bgr8.
st::FrameView frame;
frame.data   = capture.pixels();
frame.width  = capture.width();
frame.height = capture.height();
frame.stride = capture.rowBytes();   // 0 if rows are tightly packed
frame.format = st::PixelFormat::Bgr8;

// Acquire from an operator selection or your own detector.
tracker.lock(frame, st::Rect{x, y, w, h});

while (capture.grab()) {
    const st::TrackResult r = tracker.update(frame, dtSeconds);

    const float errorX = r.centerX - frame.width  / 2.0f;
    const float errorY = r.centerY - frame.height / 2.0f;
    panTilt.drive(errorX, errorY);
}
```

`update()` is synchronous and returns the measurement for that frame. There is
no internal buffering or worker thread to account for in your control loop.

### Driving a pan-tilt loop

`r.predictionOnly` is the field that matters most. When the target is briefly
occluded the tracker keeps extrapolating its motion model rather than dropping
the position, so you can keep slewing through the gap instead of stalling. The
estimate degrades the longer it persists — `r.misses` counts the consecutive
extrapolated frames, and a sensible controller reduces gain or holds after a
threshold you choose.

`r.state` is the other signal to key off: drive normally while it is
`Confirmed`, and treat `Lost` as a hold rather than a reason to recentre.

---

## 6. Profiles

Pass a profile name to the constructor:

| Profile | Use |
|---|---|
| `default` | Balanced starting point, grayscale-only, constrained-CPU policy; several targets |
| `correlation` | An improved correlation-filter tracker, no neural network; with `ml`, plus the NanoTrack neural tracker, arbitrated; single target only |
| `legacy-adaptive` | Template tracker with correlation assist (strict checks, camera-motion compensation) and scheduled foreground box fitting; several targets |

`availableProfiles()` returns this list at runtime. Which one wins depends on
your camera and target mix: see the [benchmarks](benchmarks.md), then
evaluate against your own footage.

### The correlation profile

`correlation` runs an improved correlation-filter tracker: no neural network
and no model files. It reports `r.confidence` `1` while tracking and `0` when
it has lost the target, and throws `Error` from `lock()` if the box sits so
close to the frame edge that its tracking rectangle does not fit.

With `TrackerOptions::ml` it also runs NanoTrack, a small neural tracker.
Where the two agree it reports the correlation tracker's box; where they
disagree it picks one from appearance, contrast, motion and both trackers'
scores, and restarts the other tracker once one has clearly won for ten frames
while the two are on different objects. By default NanoTrack runs on every
frame, exactly as 1.0.0's `correlation` did. `mlEvery = N` runs it on every Nth
frame while the two agree (and on every frame while they disagree or the target
is lost): about half the cost at N = 3, but on fast footage the box can jump
between checks.

```cpp
sky_tracker::sdk::Tracker tracker("correlation");          // the correlation tracker alone

sky_tracker::sdk::TrackerOptions options;
options.ml = true;                                          // plus NanoTrack, every frame
sky_tracker::sdk::Tracker checked("correlation", options);
```

Every package includes NanoTrack. (A library you build yourself needs OpenCV
4.7+ with the `dnn` module for it; without it `ml` throws `Error` and plain
`correlation` still works.) It loads two model files from the first of:

1. `$SKY_TRACKER_NANOTRACK_MODELS`
2. `sky_tracker_models/nanotrack/` beside the library
3. `share/sky_tracker/models/nanotrack/` in the parent of the library's folder,
   which is where the package keeps them
4. the install prefix the library was configured with

Ship `share/sky_tracker/models/nanotrack/` (including its `LICENSE`, Apache 2.0)
with your application, in the same place relative to the library, or point
`SKY_TRACKER_NANOTRACK_MODELS` at it. The `Tracker` constructor
throws `Error`, listing every directory it tried, when they are missing.

`r.confidence` says how the frame was resolved: `1` both trackers agree,
`0.75` a clear arbitration, `0.5` a close one, or NanoTrack alone while it still
matches the target's appearance. Below `0.5` (NanoTrack alone, appearance not
supported) the result is reported `Lost`. A controller that needs fewer wrong
frames can treat anything under `0.75` as a hold.

The reported box is steadied: it moves to the other tracker's box only after
that tracker has won three frames in a row (at once when the trackers are on
different objects) and its centre and size are smoothed, so a gimbal loop
does not see the two trackers' slightly different boxes as motion.

---

## 7. Multiple targets

```cpp
st::MultiTracker multi("default");
multi.lock(frame, {st::Rect{...}, st::Rect{...}});

for (const st::TrackResult& r : multi.update(frame, dt)) {
    // r.targetId is stable across frames, including through close passes.
}
```

During an ambiguous close pass the tracker holds identity rather than guessing:
affected targets report `interacting`, and may report `suppressed` with the ID
that claimed the observation instead of jumping onto a neighbouring object.

---

## Scope

Sky Tracker is a **selected-target** tracker. Every target starts from a
bounding box you supply, from an operator selection or your own detector. It
does not perform automatic search and acquisition.

---

## Errors

Everything throws `sky_tracker::sdk::Error` (derived from `std::runtime_error`)
— invalid arguments, `update()` before `lock()`, and licence failures. Neither
tracker is thread-safe; serialise calls or give each thread its own instance.
