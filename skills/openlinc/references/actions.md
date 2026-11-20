# Looks, blinks and clenches

Part of the OpenLinc skill; [SKILL.md](../SKILL.md) is its start.

The processing service recognises four things the wearer does with eyes or jaw and reports each as
an event. How these look in the EEG differs from wearer to wearer and with how the headband sits,
so the service has to learn them first, from examples the wearer gives in a calibration. Before a
calibration it has nothing to compare with and reports none of them.

A program therefore has two parts: it [calibrates](#calibrating) once per session, and afterwards
it [uses the events](#using-the-events).

## The events

| Event | Address | What happened | Typical use |
| --- | --- | --- | --- |
| `gaze/looking_left` | `/h1/gaze/looking_left` | the eyes jumped to the wearer's left, relative to the head | a control triggered with a look to the left |
| `gaze/looking_right` | `/h1/gaze/looking_right` | the same to the wearer's right | a control triggered with a look to the right |
| `eyes/blink` | `/h1/eyes/blink` | the wearer blinked | a control triggered with a deliberate blink |
| `jaw/clench` | `/h1/jaw/clench` | the wearer clenched the jaw | a control triggered with a short clench |

An event says that the action happened and when it started. It does not say how large it was or
where the eyes point afterwards. It arrives about 0.3 s after the action started.

## Calibrating

In a calibration the program asks the wearer for each action several times and tells the service
at which moment it asked. The service takes what the EEG showed right after each request as an
example of that action, and everything else during the calibration as an example of what is not
to be reported.

### The calls

Start only while `h1.state` is `streaming`: a calibration without samples teaches nothing and
fails.

1. `h1.start_calibration()`
2. For each request: show it, and at that moment call `h1.prompt(event_name)`, for example
   `h1.prompt("gaze/looking_left")`.
3. About two seconds after the last request: `h1.finish_calibration()`
4. The result arrives as an event, `calibration/done` or `calibration/failed`. It fails when a
   requested action showed fewer than three times in the EEG. What an earlier calibration taught
   stays in place until a new one succeeds.

A program with a window runs these steps from its own loop, so that it keeps drawing between
them. If it stops in the middle, it calls `h1.cancel_calibration()`: while a calibration runs,
the service reports no action to any program.

### What to ask of the wearer

These are what made calibrations work; adapt them to the program.

- **Ten or more requests of each action, in mixed order,** about two minutes in all. With fewer
  examples the actions are told apart less reliably.
- **Two seconds or more between two requests, at varying intervals.** The service takes what the
  EEG shows within about a second after a prompt as the example, so requests that follow more
  closely get mixed up. At a fixed rhythm the wearer acts before the request, and the example no
  longer follows the prompt.
- **A look is a target that jumps.** Let the point the wearer looks at jump to a side, stay about
  a second and jump back. Both jumps are looks: prompt the side it jumps to, and the opposite
  side when it jumps back. Make the jumps as large as the looks the program will use; small
  looks are recognised less reliably.
- **Show blink and clench requests where the wearer already looks,** for example by a change of
  colour with one short word. Reading a request elsewhere is an eye movement of its own.
- **Request blinks regularly, and ask the wearer to blink only then.** Nobody can hold back
  blinking for two minutes, and an unrequested blink is learned as something not to report.
- **Head still and eyes on the point otherwise,** because other movements show in the EEG too.
- **Request all four actions, also when the program uses only some.** An action that was never
  requested cannot be told apart and may be reported as one of the others.

### When to calibrate again

What was learned holds for one session and one position of the headband. Calibrate again when
`h1.session` has changed, when the wearer moved the headband or took it off, and after a failed
calibration.

## Using the events

After a successful calibration the program collects the events on each pass of its loop, like key
presses:

```python
for event in h1.events():
    if event.kind == "gaze/looking_left":
        ...
```

- **A look to a side and back gives two events,** one for each direction. Create the headband
  with `control.Headband("h1", look_pause_s=0.5)` to leave out looks that follow a look within
  that time, so that only the first one counts.
- **Expect a wrong or missing event now and then.** Make a single action cheap to undo, or ask
  for a deliberate pattern such as two looks to the same side.
- **In another language** the events and the calibration commands are OSC messages, see
  [The OSC protocol](osc-protocol.md).
