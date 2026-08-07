# Wavelength

**A real-time focus monitor that reads your brain's electrical activity through an EEG headband and nudges you back on task the moment your concentration drops.**

![Python](https://img.shields.io/badge/Language-Python-blue)
![Signal Processing](https://img.shields.io/badge/Domain-EEG%20%2F%20BCI-9b59b6)
![Hardware](https://img.shields.io/badge/Hardware-Muse%202-yellow)

Motivation for the project: [*Building a Focus-Level Monitor to Tackle Short Attention Spans*](https://medium.com/@karinakainth30/building-a-focus-level-monitor-to-tackle-short-attention-spans-b9ff1b340bff) 

Wavelength is a focus tracker that reads directly from your brain. It uses a Muse 2 EEG headband to measure your real-time brainwave activity, and specifically tracks the balance between two frequency bands — **beta waves** (associated with active concentration) and **theta waves** (associated with drowsiness or mind-wandering). When that ratio dips out of a focused range for a few seconds straight, Wavelength sends a desktop notification telling you to get back on track.

## How it works

1. **Signal acquisition**
- Muse 2 streams raw EEG data (brain activity) over to BlueMuse (via Bluetooth) 
- Python connects data stream with `pylsl` and samples the data as it arrives
2. **Buffering**
- Incoming samples are pushed into a 5-second buffer (so the system always has a recent window of signals to analyze rather than operating on single noisy samples)
4. **Epoching**
- Buffer is broken into 1-second epochs (0.8s between consecutive epochs + 0.2s shift)
- Keeps analysis rate fast while still giving the Fast Fourier Transform (FFT) a full second of data to work with
5. **Frequency decomposition**
- Each epoch is passed through an FFT to produce a spectrum of signals
- Spectrum is split into the four classic EEG frequency bands: delta (<4 Hz), theta (4–8 Hz), alpha (8–12 Hz), and beta (12–30 Hz)
- A 55–65 Hz notch filter (a 4th-order Butterworth bandstop filter) is applied so there's no discontinuity between chunks
6. **The focus metric**
- Beta power is divided by theta power for each epoch, then averaged across all epochs currently in the buffer
- This beta/theta ratio is a metric used in real neurofeedback research, particularly in the context of ADHD
7. **Debounced alerting**
- Once the beta/theta ratio has stayed above the focus threshold continuously for more than 3 seconds, the system fires a notification alerting the user to stay on track

## The Key Beta/Theta Ratio Code

```python
beta_metric = smooth_band_powers[Band.Beta] / smooth_band_powers[Band.Theta]

if beta_metric > .2 and (datetime.now() - t).seconds > 3:
    toast("This is a friendly reminder to FOCUS", "Please stop getting distracted.")
```

Every constant here reflects a design decision made from real signal behaviour rather than picked arbitrarily:

- **`0.2` threshold** ~ determined empirically by observing the beta/theta ratio during genuinely focused vs. distracted states (determined by own brainwave values during testing) 
- **`3` second dwell time** ~ long enough to filter out small or abnormal fluctuations in the ratio but short enough that the notification still fires in real time

## Running & Installing

**Hardware:** Muse 2 Headband

1. Install [BlueMuse](https://github.com/kowalej/BlueMuse), pair your Muse 2, and start an LSL stream.
2. Install Python 3.11 and the required packages: `pylsl`, `numpy`, `matplotlib`, `win11toast`.
3. Run the monitor script (it will automatically detect the active EEG stream, start buffering, and begin printing your live beta/theta ratio to the console).
4. Keep working normally. When your focus dips for more than 3 seconds, you'll get a desktop reminder to get back on track.

## Licensing & Credits

The raw signal-processing layer — `utils.py`, containing the circular buffering, epoching, notch filtering, and FFT-based band-power extraction — is the unmodified official utility module from the open-source [`muse-lsl`](https://github.com/alexandrebarachant/muse-lsl) project (credited in-file to its original author, Cassani). The base structure of the acquisition loop (connecting to the LSL stream, initializing buffers, the acquire → compute → print cycle) also follows `muse-lsl`'s official `neurofeedback.py` example.

**The original contribution here is everything built on top of that foundation:** turning raw band powers into a validated focus metric, empirically tuning the focus threshold and debounce timing against real recorded sessions, and wiring it into a live, actionable desktop notification system via `win11toast` — turning a signal-processing example script into an actual usable focus tool.
