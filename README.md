# Gesture-Based Access Control System

A three-layer hardware-integrated access control system that authenticates users via hand gesture recognition. Built on Raspberry Pi 400, interfacing with NI myRIO (IR proximity sensing) and Delta PLC (door actuation) over GPIO and Modbus TCP.

## System Architecture

```
┌─────────────┐    GPIO/HIGH    ┌──────────────────┐   Modbus TCP   ┌───────────┐
│   NI myRIO  │ ──────────────► │  Raspberry Pi 400 │ ─────────────► │ Delta PLC │
│  IR Sensor  │   (Pin 37)      │  MediaPipe + CV2  │  (Port 502)   │  M1 Coil  │
└─────────────┘                 └──────────────────┘                └───────────┘
                                        ▲
                                        │ USB
                                  ┌─────┴─────┐
                                  │  Webcam    │
                                  └───────────┘
                                        ▲
                                        │ HTTP
                                  ┌─────┴──────────┐
                                  │ Password Server │
                                  │ (192.168.1.6)   │
                                  └────────────────┘
```

## How It Works

1. **Trigger (myRIO → Pi):** An IR sensor on the myRIO detects a person at the door. It sends a HIGH signal to the Raspberry Pi via GPIO26 (physical pin 37) with a shared ground.

2. **Password Fetch (HTTP):** The Pi fetches the current gesture password from an HTTP server on the local network. The password is a 5-bit array representing which fingers should be raised: `[thumb, index, middle, ring, pinky]`.

3. **Gesture Recognition (MediaPipe):** The Pi opens a USB webcam and uses MediaPipe's HandLandmarker model to detect hand landmarks in real time. It converts the 21 landmark points into a 5-bit finger state array and compares it against the fetched password. The correct gesture must be held stable for 0.8 seconds within a 20-second window.

4. **Actuation (Pi → PLC):** On a successful match, the Pi sends a Modbus TCP command to a Delta PLC, pulsing coil M1 ON for 2 seconds then OFF — triggering the door mechanism.

## Hardware

| Component | Role | Connection |
|---|---|---|
| NI myRIO | IR proximity detection | GPIO26 (pin 37) + mutual GND |
| Raspberry Pi 400 | Central controller, CV processing | USB webcam, Ethernet |
| USB Webcam | Hand capture | USB to Pi |
| Delta PLC | Door actuation | Modbus TCP (192.168.1.5:502) |
| Password Server | Serves current gesture password | HTTP (192.168.1.6) |

## Network Configuration

All devices on the same `192.168.1.x` subnet:

- **Raspberry Pi 400:** Controller
- **Delta PLC:** `192.168.1.5` — Modbus TCP on port 502
- **Password Server:** `192.168.1.6` — HTTP

## Project Structure

```
gesture-access-control/
├── src/
│   ├── main.py              # Main loop — orchestrates all layers
│   ├── get_ir2.py           # GPIO trigger listener (myRIO → Pi)
│   ├── password_client.py   # HTTP client — fetches gesture password
│   ├── gesture_camera.py    # MediaPipe hand detection + comparison
│   └── plc_output.py        # Modbus TCP client — controls Delta PLC
├── requirements.txt
├── .gitignore
├── LICENSE
└── README.md
```

## Module Breakdown

### `main.py`
Infinite loop: wait for IR trigger → fetch password → run gesture camera → pulse PLC.

### `get_ir2.py`
Listens on GPIO26 using `gpiozero`. Blocks until the myRIO sends HIGH, debounces, then returns `"ON"`. Waits for signal to go LOW before returning to prevent repeated triggers.

### `password_client.py`
GETs the current password from `http://192.168.1.6/`. Parses a plain-text response like `[0,0,0,0,1]` into a validated 5-element integer list.

### `gesture_camera.py`
Opens a USB camera via V4L2. Runs MediaPipe HandLandmarker in VIDEO mode. Extracts finger states by comparing landmark Y-coordinates (tip vs. PIP joint) for each finger. Requires the correct gesture to be held stable for `0.8s` within a `20s` timeout window.

### `plc_output.py`
Connects to the Delta PLC at `192.168.1.5:502` via Modbus TCP. Writes coil M1 (address 2049) ON, sleeps for the pulse duration, then writes it OFF.

## Setup

### Prerequisites
- Raspberry Pi 400 (or any Pi with GPIO + USB)
- Python 3.11+
- USB webcam
- NI myRIO with IR sensor wired to GPIO26
- Delta PLC on Modbus TCP
- Local network with all devices on `192.168.1.x`

### Installation

```bash
git clone https://github.com/YOUR_USERNAME/gesture-access-control.git
cd gesture-access-control

python -m venv venv
source venv/bin/activate

pip install -r requirements.txt
```

### MediaPipe Model

Download the hand landmarker model and place it in `src/`:

```bash
cd src
wget https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/latest/hand_landmarker.task
```

### Run

```bash
cd src
python main.py
```

## Configuration

Key parameters are defined as constants at the top of each module:

| Parameter | File | Default | Description |
|---|---|---|---|
| `MYRIO_TRIGGER_PIN` | `get_ir2.py` | `26` | GPIO pin for myRIO trigger |
| `SERVER_URL` | `password_client.py` | `http://192.168.1.6/` | Password server address |
| `PLC_IP` | `plc_output.py` | `192.168.1.5` | Delta PLC IP |
| `PLC_PORT` | `plc_output.py` | `502` | Modbus TCP port |
| `M1_COIL_ADDRESS` | `plc_output.py` | `2049` | Delta PLC M1 coil address |
| `DEFAULT_ATTEMPT_TIMEOUT` | `gesture_camera.py` | `20.0` | Seconds before gesture timeout |
| `DEFAULT_STABLE_TIME` | `gesture_camera.py` | `0.8` | Seconds gesture must be held |
| `DEFAULT_CAMERA_INDEX` | `gesture_camera.py` | `0` | USB camera index |

## Finger Encoding

```
Index:    [thumb, index, middle, ring, pinky]
Example:  [0,     1,     1,      0,    0   ]  = Peace sign ✌️
          [1,     1,     1,      1,    1   ]  = Open hand 🖐️
          [0,     0,     0,      0,    1   ]  = Pinky only 🤙
```

## License

MIT
