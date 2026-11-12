## Appendix A. Reference soundsets

These are the exact sounds from the browser prototype, kept as the reference so mill's real sounds match them. The same data is in `soundsets.json` (machine-readable, one entry per event), which the Go build reads.

### A.1 Building blocks

Every sound is one or more **layers** started together (or after an offset, `+N ms`). There are two kinds:

| Layer | Definition |
|---|---|
| **tone** | An oscillator (`sine`, `triangle`, `square` or `sawtooth`) at a frequency, optionally sliding exponentially to a second frequency over the layer's length. Gain envelope: exponential rise from silence to the layer's gain in 3 ms, then exponential fall to silence (−80 dB) at the end of the layer. |
| **noise** | White noise whose amplitude fades as `(1 − t/length)²`, through one biquad filter (`bandpass` or `lowpass`) at a frequency and Q, scaled by the layer's gain. |

**Mixing.** All layers go through one master gain equal to `sound.volume` (default 0.6). Sounds never duck or cut each other off.

**Rules carried over from the prototype**

- `tick` plays at most once every 40 ms, however fast keys repeat.
- `boot_ok`, `boot_warn` and `boot_fail` play once per boot splash row, 70 ms apart, so `boot_ok` stays under 15 ms and quiet. They're in the `interface` group.
- Each event belongs to a group (`interface`, `editing`, `runs`), and the group's setting must be on. Previews on the settings page ignore the group switches.
- A set with no recipe for an event plays nothing for it (`bell` is mostly silent on purpose).
- The startup voice is separate (§18.11). The prototype's placeholder uses the system text-to-speech saying "mill" at pitch 0.55 and rate 0.82, so the final recording should sit in that lower, slower register.

**Making the real assets.** `go generate` renders every recipe to 48 kHz, 16-bit mono WAV with a fixed random seed for the noise, so sounds are identical on every machine and build. The files are embedded in the binary. Custom soundsets can still use recorded files instead (§18.11).

### A.2 `millrace`

Water and wood: soft clicks, a sluice opening, drops, a turning wheel. The default.

| Event | Group | Layers |
|---|---|---|
| `boot_ok` | interface | noise · bandpass 3000 Hz Q 8, 10 ms, gain 0.18 |
| `boot_warn` | interface | triangle 330 Hz, 60 ms, gain 0.10 |
| `boot_fail` | interface | sine 110→70 Hz, 260 ms, gain 0.30<br>noise · bandpass 300 Hz Q 1, 60 ms, gain 0.30 |
| `tick` | interface | noise · bandpass 2600 Hz Q 5, 14 ms, gain 0.35 |
| `open` | interface | noise · bandpass 900 Hz Q 0.7, 70 ms, gain 0.30 |
| `back` | interface | noise · bandpass 520 Hz Q 0.7, 50 ms, gain 0.25 |
| `toggle` | interface | noise · bandpass 3200 Hz Q 6, 12 ms, gain 0.30<br>+35 ms: noise · bandpass 2200 Hz Q 6, 12 ms, gain 0.25 |
| `error` | interface | triangle 170→110 Hz, 140 ms, gain 0.22 |
| `place` | editing | noise · bandpass 480 Hz Q 2, 40 ms, gain 0.50 |
| `connect` | editing | triangle 196→294 Hz, 140 ms, gain 0.20 |
| `disconnect` | editing | triangle 294→180 Hz, 120 ms, gain 0.18 |
| `save` | editing | sine 660 Hz, 160 ms, gain 0.10<br>+60 ms: sine 990 Hz, 220 ms, gain 0.07 |
| `lint_ok` | editing | sine 880 Hz, 120 ms, gain 0.09 |
| `lint_fail` | editing | triangle 210 Hz, 90 ms, gain 0.20<br>+100 ms: triangle 175 Hz, 110 ms, gain 0.20 |
| `run_start` | runs | noise · lowpass 650 Hz Q 0.5, 220 ms, gain 0.25 |
| `node_done` | runs | sine 1150→420 Hz, 100 ms, gain 0.16 |
| `retry` | runs | noise · bandpass 820 Hz Q 3, 25 ms, gain 0.40<br>+80 ms: noise · bandpass 700 Hz Q 3, 25 ms, gain 0.35 |
| `run_done` | runs | noise · lowpass 480 Hz Q 0.6, 280 ms, gain 0.18<br>+140 ms: sine 523 Hz, 500 ms, gain 0.10<br>+260 ms: sine 784 Hz, 650 ms, gain 0.08 |
| `run_failed` | runs | sine 92→52 Hz, 380 ms, gain 0.35<br>noise · bandpass 300 Hz Q 1, 90 ms, gain 0.35 |

### A.3 `foundry`

Industrial: switches, conveyor clunks, a press, a furnace roar for failures.

| Event | Group | Layers |
|---|---|---|
| `boot_ok` | interface | square 2200 Hz, 6 ms, gain 0.04 |
| `boot_warn` | interface | square 600 Hz, 40 ms, gain 0.05 |
| `boot_fail` | interface | noise · lowpass 180 Hz Q 0.4, 250 ms, gain 0.30<br>sawtooth 80 Hz, 300 ms, gain 0.08 |
| `tick` | interface | square 1900 Hz, 8 ms, gain 0.05 |
| `open` | interface | noise · bandpass 1500 Hz Q 1.5, 60 ms, gain 0.25<br>square 140 Hz, 70 ms, gain 0.06 |
| `back` | interface | square 120 Hz, 60 ms, gain 0.06 |
| `toggle` | interface | square 2400 Hz, 10 ms, gain 0.05<br>+30 ms: square 1600 Hz, 12 ms, gain 0.05 |
| `error` | interface | sawtooth 110 Hz, 180 ms, gain 0.09 |
| `place` | editing | noise · bandpass 300 Hz Q 1.2, 50 ms, gain 0.50<br>square 80 Hz, 80 ms, gain 0.08 |
| `connect` | editing | noise · bandpass 1800 Hz Q 8, 80 ms, gain 0.25 |
| `disconnect` | editing | noise · bandpass 1200 Hz Q 8, 60 ms, gain 0.20 |
| `save` | editing | sine 1200 Hz, 250 ms, gain 0.07<br>sine 2400 Hz, 200 ms, gain 0.04 |
| `lint_ok` | editing | sine 1400 Hz, 150 ms, gain 0.06 |
| `lint_fail` | editing | sawtooth 90 Hz, 200 ms, gain 0.08 |
| `run_start` | runs | noise · lowpass 200 Hz Q 0.5, 300 ms, gain 0.30 |
| `node_done` | runs | sine 1300 Hz, 180 ms, gain 0.07<br>sine 2600 Hz, 120 ms, gain 0.04 |
| `retry` | runs | square 160 Hz, 50 ms, gain 0.07<br>+100 ms: square 160 Hz, 50 ms, gain 0.07 |
| `run_done` | runs | noise · bandpass 250 Hz Q 1, 120 ms, gain 0.40<br>+120 ms: sine 988 Hz, 700 ms, gain 0.08 |
| `run_failed` | runs | noise · lowpass 180 Hz Q 0.4, 500 ms, gain 0.40<br>sawtooth 65 Hz, 500 ms, gain 0.08 |

### A.4 `chiptune`

8-bit square-wave blips and arpeggios.

| Event | Group | Layers |
|---|---|---|
| `boot_ok` | interface | square 2093 Hz, 12 ms, gain 0.03 |
| `boot_warn` | interface | square 440 Hz, 50 ms, gain 0.05 |
| `boot_fail` | interface | square 262 Hz, 100 ms, gain 0.06<br>+90 ms: square 196 Hz, 140 ms, gain 0.06 |
| `tick` | interface | square 1320 Hz, 20 ms, gain 0.04 |
| `open` | interface | square 660 Hz, 40 ms, gain 0.05<br>+40 ms: square 990 Hz, 50 ms, gain 0.05 |
| `back` | interface | square 990 Hz, 40 ms, gain 0.05<br>+40 ms: square 660 Hz, 50 ms, gain 0.05 |
| `toggle` | interface | square 1760 Hz, 30 ms, gain 0.04 |
| `error` | interface | square 220→110 Hz, 140 ms, gain 0.06 |
| `place` | editing | square 523 Hz, 60 ms, gain 0.05 |
| `connect` | editing | square 523 Hz, 50 ms, gain 0.05<br>+40 ms: square 659 Hz, 50 ms, gain 0.05<br>+80 ms: square 784 Hz, 50 ms, gain 0.05 |
| `disconnect` | editing | square 784 Hz, 50 ms, gain 0.05<br>+40 ms: square 659 Hz, 50 ms, gain 0.05<br>+80 ms: square 523 Hz, 50 ms, gain 0.05 |
| `save` | editing | square 784 Hz, 70 ms, gain 0.05<br>+60 ms: square 1047 Hz, 70 ms, gain 0.05 |
| `lint_ok` | editing | square 1047 Hz, 80 ms, gain 0.04 |
| `lint_fail` | editing | square 330 Hz, 80 ms, gain 0.05<br>+80 ms: square 262 Hz, 80 ms, gain 0.05 |
| `run_start` | runs | square 392 Hz, 70 ms, gain 0.05<br>+70 ms: square 523 Hz, 70 ms, gain 0.05 |
| `node_done` | runs | square 1568 Hz, 60 ms, gain 0.04 |
| `retry` | runs | square 440 Hz, 50 ms, gain 0.05<br>+100 ms: square 440 Hz, 50 ms, gain 0.05 |
| `run_done` | runs | square 523 Hz, 90 ms, gain 0.05<br>+80 ms: square 659 Hz, 90 ms, gain 0.05<br>+160 ms: square 784 Hz, 90 ms, gain 0.05<br>+240 ms: square 1047 Hz, 90 ms, gain 0.05 |
| `run_failed` | runs | square 392 Hz, 120 ms, gain 0.06<br>+100 ms: square 330 Hz, 120 ms, gain 0.06<br>+200 ms: square 262 Hz, 120 ms, gain 0.06<br>+300 ms: square 196 Hz, 120 ms, gain 0.06 |

### A.5 `bell`

Only a bell, and only for errors and finished or failed runs.

| Event | Group | Layers |
|---|---|---|
| `boot_ok` | interface | silent |
| `boot_warn` | interface | silent |
| `boot_fail` | interface | sine 440 Hz, 450 ms, gain 0.12 |
| `tick` | interface | silent |
| `open` | interface | silent |
| `back` | interface | silent |
| `toggle` | interface | silent |
| `error` | interface | sine 660 Hz, 200 ms, gain 0.08 |
| `place` | editing | silent |
| `connect` | editing | silent |
| `disconnect` | editing | silent |
| `save` | editing | silent |
| `lint_ok` | editing | silent |
| `lint_fail` | editing | silent |
| `run_start` | runs | silent |
| `node_done` | runs | silent |
| `retry` | runs | silent |
| `run_done` | runs | sine 880 Hz, 700 ms, gain 0.12 |
| `run_failed` | runs | sine 440 Hz, 450 ms, gain 0.12<br>+220 ms: sine 415 Hz, 550 ms, gain 0.10 |

### A.6 `off`

Plays nothing.

---

[← spec index](index.md)
