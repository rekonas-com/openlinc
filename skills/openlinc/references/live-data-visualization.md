# Live data visualization in the browser

Part of the OpenLinc skill; [SKILL.md](../SKILL.md) is its start.

For a dashboard, a live plot or a 3D view, draw in a browser page and let a small Python program
bring the data to it. These are recommendations for where the user names nothing else; the look
of the page is the user's to decide.

## How the data gets to the page

A page in the browser cannot receive UDP, so it cannot subscribe to the bridge itself. One Python
program does both jobs: it reads the headband with `control.Headband`, and it serves the page and
a stream of the newest samples.

- **Prefer lightweight solutions such as Python's `http.server`,** unless the user asks for
  something else.
- **Send in batches,** each sample with its headband time. One message per sample is far too
  many.
- **Turn NaN into `null` before sending JSON.** A second without a light reading is NaN; Python
  writes it into JSON as `NaN`, and the browser then rejects the whole message.
- **A page that opens late** first gets the seconds the program has kept, then what is new.
- **A new session** (`h1.session` changed) starts every stream afresh; tell the page to clear.
- **Listen on `127.0.0.1` only, and refuse requests whose `Host` header is not the program's own
  address.** Otherwise a page of another website can read the wearer's data.
- **Start the bridge from the program and open the page once the phone is linked,** see "One
  command starts everything" in [SKILL.md](../SKILL.md).

## Drawing

- **Libraries come from a CDN with a fixed version,** as a script or an import map in the page.
  That keeps the folder free of Node and of a build step. The page then needs the internet when
  it loads; copy the files next to the page when it has to work without.
- **Line plots: uPlot.** It is small and fast enough for several signals at 256 Hz, and it brings
  axes, scaling, a cursor and zoom, which take long to get right by hand. Draw on a plain canvas
  instead when the display needs a look of its own that works against a plotting library.
- **3D: three.js,** for example a model of the headband that turns with `motion/orientation`.
  Apply the quaternion to the model directly; the axes are in [signals.md](signals.md).
- **Draw once per frame** (`requestAnimationFrame`), whatever arrived since the last one. Do not
  draw on every message.
- **The time axis is headband time, not arrival.** Samples arrive in bursts. Let the right edge
  run with the browser's clock a little behind the newest sample, about 0.1 s, and jump only
  when it has fallen far behind. A plot that ends at the newest sample stutters.
- **Filter the EEG for the display** and **draw the light on a logarithmic scale,** see the two
  sections below.
- **Show the state.** When `state` is not `streaming`, say on the page what the wearer has to do;
  the table is in [SKILL.md](../SKILL.md).

## Showing the EEG

`eeg/raw` is unfiltered. Whenever a program shows the EEG, as a live plot or in any other view,
filter it for the display first:

- **A high-pass at 0.5 Hz** (Butterworth, second order). The electrodes' offset and slow drift
  are far larger than the signal; without the high-pass the trace is a line that wanders off.
- **A notch at the frequency of the mains,** 50 Hz or 60 Hz. Power lines and plugged-in devices
  add a hum at that frequency, which otherwise covers the trace.
- **Ask the user which of the two applies, and say why you ask.** Ask only roughly, for the
  region and never for a place or an address, for example: "Power lines add a hum to the EEG at
  50 Hz or 60 Hz, depending on the region, and I filter it out. Are you in a 60 Hz region (North
  and Central America, parts of South America, western Japan and a few others) or a 50 Hz region
  (Europe, Africa, most of Asia, Australia)?" Do not guess it from the language or the time zone.
- **Say on the display what was filtered,** for example "0.5 Hz high-pass, 50 Hz notch", so that
  nobody takes the trace for the raw signal.

Filter each channel on its own, forward and with saved state, see "Real-time rules" in
[SKILL.md](../SKILL.md). This is for
what is shown; a program that computes on the EEG chooses its filters for what it computes.

## Showing the light

`light/lux` spans from 0 in darkness over a few hundred indoors to tens of thousands in daylight.
In a graph, draw it on a logarithmic scale as log10(lux + 1), which keeps 0 on the scale, and
label the axis in lux (0, 10, 100, 1000 and so on). On a linear scale everything indoors lies
flat on the bottom as soon as daylight was in view once.
