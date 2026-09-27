# DeskHPSDR — TecnoTalarn variant

> **This is a personal variant (fork) of [deskHPSDR by DL1BZ](https://github.com/dl1bz/deskhpsdr).**
> All credit for the original application goes to **DL1BZ** and the WDSP authors.
> This fork exists only to try a few user-experience improvements for the
> **Brick2 / Brick3** SDR series, and is not affiliated with the upstream project.

## Why this fork exists

We run deskHPSDR on macOS with a **Brick2** transceiver over OpenHPSDR protocol 2.
Two things bothered us in daily use:

1. The **TX audio monitor** could only ever be heard on the computer's own
   speakers — never on the Brick's own headphones, which is where you actually
   want it when operating.
2. Once we fixed that, the monitor turned out to be **very late** (~200 ms).
   That much delay makes monitoring your own voice on headphones unpleasant and
   nearly useless for checking how you sound.

This fork attacks both, plus one small MIDI annoyance.

---

## Changes in this fork

Three commits, one per feature, on top of upstream `470d605`:

### 1. TX monitor also goes to the SDR hardware audio path
`9c69303` — *TX monitor: also send monitor audio to the SDR hardware audio path*

The monitor was only written to the local host backend
(`audio_write_monitor()`), so it could only be heard on the Mac's speakers. The
hardware audio path (network **port 1028** / `AUDIO_FROM_HOST_PORT` for P2) is a
completely separate route that is fed only from the RX chain.

A new `tx_monitor_write()` writes the monitor sample to **both** paths: the local
backend as before, plus the hardware audio path when the new
**“HL2 TX Monitor to hardware”** option is enabled. All three monitor call sites
now go through it.

*Off by default. Persisted as `hl2_monitor_to_hardware`.*

### 2. Low-latency TX monitor
`4511dce` — *TX monitor: add low-latency mode tapping the raw microphone sample*

The normal monitor is fed from `tx_full_buffer()`, which works on **WDSP blocks of
up to 1024 samples** and runs the whole TX chain. That block and pipeline delay is
what makes the operator hear their own voice roughly 200 ms late.

An optional low-latency mode taps the microphone sample in `tx_add_mic_sample()`,
i.e. **once per incoming sample, before any DSP**. The monitor level still mirrors
the real TX path (monitor gain × the WDSP PanelGain, with 0 dB for DIGL/DIGU and
captured/voice-keyer TX), so you hear the same level that is transmitted.

Two more buffers had to come out of the monitor path for this to actually pay off:

* **`buffered_audio.c`** — `audio_write_internal()` padded the local ring with
  silence up to `RX_LAT_TARGET` (1536 samples, **~32 ms**) whenever it ran low.
  Right for RX continuity, pure added delay for the monitor. `audio_write_monitor()`
  now passes `is_monitor` and skips it.
* **`new_protocol.c`** — the port 1028 ring carries RX audio, and on the RX→TX
  transition it was **never drained** (the drain existed only for CW). Monitor
  samples were therefore queued behind up to **~85 ms** of RX backlog. The backlog
  is now dropped on the RX→TX transition when the low-latency monitor is active.

*Off by default. Persisted as `hl2_monitor_low_latency`.*

**Trade-off:** the monitor then carries the **unprocessed** microphone signal — no
equalizer, no filter, no ALC. That is the price of the latency.

### 3. MIDI: invert wheel direction
`e2a741e` — *MIDI: add option to invert wheel direction*

Some MIDI controllers report their modulation and pitch wheels with the opposite
polarity to what the operator expects. An **“Invert Wheel Direction”** checkbox in
the MIDI menu negates the wheel value before it is dispatched.

*Off by default. Persisted as `midiInvertWheels`.*

---

## Honest status of the latency work

We removed roughly **117 ms** of buffering on the deskHPSDR side:

| Stage | Before | After |
| --- | --- | --- |
| Microphone packet (P2, 64 samples) | 1.33 ms | 1.33 ms |
| Low-latency tap (pre-DSP) | — | ~0 ms |
| Port 1028 ring (RX backlog) | ~85 ms | 0 ms |
| Local Mac ring (silence fill) | ~32 ms | 0 ms |
| **Known total** | **~118 ms** | **~2.7 ms** |

**However:** even after all of this, the monitor **still sounds noticeably late on
the Brick's headphones**, and it is still annoying to listen to. The remaining
delay is therefore **not** in deskHPSDR — every buffer we could find on the host
side has been removed.

The remaining suspect is the **Brick gateware**, most likely its internal DAC FIFO.
That is outside this repository and needs the Brick developer's input.

We are publishing this fork mainly so that work is not lost and so others hitting
the same problem can see what has already been tried and ruled out.

---

## Building

The build procedure is unchanged from upstream. See `COMPILE.macOS` for macOS,
and the upstream repository for Linux and Windows instructions.

```sh
make -j8
```

On macOS, `make` picks up Homebrew FFTW and libwebsockets automatically.

---

## Credits and licence

* Original application: **deskHPSDR by DL1BZ** — https://github.com/dl1bz/deskhpsdr
* DSP library: **WDSP by Warren C. Pratt (NR0V)**
* Brick SDR series: **Anton**, for the technical documentation that made Brick
  support possible
* The upstream project was itself forked once from
  [DL1YCF's pihpsdr codebase](https://github.com/dl1ycf/pihpsdr) in October 2024.

The licence of the original project applies to this fork.
