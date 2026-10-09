# Mine Nav Rover

A starter repository for the Mine Nav Rover project. This scaffold separates camera/OpenCV processing on the Raspberry Pi from low-level motor control on an ESP32 over USB serial.

> **Important:** This is a software starter, not a validated vehicle controller. Test with the wheels lifted, use a physical power cutoff, and verify motor-driver wiring and pin assignments before connecting motors. OpenCV output in this starter is only a camera preview; it does not provide autonomous navigation or obstacle avoidance.

## Architecture

```text
USB webcam
   │
   ▼
Raspberry Pi ── OpenCV camera processing
   │
   └── USB serial (115200 baud; newline-delimited commands)
              │
              ▼
            ESP32 ── motor driver ── left/right motors
```

The Raspberry Pi sends one of these commands to the ESP32:

- `M,left,right` where each value is an integer from `-255` to `255`
- `STOP`

Examples: `M,120,120` forward, `M,-100,-100` reverse, `M,-80,80` turn in place, `STOP` stop motors.

The ESP32 firmware stops the motors if no valid command is received for 500 ms. Adjust this timeout only after testing.

## Repository layout

```text
.
├── README.md
├── requirements.txt
├── .gitignore
├── config.example.json
├── src/
│   ├── opencv_camera.py
│   └── esp_motor_control.py
└── firmware/
    └── esp32_motor_control.ino
```

## Requirements

- Raspberry Pi OS (Python 3)
- USB webcam
- ESP32 development board
- Dual H-bridge motor driver compatible with your motors
- Correctly rated motor power supply; do not power motors from the ESP32 5 V/3.3 V pin

## Setup on Raspberry Pi

```bash
sudo apt update
sudo apt install -y python3-opencv python3-serial
```

Or use pip in a virtual environment:

```bash
python3 -m venv .venv --system-site-packages
source .venv/bin/activate
pip install -r requirements.txt
```

Find the ESP32 serial port:

```bash
ls /dev/ttyUSB* /dev/ttyACM*
```

If permission is denied, add your user to the appropriate serial-access group (often `dialout`) and log out/in again.

## Run the camera preview

```bash
python3 src/opencv_camera.py --camera 0
```

Press `q` or `Esc` to exit. This program only displays the webcam feed and basic FPS; add and validate perception algorithms separately.

## Run motor teleoperation

First flash `firmware/esp32_motor_control.ino` to the ESP32 using Arduino IDE. Review and change the GPIO pin assignments to match your motor driver.

Then, with wheels lifted and a safe power cutoff available:

```bash
python3 src/esp_motor_control.py --port /dev/ttyUSB0
```

Use `w` forward, `s` reverse, `a` turn left, `d` turn right, `x` stop, and `q` quit. This terminal interface uses `curses`; run it in a real terminal. On quit, it sends `STOP`.

Optional parameters:

```bash
python3 src/esp_motor_control.py --port /dev/ttyACM0 --baud 115200 --speed 100
```

Speed is clamped to 0–255. Start at a low speed.

## Serial protocol

Commands are ASCII text terminated by `\n`. The Raspberry Pi Python script sends `M,left,right\n` or `STOP\n`. The ESP32 responds with `OK` for accepted commands.

## Safety and integration notes

- Confirm motor driver logic, enable pins, PWM frequency, current ratings, and GPIO compatibility against your exact hardware.
- Never connect motors directly to ESP32 GPIO pins.
- Keep motor power separate from logic power as required by the driver, and establish a common ground where the driver design requires it.
- Test direction mapping with the rover raised off the ground.
- Add encoder feedback, watchdogs, emergency-stop circuitry, and obstacle-stop logic before unattended operation.
- Do not treat a webcam preview or a basic OpenCV pipeline as a safety-rated navigation system.
- If using a Vicharak Shrike FPGA board for motor/encoder control instead of ESP32, do not use this ESP32 firmware unchanged; define a separate interface for the board you actually use.
