# Sky Tracker 1.1.1

1.1.1 adds free 7-day trial keys and options for tracking meteors on a static
camera. Tracking with the settings you already use is unchanged.

## 7-day trial keys

Anyone can now try the full SDK (C++, Python and Node.js, every profile) for 7
days: ask for a key at
[sky-tracker.deneb-systems.com/trial](https://sky-tracker.deneb-systems.com/trial/)
and it arrives by email.

- Trial keys start with `SKT2.` and need 1.1.1 or later. Earlier versions
  report `unknown license token version: SKT2`.
- When a tracker starts, the SDK confirms a trial key with the licence server
  over HTTPS, so a trial needs an internet connection. It sends the key's ID and
  a random number, nothing about your video. The server's signed answer, not the
  computer's clock, decides when the trial ends.
- On Windows the check uses WinHTTP. On macOS and Linux it loads the system's
  libcurl only when a trial key is used (on Linux, install `libcurl4` if it is
  missing). Trial keys are not supported on Android.
- Bought and student licences are unaffected: they are checked fully offline
  and never contact the network. The C++ package gains no new dependency.

## Meteors and other targets that brighten

All opt-in on the CLI; output without them is unchanged.

- `--mode bright-diff` detects what got brighter since the last frame, so a
  meteor is boxed at its head instead of one frame behind on its trail. Static
  camera only.
- `--moving-only` reports a motion track only once it has moved 3 px, and drops
  coasting predictions and raw detection outlines from the overlay.
- `--box-padding PX` grows every reported box (video, CSV and JSON), in every
  mode.
- `--target-end-when-still N` ends a locked target after N frames with no change
  in its box, for targets that burn out or leave the frame.
