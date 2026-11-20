---
name: openlinc
description: Build real-time programs on live Somnolinc headband data from the local bridge — events for looks to the left and right (hands-free control with the eyes), raw EEG and its broadband power (muscle activity such as a jaw clench, blinks), head orientation and motion (IMU), ambient light, and whether the headband is connected and streaming. Use this whenever the user wants to read, react to, plot, record or analyse the live headband stream in this folder, or build any application that interfaces with the headband or is controlled with the eyes or the head (desktop applications and tools, hands-free control, live displays, recorders, experiments, installations, games, in Python or in any tool that speaks OSC), even if they do not mention OSC, the bridge or the phone.
---

# Live headband data

The Somnolinc phone app streams the headband's raw data over Wi-Fi to the **bridge**, a Python
program in this folder. Programs on this PC get the data from the bridge over localhost, as OSC
over UDP. The bridge also starts a processing service, which computes results from the raw data,
such as looks to the left and right, and the bridge passes those on like the raw data. In Python,
`control` from the `openlinc` package receives and buffers everything with headband times; prefer
it to writing socket or decoding code.

This serves any application that interfaces with the headband, not one kind of program: desktop
applications and tools, hands-free control of other software, live displays and recorders,
experiments and measurements, installations, and games. "Program" below means any of these.

## Further files

This file is enough for a program that reads streams and the state in Python. Read the file for
what the program does beyond that, before writing it:

| File | Read it when the program |
| --- | --- |
| [signals.md](references/signals.md) | computes on samples: head axes and orientation, how samples get their time, what the EEG channels measure, each message's fields |
| [actions.md](references/actions.md) | reacts to looks to the left and right, blinks or jaw clenches; it holds the calibration these need |
| [live-data-visualization.md](references/live-data-visualization.md) | shows live data: a dashboard, a live plot, a 3D view; how to show the EEG and the light in any view |
| [osc-protocol.md](references/osc-protocol.md) | speaks OSC with the bridge itself instead of using the Python package: the messages, their arguments and how to subscribe |

## Before building

1. Check that data arrives: `uv run bridge status`. It prints JSON with these keys:
   - `link`: `waiting`, `linked` or `not_responding` (see step 3).
   - `phone`: the linked phone's address and port, or null.
   - `seconds_since_last_message`: time since the phone last sent anything, or null.
   - `lost_datagrams`: data messages lost between phone and bridge.
   - `malformed_datagrams`: messages the bridge could not read.
   - `subscribers`: local programs currently receiving the stream.
   - `service`: `running` or `not_running`, the processing service that computes the events.
   - `streams`: one entry per address, with `rate_hz` (measured), `expected_rate_hz`, `packets`,
     `samples`, `missing_samples` (samples the headband itself dropped), `session` and
     `arrival_delay_ms` (`median`, `p95`, `max`: how much later than the fastest packet the others
     arrived over the last seconds).
2. If no bridge answers, start it in the background with `uv run bridge run`. It opens a pairing
   page with a QR code in the browser. Ask the user to tap "Scan QR code" on the Live screen's
   "Stream to PC" card and scan the code; data flows once the headband is connected too.
   `/h1/state` says whether it is (see [What arrives](#what-arrives)).
3. `link` in the status says what to tell the user: `waiting` means scan the code on the page
   (reload it for the current code); `not_responding` means the phone went quiet (Wi-Fi, app in the
   background) and continues by itself when it is back.
4. If a bridge is already running, `uv run bridge run` does not start a second one: it prints the
   pairing page's path and ends with exit code 1. Say what you want with `--if-running`:
   - `uv run bridge run --if-running open` opens the running bridge's pairing page again, for
     example when the user has closed it and needs the QR code. The bridge keeps running.
   - `uv run bridge run --if-running restart` replaces the running bridge. Use it only when the
     user asks for it or the bridge misbehaves: it ends the phone's link, so the user has to scan
     the QR code again.

Never print the raw stream into the conversation; it is hundreds of values per second. Use
`uv run bridge listen --seconds 3` for message rates, or compute and show summaries.

## How a program uses it

A program subscribes to the bridge once and then deals with four kinds of things. Choose by what
the program needs, not by what is easiest to compute yourself:

| Kind | What it is | Use it for | In Python |
| --- | --- | --- | --- |
| Stream | numbers at a regular rate, raw or computed, each sample with its headband time | anything that has a value at every moment: plotting, recording, filtering, steering by head tilt | `h1.window(name, seconds)` |
| State | one word for what the headband is doing | pausing while no samples flow, and telling the wearer what to do about it | `h1.state` |
| Event | something that happened at one moment, such as a look to the left | a trigger, handled like a key press | `h1.events()` |
| Command | something only the program can know, told to the processing service, such as "the wearer is asked to blink now" | calibrating what the service computes | `h1.prompt()` |

- **A program asks and never waits.** On each pass of its own loop it reads the state, collects the
  events and takes the latest window of the streams it needs, then draws or acts. Nothing blocks the
  program's loop, be it an application's timer, a display's refresh or a game's frame, and nothing
  calls back into it.
- **Prefer what is offered ready-made.** For looks to the left and right use the events, and for
  "is there muscle activity" or "is the headband worn" the broadband power, before building a
  detector on the raw EEG.
- **Subscribing** is what makes the bridge send to a program. `control.Headband` does it; a program
  in another language does it itself, see [The OSC protocol](references/osc-protocol.md).
- **The bridge's status** (`uv run bridge status`) is for you, before building and when nothing
  arrives: it says whether a phone is linked, what arrives at which rate and what was lost. A
  program rarely needs it; inside a program, `h1.state` says what to tell the wearer.

## What arrives

| Address | Values per sample | Rate | Unit and meaning |
| --- | --- | --- | --- |
| `/h1/eeg/raw` | left, right | 256 Hz | µV, unfiltered; see [EEG](references/signals.md#eeg) |
| `/h1/motion/orientation` | w, x, y, z | 50 Hz | orientation quaternion, device-to-world, scalar first; see [Head axes](references/signals.md#head-axes-and-orientation) |
| `/h1/motion/raw` | ax, ay, az, gx, gy, gz | 50 Hz | acceleration in m/s² (includes gravity), rotation in rad/s |
| `/h1/light/lux` | lux | 1 Hz | ambient light; `0.0` is darkness, NaN is a second without a reading |

The processing service adds one computed stream, which arrives and is read like the raw ones:

| Address | Values per sample | Rate | What it is | What it is for |
| --- | --- | --- | --- | --- |
| `/h1/eeg/broadband_power` | left, right | 10 Hz | power of each EEG channel from 1 to 45 Hz over the last second, in µV²; each value carries the time of the end of its second | a value that rises with muscle activity such as a jaw clench, and with strong eye movements and blinks; a check that the headband is worn and delivers a signal |

At rest it is the power of the EEG itself, together with the sensor's own noise; it does not fall
to zero. In Python: `h1.window("eeg/broadband_power", seconds=5)`.

`h1` is the first headband; more devices would follow as `h2` and so on. Each message is one
headband packet (a few samples). The streams, their rates and their numbers of values are fixed as
in the table; nothing announces them at run time.

`/h1/state` carries one word for what the headband is doing, sent on every change and every 5 s.
Tell the user what to do from it:

| State | Meaning | Tell the user |
| --- | --- | --- |
| `disconnected` | the phone has no headband connected | connect the headband in the app |
| `connecting` | connecting, or connected and starting | nothing; wait a few seconds |
| `streaming` | the headband measures and samples flow | nothing |
| `start_failed` | connected, but the headband did not start measuring | disconnect and reconnect the headband in the app |

Without a phone linked to the bridge, nothing arrives, not even a state; `uv run bridge status`
then says why.

## Writing and running a program

This folder is a uv project with the `openlinc` package in it. Write the user's program next to
`README.md`, as one file or as a folder of its own. Avoid changes in the `openlinc` folder unless
they are necessary: it is the package as released, and a change there is lost with the next
update.

- **Run it** with `uv run my_program.py`, while the bridge runs.
- **Already there:** numpy and scipy, enough for filtering and spectra.
- **Add a library** with `uv add <package>`. Nothing for windows, drawing or controlling other
  software comes with the folder, so add what the program needs.

Suggestions by kind of program, where the user names nothing else:

| Kind of program | Suggestion |
| --- | --- |
| live display or dashboard: plots, numbers, a 3D view | a page in the browser, with a small Python program that reads the headband and passes the data on to the page; uPlot for line plots, three.js for 3D. See [live-data-visualization.md](references/live-data-visualization.md) |
| game or full-screen visuals | pygame |
| pressing keys or moving the mouse in other software | pynput, which presses keys and moves the mouse as the user would |
| output for a terminal only | none |

Choose a library whose loop the program owns or that has a timer, and read the headband from
there: once per frame in pygame, and in the loop that sends to the page for a display in the
browser.

A program in another language or tool is as welcome; see [osc-protocol.md](references/osc-protocol.md).

### One command starts everything

Let the program make sure a bridge runs when it starts, so that the user starts one thing and not
a bridge first. In this order:

1. **Ask for the bridge's status** (`bridge status`).
2. **No bridge answers:** start `bridge run` as a child process and wait until it answers. It
   opens its pairing page with the QR code itself.
3. **A bridge answers, but `link` is not `linked`:** run `bridge run --if-running open`, which
   opens the running bridge's pairing page again.
4. **Wait until `link` is `linked`** before the program opens its own window or page; ask about
   once a second. The user then sees the QR code first and the program afterwards, and nothing
   covers the pairing page. Say in the terminal what the program waits for. With a phone already
   linked there is nothing to wait for.
5. **At the end, end only a bridge the program started.** A bridge that was running already may
   serve another program, or be the one you started while building; leave it running.

From Python, run the bridge's command line with the Python the program itself runs with, for
example `subprocess.run([sys.executable, "-c", "from openlinc.bridge.cli import main; main()",
"status"], capture_output=True)`; it prints the status as JSON and ends with an exit code other
than 0 when no bridge answers.

A program in another language follows the same steps with `uv run bridge status` and
`uv run bridge run`, or asks for the status over OSC, see
[The OSC protocol](references/osc-protocol.md).

## Reading the stream in Python

`control.Headband` subscribes to the bridge and keeps the latest seconds of each of one device's
streams, each sample with its headband time. It receives in a background thread, so the
program's own loop never waits: it asks for the latest data whenever it wants.

```python
import time

from openlinc import control

with control.Headband("h1", seconds=10) as h1:
    started = time.monotonic()
    while True:
        times, eeg = h1.window("eeg/raw", seconds=4)  # times in µs; eeg shape (samples, 2)
        rate = h1.rate_hz("eeg/raw")  # samples per second
        if h1.state != "streaming" and time.monotonic() - started > 1:
            print(f"headband: {h1.state}")  # None: no phone, see `uv run bridge status`
        time.sleep(0.25)
```

- Stream names are the address without the device: `eeg/raw`, `motion/orientation`, `motion/raw`,
  `light/lux`.
- `window()` covers the latest `seconds` of headband time, oldest first. Times are continuous
  across the clock's wrap and line the device's streams up directly. A window never mixes two
  sessions: a new session starts every stream afresh. Missing samples show as jumps in the times,
  and a packet that arrives out of order is dropped, so a window is always in time order.
- **Start:** the bridge sends a new subscriber each device's latest state at once, and windows
  fill with the first packets. Until something arrives, a window is empty, `state` is None and
  `receiving` is false; that is the normal first moment, not an error.
- **Receiving:** `h1.receiving` is true while a message of this device arrived within the last
  second. It stays true while samples flow. Without samples it is false nearly all the time,
  because then only the state arrives, every 5 s; `state` says why no samples flow.
- **State:** `state` is the phone's latest `/h1/state` word, or None when none arrived within the
  last 11 s (no phone linked). Windows fill only while it is `streaming`.
- **Session:** `h1.session` is the session the streams are in, or None before any sample. It
  changes when the headband starts measuring anew or connects to the phone again after a full
  disconnect; then every stream has started afresh.
- **Events:** `h1.events()` returns what happened since the last call, oldest first, each exactly
  once, and never waits. Call it on every pass of the loop, as an application collects its key presses.
  Each event has a `kind`, for example `gaze/looking_left`, and a `time_us` on the same clock as
  the windows. See [Looks, blinks and clenches](references/actions.md).
- Without `with`, call `h1.start()` and `h1.stop()`, for example in a notebook.

## Showing signals

Before a program shows the EEG or the light, in a plot or in any other view, read
[live-data-visualization.md](references/live-data-visualization.md). Raw EEG needs a high-pass and
a mains notch to be readable, and light needs a logarithmic scale; the file says how and why.

## Real-time rules

- Filter forward with saved state (`scipy.signal.sosfilt` with `zi`), never over the whole
  recording at once (`filtfilt`): live data has no future samples.
- Keep heavy work out of the receiving thread; `control.Headband` already receives in its own.
- Expect packets to arrive a few tens of milliseconds apart and late at times; time samples by
  their headband timestamps as described in [Timing](references/signals.md#timing), never by arrival.
