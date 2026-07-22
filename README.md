# Neuro-Inspired System Engineering

A hand-gesture recognition pipeline (webcam → MediaPipe/cvzone landmarks → classifier) feeding two downstream applications: an assistive fingerspelling-to-speech tool, and a Unity rhythm game controlled by hand movements.

## Overview

1. **Fingerspelling → speech** — `client.py` is a Tkinter GUI that reads the webcam, detects hand landmarks with `cvzone`'s `HandDetector` (built on MediaPipe), computes 15 pairwise distances between key landmarks (thumb–index, palm–pinky, etc.), and classifies the resulting feature vector into a character using a pre-trained decision tree (`models/dt_classifier.pickle`). Recognized characters are assembled into a word, which is sent over UDP to `server.py`. The server can pass that text to `speech_synthesis_micAzure.py`, which synthesizes and plays it aloud via Azure Cognitive Services TTS — effectively an AAC (assistive communication) demo.
2. **Unity hand-controlled rhythm game** — the same hand-landmark stream is sent over UDP to a Unity project; `HandTracking.cs` reads the incoming landmarks, positions an in-scene hand, and uses thumb-to-fingertip distances to trigger different chords (via FMOD audio) in a "Let It Be" rhythm game / free-play mode — likely built for gamified hand-motor training rather than as production music software.
3. **`Arduino_UDP_server/`** — an additional UDP-connected hardware component (see note below — contents weren't accessible in this pass).

## Repository layout

| Path | Role |
|---|---|
| `client.py` | Webcam capture, hand-landmark feature extraction, classification, UDP sending, optional training-data collection (writes to `data2/`) |
| `server.py` | UDP listener; optionally forwards recognized text to speech synthesis |
| `speech_synthesis_micAzure.py` | Azure Cognitive Services Speech SDK wrapper (SSML generation + synthesis) |
| `mediapipe.py`, `run_mediapipe.ipynb` | MediaPipe-based hand-tracking utilities/notebook |
| `train_classifier.ipynb` | Trains the decision tree classifier used by `client.py` |
| `test.py` | Test/utility script |
| `HandTracking.cs` | Unity script consuming the UDP landmark stream, driving the in-game hand and rhythm-game chord playback |
| `Arduino_UDP_server/` | Arduino sketch(es) for the UDP-connected hardware component |
| `data/`, `data2/` | Collected hand-landmark training data |
| `models/` | Trained classifier(s), e.g. `dt_classifier.pickle` |

> GitHub's automated-browsing block prevented listing the exact contents of `Arduino_UDP_server/`, `data/`, `data2/`, `mediaPipe/`, and `models/` — the descriptions above are inferred from what's referenced in the Python/C# source rather than confirmed directly. Worth double-checking and adjusting if anything's off.

## How the fingerspelling → speech pipeline works

1. `client.py` captures the webcam feed and detects a hand with `cvzone.HandTrackingModule.HandDetector`.
2. It computes 15 normalized Euclidean distances between key landmark pairs and classifies the feature vector with the pre-trained decision tree.
3. Recognized characters accumulate into a word; a specific hand shape ("Empty") signals the word is complete.
4. The completed word is sent via UDP to `server.py`.
5. `server.py` can forward that text to `speech_synthesis_micAzure.py` for Azure TTS playback.

## Setup

### Python dependencies

Not pinned in a `requirements.txt` in this repo. Based on the imports in source, you'll need at least:

```
opencv-python
cvzone
numpy
scikit-learn
pillow
azure-cognitiveservices-speech
```

(`tkinter` ships with most Python installs; on Linux you may need to install `python3-tk` separately.)

### Running the spelling → speech demo

1. Start the server first (it will prompt for your machine's local IP — see the `ifconfig` hint at the top of `server.py`):
   ```bash
   python server.py
   ```
2. Start the client, enter the server's IP and port in the GUI, then press **Start**:
   ```bash
   python client.py
   ```

### Unity rhythm game

Open the Unity project containing `HandTracking.cs` and point its `UDPReceive` component at the same port the Python side sends to. Requires the FMOD Unity integration for chord audio.

## Security note

`speech_synthesis_micAzure.py` currently hardcodes a live Azure Speech subscription key and region as a fallback value instead of reading them from the `SPEECH_KEY`/`SPEECH_REGION` environment variables the comment above it references. **That key is exposed in this public repo right now** — rotate it in the Azure Portal and switch the code to read from the environment. Since it's already committed, deleting the line won't remove it from git history; treat the key as compromised regardless.

## License

Licensed under the Apache License 2.0 — see [`LICENSE`](LICENSE).
