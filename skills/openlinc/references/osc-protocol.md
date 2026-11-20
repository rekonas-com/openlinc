# The OSC protocol

Part of the OpenLinc skill; [SKILL.md](../SKILL.md) is its start.

Anything that sends and receives OSC over UDP can use the bridge without Python: a game engine, a
creative coding tool, a program in any language. In Python, use `control.Headband` instead.

**Subscribe.** Send `/bridge/subscribe`, without arguments, to `127.0.0.1:47101` and repeat it
every 2 seconds. The bridge sends everything to the address and port the message came from, until
it has heard nothing from there for 5 seconds. So send and receive with the same UDP socket; a tool
that sends from one port and listens on another receives nothing. A new subscriber gets each
device's latest state at once.

**What arrives,** all on that socket; pick what you need by address:

| Address | Arguments | Meaning |
| --- | --- | --- |
| `/h1/eeg/raw`, `/h1/motion/orientation`, `/h1/motion/raw`, `/h1/light/lux`, `/h1/eeg/broadband_power` | int32 `seq`, int32 `session`, int32 `first_index`, int32 `timestamp_us`, then float32 values | one packet of a stream; the values are frame-major, all values of one sample next to each other, as many per sample as in [What arrives](../SKILL.md#what-arrives) |
| `/h1/state` | one string | the headband's state, see [What arrives](../SKILL.md#what-arrives) |
| `/h1/gaze/looking_left`, `/h1/gaze/looking_right`, `/h1/eyes/blink`, `/h1/jaw/clench`, `/h1/calibration/done`, `/h1/calibration/failed` | int32 `seq`, int32 `session`, int32 `timestamp_us` | an event; `seq` counts that address's events, so a gap means a lost one |

The integers are unsigned 32-bit numbers carried in the bits of an OSC int32: add 2³² to a
negative one. [Timing](signals.md#timing) says how `session`, `first_index` and `timestamp_us` give each
sample its time; `control.Headband` does that work in Python, here the program does it.

**What a program sends,** to the same port and from the socket it subscribed with, because the
bridge takes messages only from subscribers:

| Address | Arguments | Meaning |
| --- | --- | --- |
| `/h1/calibration/running` | one int32, 1 or 0 | 1 starts a calibration of the wearer's actions, 0 ends it; see [Looks, blinks and clenches](actions.md) |
| `/h1/calibration/prompt` | one string, an action's name such as `gaze/looking_left` | the wearer is asked to do this action now |
| `/h1/calibration/cancel` | one int32, 1 | the calibration that runs is given up; nothing is learned and no event answers |

**Status.** `/bridge/status`, without arguments, is answered with `/bridge/status` and one JSON
string with the keys listed in [Before building](../SKILL.md#before-building). It needs no subscription.
