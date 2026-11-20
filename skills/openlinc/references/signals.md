# The signals in detail

Part of the OpenLinc skill; [SKILL.md](../SKILL.md) is its start and lists the streams.

## Head axes and orientation

`/h1/motion/orientation` and `/h1/motion/raw` use the same device axes, and samples of the two
with the same timestamp come from the same reading. As worn:

| Axis | Points |
| --- | --- |
| `+X` | to the wearer's left |
| `+Y` | toward the crown |
| `+Z` | out of the forehead |

These directions are nominal: the forehead's slope and how the band sits tilt them by a few
degrees. When a program needs a neutral pose, calibrate with the user (for example "look straight
ahead for 3 seconds").

- **Quaternion** `(w, x, y, z)`: rotates device axes into world axes; world `+Z` is up. The
  identity means lying flat on the back, face up.
  - Tilt (which way is up) is reliable. It settles within a few seconds after the headband
    switches on, and fast movements skew it briefly.
  - Heading (turning about the vertical) has no fixed reference and drifts. Use it only for
    changes over a few seconds, never as a compass direction.
- **World up in head axes** answers "how is the head tilted" and ignores heading. For a normalized
  quaternion: `up = (2(xz − wy), 2(yz + wx), 1 − 2(x² + y²))`. Sitting upright gives about
  `(0, 1, 0)` and lying on the back `(0, 0, 1)`. Tilting toward the left shoulder makes `up.x`
  negative, toward the right shoulder positive; nodding forward makes `up.z` negative, leaning the
  head back positive.
- **Acceleration** includes gravity: at rest about 9.81 m/s² points along world up (sitting
  upright, `ay` reads about +9.81). It clips at about ±2 g (±19.6 m/s²) per axis, so knocks and
  fast nods can saturate it.
- **Rotation** is in rad/s about the device axes, right-handed.
- **Failed reads:** if the headband cannot read its motion sensor, it still sends a sample: all six
  `/h1/motion/raw` values exactly `0.0`, and the identity quaternion with the same timestamp. Treat such
  samples as missing.

## Timing

All four streams share one headband clock, so their timestamps line them up directly; EEG and
motion can still be about 10 ms apart. Never time samples by when packets arrive on the PC: Wi-Fi
delivers them in bursts and late at times.

- `timestamp_us` and `first_index` describe the packet's first sample. Both are unsigned 32-bit;
  `timestamp_us` wraps every 2³² µs (about 71.6 minutes), so compute differences modulo 2³².
- A new `session` starts when the headband starts measuring anew or connects to the phone again
  after a full disconnect. Compare session numbers only for equality. Indices usually restart at 0
  with a new session, but continue when the headband was still measuring. Timestamps keep
  counting, but the jump from the last timestamp of one session to the first of the next does not
  equal the real pause between them, so compare timestamps only within one session.
- **EEG:** sample `i` of a packet has index `first_index + i` and lies on a 256 Hz grid (steps of
  3906 or 3907 µs).
- **Motion (`motion/orientation`, `motion/raw`):** sample `i` lies about `i × 20 ms` after the
  first.
- **Light:** value `i` lies `i` seconds after the first.
- An index gap means samples the headband itself dropped (`missing_samples` in the status), not
  data lost on the way.

## EEG

| Channel | Electrode |
| --- | --- |
| left | left forehead, roughly Fp1 |
| right | right forehead, roughly Fp2 |

Both channels are measured against one reference electrode in the middle of the forehead, between
the two (roughly Fpz).

- **Polarity:** each value is reference minus electrode.
- **Unfiltered:** electrode offsets and slow drift are part of the values, and nothing removes mains
  hum, at 50 Hz or 60 Hz depending on the region. "Showing the EEG" in
  [live-data-visualization.md](live-data-visualization.md) says how to filter for a display.
- **Muscle activity** (jaw clenching, frowning, talking) shows as broadband noise across the whole
  band; `eeg/broadband_power` rises with it.
- The electrodes sit just above the eyes, so eye movements and blinks show strongly.

## Each message as it arrives

`openlinc.stream.packets()` yields each message as it arrives instead, unbuffered. Use it to record
raw messages or to see every packet's fields. The messages are these classes from
`openlinc.messages.phone_format`:

| Class | What it is | Fields |
| --- | --- | --- |
| `Packet` | one packet of a stream | see below |
| `DeviceState` | a device's state | `device`, `state` |
| `EventMessage` | an event | `address`, `seq` (counter of that address's events), `session`, `timestamp_us` |
| `Command` | a command another program sent | `address`, `value` |

A `Packet` has:

| Field | Meaning |
| --- | --- |
| `address` | stream, for example `/h1/eeg/raw` |
| `seq` | the phone's counter of data messages; a gap means messages lost on the way to the PC |
| `session` | session number; see [Timing](#timing) |
| `first_index` | headband sample index of the first sample |
| `timestamp_us` | headband time of the first sample in µs |
| `values` | the samples as a flat tuple, frame-major: `np.asarray(values).reshape(-1, channels)` |
