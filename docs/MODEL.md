# Burst classifier: model card


## What it does
Every RF burst the detector finds is described by 8 measured features (below). A small
neural network decides which of 5 signal classes the burst belongs to. It runs on the CPU
inside the app (ONNX Runtime), takes microseconds per burst, and needs no internet.

## Data
- **Source:** a C# dataset generator renders synthetic transmissions into 60 ms captures at
  2.4 MS/s with white noise, then runs the app's own spectrogram + burst detector on them,
  so the network learns from the same measurements it sees live.
- **Training set:** seed 1, 35603 bursts (30263 train / 5340 validation for early stopping).
- **Test set:** seed 2, 12333 bursts, never used for training.
- **Classes and generators:**
  - Noise / spur: unmodulated carriers (spurs), sub-100 us wideband clicks, noise peaks.
  - LoRa 500 kHz: SF6-SF9 chirp packets, 8-30 data symbols (the waveform family of 900 MHz long-range drone control links).
  - LoRa 125-250 kHz: SF7-SF10 packets, 10-40 symbols (LoRaWAN / Meshtastic style IoT).
  - FSK data: 32-128 kbit/s 2-FSK, modulation index 0.8-2.0, 1-15 ms packets (telemetry-radio style).
  - Narrowband: OOK pulse trains and 2.4-10 kbit/s narrow FSK (sensors, remotes).
  - SNR 6-30 dB in the signal's bandwidth; frequency offset up to +-600 kHz.

## Features
1. log10 duration (us)
2. log10 bandwidth (Hz)
3. instant width / bandwidth
4. chirp score
5. upward step ratio
6. peak spread / bandwidth
7. mean SNR / 20 dB
8. log10 frames

## Model
`8 -> 32 -> 32 -> 5` fully connected network, ReLU, softmax; 1509 parameters.
Feature standardisation is part of the exported graph. Cross-entropy with class weights,
Adam (lr 3e-3, weight decay 1e-4), batch 256, early stopping on validation loss.

## Results on the held-out test set
| | Accuracy | Macro F1 |
|---|---|---|
| **Neural network** | **0.964** | **0.965** |
| Rule-based baseline | 0.661 | 0.420 |

### Per class (neural network)
| Class | Precision | Recall | F1 | Test bursts |
|---|---|---|---|---|
| Noise / spur | 0.875 | 0.966 | 0.918 | 1970 |
| LoRa 500 kHz (drone control link class) | 0.999 | 0.987 | 0.993 | 1016 |
| LoRa 125-250 kHz (IoT) | 0.991 | 0.986 | 0.988 | 1074 |
| FSK data link (telemetry class) | 0.928 | 0.986 | 0.956 | 1090 |
| Narrowband (sensor / remote) | 0.989 | 0.954 | 0.971 | 7183 |

### Confusion matrix (neural network)
| true \ predicted | Noise | LoRa-500 | LoRa-IoT | FSK data | Narrowband |
|---|---|---|---|---|---|
| **Noise** | 1904 | 1 | 0 | 11 | 54 |
| **LoRa-500** | 10 | 1003 | 0 | 2 | 1 |
| **LoRa-IoT** | 4 | 0 | 1059 | 0 | 11 |
| **FSK data** | 8 | 0 | 0 | 1075 | 7 |
| **Narrowband** | 250 | 0 | 10 | 71 | 6852 |

## Scope
- A class is a waveform family, not a product: a LoRa 500 kHz burst can come from a drone
  control link or from other equipment that uses the same modulation.
