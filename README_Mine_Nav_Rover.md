# Mine Nav Rover

**A Raspberry Pi 5 and FPGA-assisted autonomous rover for GPS-denied navigation, perception, and remote manual control.**

Mine Nav Rover combines onboard perception and navigation with low-level motor control. The Raspberry Pi 5 processes camera and 2D LiDAR data, while the Vicharak Shrike-fi is responsible for motor-control and encoder-interface tasks. A local Wi-Fi manual-override feature is planned around an ESP-based controller and a PC, allowing an operator to take control when needed.

> **Project status:** Hardware architecture and software integration are under development. Features marked **Planned / In integration** should not be considered operational until tested on the rover.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Current Hardware Architecture](#current-hardware-architecture)
- [Power Architecture](#power-architecture)
- [Software Stack](#software-stack)
- [Local Wi-Fi Manual Override](#local-wi-fi-manual-override)
- [Data and Control Flow](#data-and-control-flow)
- [Repository Structure](#repository-structure)
- [Software Setup](#software-setup)
- [Hardware Integration Notes](#hardware-integration-notes)
- [Testing Checklist](#testing-checklist)
- [Safety Considerations](#safety-considerations)
- [Roadmap](#roadmap)

---

## Project Overview

Mine Nav Rover is designed for mobile robotic operation in environments where GPS may be unavailable or unreliable. It combines:

- Camera-based perception using OpenCV.
- 2D LiDAR data for environmental sensing and mapping/localization integration.
- Raspberry Pi 5 for high-level processing and software orchestration.
- Vicharak Shrike-fi for low-level motor control and encoder interfacing.
- L298N motor driver for driving two DC encoder motors.
- Local Wi-Fi manual override from a PC through an ESP-based controller.
- A protected 3S LiPo power system with dedicated regulated rails.

The software is divided into high-level autonomy, hardware communication, and operator override so that manual commands can take priority over autonomous motion.

## Current Hardware Architecture

| Component | Responsibility | Connection / Notes |
|---|---|---|
| Raspberry Pi 5 | Main computer; OpenCV, perception, navigation, and system coordination | Powered from dedicated 12 V → 5.1 V buck converter |
| Vicharak Shrike-fi | Motor-control tasks and encoder interfacing | Dedicated regulated 5 V supply; check board documentation for allowed power input |
| L298N motor driver | Drives left and right DC motors | Motor supply from protected battery bus; control inputs from Shrike-fi |
| Left and right encoder motors | Rover locomotion and wheel feedback | Motor outputs from L298N; encoder signals to Shrike-fi |
| 2D LiDAR | Scan data for mapping and navigation | USB to Raspberry Pi 5; powered from Pi USB, subject to the LiDAR and Pi power budgets |
| USB webcam | Camera frames for OpenCV | USB to Raspberry Pi 5 |
| ESP-based Wi-Fi controller | Local wireless command bridge for manual override | Wi-Fi link to the PC/operator interface; electrical/control interface to be finalized |
| PC / operator station | Manual-control interface and status display | Connects to the rover's local Wi-Fi network |
| 3S LiPo battery | Main energy source | 11.1 V nominal, 12.6 V fully charged; use suitable protection and charging practices |
| DC-DC buck converters | Regulated supplies for Pi and Shrike-fi | Dedicated converter for each rail is recommended |

### High-level block diagram

```text
                         3S LiPo Battery
                                |
                   Fuse + Master Switch + Protection
                                |
                        Protected Power Bus
              ________________|________________
             |                |                |
             v                v                v
        L298N Motor      12 V -> 5.1 V    12 V -> 5.0 V
           Driver          Buck               Buck
             |                |                 |
       Left/Right DC       Raspberry Pi 5    Shrike-fi
       Encoder Motors          |                 |
             |          ______|_______      Motor control
       Encoder feedback  |      |      |     + encoder I/O
             |           v      v      v           ^
             +-------> Shrike-fi 2D LiDAR Webcam   |
                         ^       USB      USB       |
                         |__________________________|
                              USB data link

 PC / Operator Controller
          |
   Local Wi-Fi network
          |
 ESP-based manual-override controller
          |
   Defined command interface
          |
   Control arbitration / safety logic
          |
   Shrike-fi motor-control commands
```

**Integration note:** The diagram shows the intended logical flow. The exact ESP-to-Shrike-fi wiring/protocol and the USB data path must match the actual board interfaces and firmware. Do not assume the ESP can directly drive Shrike-fi signals without checking voltage levels and interface support.

---

## Power Architecture

The rover uses a **3S LiPo battery** as its main source.

- **Nominal pack voltage:** 11.1 V.
- **Fully charged pack voltage:** 12.6 V.
- **Motor branch:** protected battery bus to the L298N motor supply, only if the motors and driver are rated for the operating voltage/current.
- **Raspberry Pi 5 branch:** dedicated 12 V → 5.1 V buck converter capable of supplying the required load. A 5 V / 5 A class supply is recommended for full Pi 5 operation, with the correct USB-C power-delivery implementation if powering through USB-C.
- **Shrike-fi branch:** dedicated 12 V → regulated 5 V converter, sized for the board and attached loads. Follow the board's documented power-input method.
- **LiDAR and webcam:** powered through the Raspberry Pi USB ports in this design. Verify the total USB current demand and use a suitable powered USB hub if needed.
- **Grounding:** use a common reference and plan the return wiring so high motor current does not flow through sensitive signal/USB ground paths.

### Recommended protection and decoupling

| Location | Starting point | Purpose |
|---|---|---|
| Battery output | Main fuse close to battery positive | Protects main wiring |
| Individual branches | Correctly rated branch fuses | Isolates faults |
| Battery input | Reverse-polarity protection; consider a correctly selected TVS strategy | Reduces risk from wiring mistakes and transients |
| L298N supply near driver | 470–1000 µF electrolytic rated at least 25 V, plus 100 nF ceramic | Buffers supply transients |
| Pi supply near load | 470–1000 µF electrolytic rated suitably above 5.1 V, plus 100 nF ceramic | Helps reduce local transients |
| Shrike-fi supply near load | 220–470 µF electrolytic rated suitably above 5 V, plus 100 nF ceramic | Helps reduce local transients |
| Brushed motor terminals | Optional 100 nF ceramic across each motor, if suitable for the motor/driver | Helps reduce brush noise |

These are starting values, not a finalized protection design. Check buck-converter stability requirements, capacitor ripple-current ratings, battery discharge capability, fuse selection, wire gauge, and actual motor stall current before operation. Capacitors do not replace proper regulation, wiring, fusing, or transient protection.

> **Power caution:** Do not feed battery voltage directly to the Raspberry Pi 5 or Shrike-fi. Do not power Shrike-fi simultaneously from USB-C and its 5 V header unless the official documentation explicitly permits that configuration. Verify the exact Shrike-fi power-input specifications before wiring.

---

## Software Stack

The software is organized into layers so perception and navigation can evolve without putting timing-sensitive motor control on the Raspberry Pi's general-purpose operating system.

| Layer | Technology | Role | Status |
|---|---|---|---|
| Main operating system | Raspberry Pi OS (64-bit; Lite or Desktop as required) | Runs rover services and hardware interfaces | Setup-dependent |
| Programming language | Python 3 | High-level control, integration, tooling | Core choice |
| Computer vision | OpenCV (`cv2`) | Webcam capture, image processing, visual detection | Core software component |
| LiDAR interface | Driver/SDK appropriate to the installed 2D LiDAR | Acquires scan data over USB/serial | Model-dependent |
| Navigation and mapping | ROS 2 or a custom Python pipeline, depending on the selected implementation | Scan processing, mapping, localization, path planning | Confirm the project's chosen framework before deployment |
| Hardware communication | USB serial or the supported Shrike-fi interface/protocol | Sends high-level commands and receives telemetry | In integration |
| Low-level motor control | Shrike-fi firmware / supported I/O | Motor-control signals and encoder feedback | Hardware-dependent |
| Motor driver | L298N | H-bridge power stage for two DC motors | Hardware |
| Manual override | ESP-based local Wi-Fi service + PC controller UI | Operator commands and mode switching | Planned / In integration |
| Command arbitration | Software state machine | Prioritizes emergency stop, manual control, and autonomy | Required integration feature |
| Logging and diagnostics | Python logging; optional CSV/JSON logs | Records telemetry, commands, faults, and sensor status | Recommended |

### Suggested software modules

1. **`camera.py`** — opens the USB webcam and supplies frames to OpenCV.
2. **`lidar_interface.py`** — reads scans using the correct driver for the installed LiDAR.
3. **`shrike_link.py`** — implements the documented USB/serial or other supported communication protocol to Shrike-fi.
4. **`autonomy.py`** — high-level navigation and autonomous command generation.
5. **`manual_override.py`** — receives validated manual commands from the ESP/PC pathway.
6. **`control_arbiter.py`** — selects which command source is allowed to control motion.
7. **`safety.py`** — enforces command timeouts, valid speed limits, connection-loss behavior, and stop states.
8. **`telemetry.py`** — publishes status and records sensor/controller health.
9. **`main.py`** — starts services and coordinates the system lifecycle.

These are suggested module responsibilities; create files only as implementation is added to the repository.

---

## Local Wi-Fi Manual Override

### Goal

Provide an operator with a way to manually control the rover from a PC connected to the rover's **local Wi-Fi network**, using an ESP-based controller as the wireless command bridge.

This feature is intended for supervised testing, recovery, and operator intervention. It should not depend on internet access.

### Proposed communication path

```text
PC controller interface
  (keyboard / on-screen controls / game controller)
              |
              | Local Wi-Fi
              v
      ESP-based controller
       (command validation)
              |
              | Defined wired or wireless interface
              v
      Control arbitration layer
              |
              | Authorized target speed / stop command
              v
           Shrike-fi
              |
              v
         L298N driver
              |
              v
        Left/right motors
```

The exact ESP model, firmware, and ESP-to-Shrike-fi interface must be selected and tested. If the ESP connects to the PC over Wi-Fi, it can operate as an access point or join a dedicated local router/network. Keep this control network local; cloud access is not required.

### Manual-control behavior

The planned PC interface can provide:

- Forward and reverse commands.
- Left and right steering or differential wheel-speed commands.
- Stop / emergency-stop command.
- Manual / autonomous mode selection.
- Connection status and basic telemetry, where implemented.
- Configurable speed limits for safe indoor or bench testing.

### Command priority

The control arbiter should use this priority order:

1. **Emergency stop / fault stop**
2. **Manual override**, while the operator session is valid
3. **Autonomous navigation**, only when autonomy is enabled and no higher-priority stop/manual command is active
4. **Safe idle**, when no valid command is available

Manual override must not bypass low-level motor limits or safety checks. The Shrike-fi should remain responsible for its local motor-control behavior and encoder handling.

### Required safety behavior

- Commands must be range-checked and rejected if malformed.
- Use a heartbeat or command timeout; if the link is lost, command a safe stop.
- Require an explicit action to leave emergency stop.
- Prevent automatic resumption of motion after reconnection unless the operator explicitly re-enables it.
- Apply speed and acceleration limits.
- Provide a physical power cut-off accessible to the operator.
- Keep Wi-Fi credentials private and restrict control access to authorized devices.
- Do not rely on Wi-Fi as the only emergency-stop mechanism.

**Status:** This is a planned software feature until the ESP firmware, PC interface, command protocol, control arbitration, and loss-of-link tests have been implemented and verified.

---

## Data and Control Flow

### Sensor-to-compute flow

1. The USB webcam streams frames to the Raspberry Pi 5.
2. OpenCV processes camera frames for the perception tasks implemented by the project.
3. The 2D LiDAR streams scan data to the Raspberry Pi 5 over USB.
4. The selected LiDAR driver exposes scans to the mapping/navigation pipeline.
5. High-level software generates movement targets or stop requests.
6. The Pi sends supported high-level commands to Shrike-fi over the configured communication link.
7. Shrike-fi handles motor-control outputs and reads encoder feedback.
8. Encoder and system telemetry are returned to the high-level software if the selected interface supports it.

### Manual override flow

1. The operator connects a PC to the rover's local Wi-Fi network.
2. The PC interface sends a movement or stop request to the ESP-based controller.
3. The ESP validates the request and forwards it through the defined command interface.
4. The control arbiter gives the command the correct priority.
5. Authorized motor targets are passed to Shrike-fi.
6. If the command link times out, the system enters its configured safe-stop state.

---

## Repository Structure

A suggested layout for the software as it grows:

```text
mine-nav-rover/
├── README.md
├── requirements.txt
├── config/
│   ├── rover.yaml
│   └── manual_control.yaml
├── docs/
│   ├── architecture.md
│   ├── power-schematic.png
│   └── wiring-notes.md
├── src/
│   ├── main.py
│   ├── camera.py
│   ├── lidar_interface.py
│   ├── shrike_link.py
│   ├── autonomy.py
│   ├── manual_override.py
│   ├── control_arbiter.py
│   ├── safety.py
│   └── telemetry.py
├── esp/
│   ├── README.md
│   └── firmware/              # Add ESP firmware when implemented
└── tests/
    ├── test_control_arbiter.py
    └── test_command_validation.py
```

This is a proposed structure. It does not imply that all files or tests already exist.

---

## Software Setup

### 1. Prepare the Raspberry Pi

Use a supported 64-bit Raspberry Pi OS installation. Update the package index and system packages:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo reboot
```

### 2. Install basic tools

```bash
sudo apt install -y python3 python3-venv python3-pip git
```

### 3. Clone the repository

Replace the placeholder with the actual repository URL:

```bash
git clone <YOUR_REPOSITORY_URL>
cd mine-nav-rover
```

### 4. Create a Python environment

```bash
python3 -m venv .venv --system-site-packages
source .venv/bin/activate
python -m pip install --upgrade pip
```

### 5. Install OpenCV

On Raspberry Pi OS, using the distribution package is often convenient:

```bash
sudo apt install -y python3-opencv
```

Check the installation:

```bash
python3 -c "import cv2; print(cv2.__version__)"
```

If using the virtual environment above, `--system-site-packages` allows it to access the system OpenCV package. Choose one dependency strategy and keep it consistent.

### 6. Add the appropriate LiDAR library

Install the SDK/driver for the **exact LiDAR model** in use. Do not copy a package name from another model's instructions. Verify USB device permissions and the device path before running the driver.

### 7. Configure hardware communication

Document the confirmed Shrike-fi interface, serial device path or protocol, baud rate if applicable, command format, and telemetry format in `docs/wiring-notes.md` or the relevant module. Do not assume a serial protocol until it is implemented on both sides.

### 8. Configure the manual-override network

When the ESP firmware and PC interface are implemented:

1. Configure the ESP for the chosen local network mode (access point or dedicated LAN).
2. Set a unique network name and strong password.
3. Configure the command endpoint/protocol and allowed operator devices.
4. Set command timeouts and speed limits.
5. Test the command interface with the motors physically disabled or the rover securely raised before enabling drive power.

---

## Hardware Integration Notes

- Verify the exact L298N module, motor rated voltage, and motor **stall current**. The L298N has substantial voltage drop and heat dissipation; confirm it can safely drive the selected motors.
- Check that Shrike-fi output logic levels are compatible with the L298N input pins. Do not assume compatibility without checking the board and driver specifications.
- Check encoder supply voltage and output levels before connecting encoder signals. Protect 3.3 V inputs from higher-voltage signals.
- Use the documented Shrike-fi power input. Do not simultaneously apply power through multiple inputs unless explicitly supported.
- The Raspberry Pi USB power budget must cover the LiDAR and webcam together. If necessary, use a suitable powered USB hub.
- Use short, well-routed USB and signal wiring. Keep motor wires away from encoder, USB, and other sensitive signal cables.
- Use a common electrical ground where required by the selected interfaces, while routing high-current motor returns separately from sensitive signal returns.
- Choose fuses, wire gauge, connectors, and converter ratings based on measured or documented current—not only nominal current.
- Verify all capacitor polarities and voltage ratings before powering the system.

---

## Testing Checklist

### Power and hardware

- [ ] Verify 3S LiPo polarity and pack voltage before connecting electronics.
- [ ] Confirm fuse ratings and polarity-protection orientation.
- [ ] Measure the unloaded and loaded output of each buck converter.
- [ ] Verify Pi input voltage under load and confirm the LiDAR/webcam power budget.
- [ ] Verify Shrike-fi input voltage and power-input configuration.
- [ ] Test motor driver and motors with wheels lifted and current limits considered.
- [ ] Verify encoder direction, counts, and logic levels.

### Software

- [ ] Confirm the webcam can be opened and frames are received.
- [ ] Confirm the LiDAR driver receives valid scan data.
- [ ] Confirm the Pi can communicate with Shrike-fi using the implemented protocol.
- [ ] Validate motor commands and encoder telemetry at low speed.
- [ ] Confirm that invalid commands are rejected.
- [ ] Confirm command timeout results in a safe stop.
- [ ] Confirm manual mode takes priority over autonomy.
- [ ] Confirm emergency stop takes priority over every movement command.
- [ ] Test Wi-Fi loss, ESP restart, Pi restart, and reconnection behavior.
- [ ] Verify the rover does not move automatically after reconnecting.

---

## Safety Considerations

This rover includes moving motors, a high-current battery, and exposed electronics. Testing should be supervised.

- Use a LiPo-compatible charger and follow the battery manufacturer's charge, storage, and discharge instructions.
- Never short-circuit or leave a damaged or swollen LiPo pack in service.
- Secure the battery and electronics mechanically.
- Keep wheels raised during initial motor-control and software tests.
- Provide a physical battery disconnect and accessible emergency-stop method.
- Avoid connecting or disconnecting motor wiring while powered.
- Do not treat software stop commands or Wi-Fi connectivity as a replacement for physical power isolation.

---

## Roadmap

- [x] Raspberry Pi 5 as the high-level compute platform.
- [x] Shrike-fi selected for motor-control and encoder-interface tasks.
- [x] L298N motor-driver architecture for two DC encoder motors.
- [x] 2D LiDAR and USB webcam connected to the Raspberry Pi 5.
- [x] 3S LiPo-based power architecture planned with dedicated buck converters.
- [ ] Integrate and validate the LiDAR driver and scan pipeline.
- [ ] Complete OpenCV perception pipeline for the selected use cases.
- [ ] Finalize the Pi-to-Shrike-fi communication protocol.
- [ ] Implement and test command arbitration and safe-stop behavior.
- [ ] Implement ESP-based local Wi-Fi manual override from a PC.
- [ ] Add telemetry dashboard, logging, and fault reporting.
- [ ] Validate the full system under controlled test conditions.

---

## Contributing

Keep wiring diagrams, software configuration, and tested behavior synchronized with the README. When adding a feature, document its hardware dependencies, setup steps, safety behavior, and validation results.

## License

Add the license selected for this project before publishing or distributing the repository.
