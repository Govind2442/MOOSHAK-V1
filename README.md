# Mine Nav Rover
### GPS-Denied Autonomous Ground Rover for Mine-Environment Reconnaissance and Hazard Sensing

Mine Nav Rover is a mobile robotics project focused on rover navigation and environmental sensing in GPS-denied environments. The platform combines onboard vision, wheel-motion sensing, inertial heading/orientation data, and an experimental electronic-nose (e-nose) gas-sensor array. The software is organized to separate high-level processing on the Raspberry Pi from embedded sensor and motor-control functions.

> **Project status:** This repository is a development scaffold. Some components are under integration or evaluation, and planned sensors are identified separately below. Do not interpret the listed sensors as validated mine-detection capability.

## Project goals

- Explore rover navigation in environments where GPS may be unavailable or unreliable.
- Capture camera data for future computer-vision and perception pipelines.
- Estimate rover movement using wheel encoders and inertial/magnetic sensing.
- Evaluate an array of gas sensors for changes in volatile-gas signatures.
- Keep motor control, sensor acquisition, and high-level computing modular.
- Provide a path for future ranging sensors and improved autonomy.

## System overview

```text
                       ┌────────────────────────────┐
                       │       USB Webcam           │
                       └─────────────┬──────────────┘
                                     │ USB
                                     ▼
┌────────────────────────────────────────────────────────────────┐
│ Raspberry Pi                                                   │
│ OpenCV camera pipeline • high-level logic • logging/telemetry  │
└───────────────────────────┬────────────────────────────────────┘
                            │ USB serial / defined interface
              ┌─────────────┴─────────────────┐
              ▼                               ▼
┌──────────────────────────┐     ┌──────────────────────────────┐
│ Embedded sensor subsystem│     │ Motor-control subsystem      │
│ e-nose sensor array      │     │ Shrike FI (planned/current    │
│ e.g. Arduino Nano during │     │ integration target) or ESP32 │
│ bench testing            │     │ reference implementation     │
└──────────────────────────┘     └──────────────┬───────────────┘
                                                │
                                     Motor driver and motors
                                                │
                                  Wheel encoders / odometry data
```

The diagram is a logical overview, not a final wiring schematic. Exact interfaces depend on the controller configuration used on the rover. If Shrike FI is the deployed motor/encoder controller, the ESP32 example in this repository is only a reference and must not be assumed to control Shrike FI.

## Sensors and hardware

### Current / experimental sensing

| Component | Role | Notes |
|---|---|---|
| USB webcam (720p/1080p class) | Image capture for OpenCV and future visual perception | Camera index and resolution are configurable. Validate lighting, frame rate, and image quality on the rover. |
| Wheel encoders | Wheel rotation and odometry input | Useful for estimating displacement and supporting dead reckoning; errors accumulate with wheel slip and uneven terrain. |
| IMU | Angular-rate and acceleration measurements | Supports motion estimation and orientation filtering. Exact part number and interface should be recorded in the hardware BOM. |
| Magnetometer | Magnetic-field measurements for heading estimation | Susceptible to interference from motors, wiring, steel structures, and nearby magnetic materials; calibrate in the assembled rover. |
| E-nose gas-sensor array | Experimental detection of changes in gas/VOC response patterns | Bench setup uses five gas sensors; MQ-2 and MQ-3 have been used in testing. Record the exact models of the remaining sensors before claiming specific gas coverage. |
| Arduino Nano (e-nose bench prototype) | Reads the analog gas-sensor outputs during experiments | A practical way to sample multiple analog sensors. Confirm voltage levels and ADC range before connecting to another controller. |

**E-nose limitations:** MQ-series sensors are broad-response metal-oxide gas sensors, not selective chemical identifiers. Their readings vary with warm-up time, humidity, temperature, sensor age, and calibration. Perfume and hand sanitizer can be useful repeatable test stimuli for early bench checks, but a response to them does not establish detection of explosives, narcotics, or mines. Treat readings as experimental signals and validate with controlled procedures.

### Motor and computing hardware

| Component | Role | Notes |
|---|---|---|
| Raspberry Pi | Main computer for OpenCV, high-level logic, and data logging | Runs Python tools in this repository. |
| Vicharak Shrike FI FPGA board | Intended low-level motor/encoder and related sensor-control subsystem | Confirm the exact firmware, pin mapping, and communication protocol used by the project. |
| ESP32 | Optional/reference motor-control or wireless manual-override path | The included example firmware is for a generic dual H-bridge and is not a drop-in Shrike FI driver. |
| Dual motor driver and DC motors | Physical rover locomotion | Driver, motor voltage/current, and pin assignments must match the actual hardware. |

### Planned / future sensing

| Component | Intended use | Status / caveat |
|---|---|---|
| 2D LiDAR | Range scans for obstacle detection and mapping/SLAM experiments | **Integrated on the rover.** Software integration and validation status should be documented separately. |
| 64 GHz mmWave sensor | Investigate presence/motion or concealed-object-related sensing | Exploratory only. Capabilities depend on the exact sensor and signal-processing method; do not claim it can identify narcotics or reliably identify concealed packages without validated evidence. |

## Repository structure

```text
.
├── README.md
├── requirements.txt
├── config.example.json
├── .gitignore
├── src/
│   ├── opencv_camera.py
│   └── esp_motor_control.py
└── firmware/
    └── esp32_motor_control.ino
```

## Software stack

- **Python 3** — high-level scripts and serial communication
- **OpenCV** — camera capture and image-processing foundation
- **pySerial** — USB serial communication with embedded controllers
- **Arduino IDE / C++** — example ESP32 firmware
- **Raspberry Pi OS** — target high-level computing platform

ROS 2, Nav2, visual SLAM, LiDAR mapping, sensor fusion, and autonomous route planning may be added as separate integration milestones; they are not implemented by the starter scripts alone.

## Setup on Raspberry Pi

Recommended system packages:

```bash
sudo apt update
sudo apt install -y python3-opencv python3-serial
```

Alternatively, create a virtual environment:

```bash
python3 -m venv .venv --system-site-packages
source .venv/bin/activate
pip install -r requirements.txt
```

Find the USB serial port:

```bash
ls /dev/ttyUSB* /dev/ttyACM*
```

The device name may differ. Close other programs using the serial port before starting the control script.

## Run the OpenCV camera preview

```bash
python3 src/opencv_camera.py --camera 0
```

Press `q` or `Esc` to exit. The script captures frames, shows a live preview, and displays approximate FPS. It is a camera smoke test—not object detection, mine detection, obstacle avoidance, or autonomous navigation.

## Run the ESP32 motor-control example

1. Check the ESP32 and motor-driver pin map in `firmware/esp32_motor_control.ino`.
2. Flash the firmware using Arduino IDE.
3. Lift the rover so the wheels cannot touch the ground.
4. Keep a physical motor-power cutoff accessible.
5. Start at low speed:

```bash
python3 src/esp_motor_control.py --port /dev/ttyUSB0 --speed 80
```

Keyboard commands:

| Key | Command |
|---|---|
| `W` | Forward |
| `S` | Reverse |
| `A` | Turn left in place |
| `D` | Turn right in place |
| `X` or Space | Stop |
| `Q` | Quit and send stop |

Change `/dev/ttyUSB0` to the correct port for your device. On Windows, a port such as `COM5` may be used.

## Serial protocol used by the ESP32 example

Commands are newline-terminated ASCII:

- `M,left,right` — wheel commands from `-255` to `255`
- `STOP` — stop both motors

Examples:

```text
M,100,100
M,-80,80
STOP
```

The example firmware has a 500 ms command timeout. It uses a generic dual H-bridge mapping and Arduino-ESP32 core 2.x LEDC APIs; changes may be required for different motor drivers, GPIO assignments, or Arduino-ESP32 core versions.

## Sensor integration checklist

- [ ] Record exact part numbers for all five e-nose sensors and add them to the hardware BOM.
- [ ] Document analog/digital output, operating voltage, heater current, warm-up time, and calibration procedure for each gas sensor.
- [ ] Verify ADC voltage compatibility before connecting sensor outputs to a controller.
- [ ] Log raw readings with timestamps, sensor warm-up state, and ambient conditions.
- [ ] Calibrate the IMU and magnetometer after installing them on the rover.
- [ ] Verify encoder polarity, counts per revolution, wheel diameter, and wheel separation.
- [ ] Define a documented message format between the embedded controller and Raspberry Pi.
- [ ] Test motor direction, emergency stop, and communication-loss behavior with wheels lifted.
- [ ] Document the integrated 2D LiDAR model, interface, mounting, scan topic/protocol, and validation results.
- [ ] Treat mmWave sensing as planned/experimental until hardware and software validation are complete.

## Safety and responsible use

- This project is a robotics and sensing prototype, not certified mine-detection or explosive-detection equipment.
- Do not use it to declare an area safe, locate live mines, or replace trained professionals and approved equipment.
- Never connect motors directly to controller GPIO pins. Use a properly rated motor driver and power source.
- Keep the electronics and motor power arrangement within each board/driver's specifications; establish common ground where required.
- Validate fail-safe stopping, physical emergency stop, wiring, and thermal/current limits before ground testing.
- Do not operate the rover unattended based on the included preview or teleoperation scripts.

## Contributing

Contributions should document the hardware revision, exact sensor models, wiring/interface changes, software dependencies, and test conditions. Keep experimental results separate from validated capabilities.

## License

Add the license selected for your project before publishing or accepting external contributions.
