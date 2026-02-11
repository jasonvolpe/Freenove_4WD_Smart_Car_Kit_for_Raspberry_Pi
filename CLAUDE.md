# CLAUDE.md

## Project Overview

Freenove 4WD Smart Car Kit for Raspberry Pi — a robotics platform with a **client-server architecture** for controlling a 4-wheel drive smart car over TCP/IP. The server runs on a Raspberry Pi (on the car), and the client runs on a desktop/laptop.

## Repository Structure

```
.
├── Code/
│   ├── Server/          # Raspberry Pi on-board server (hardware control + networking)
│   ├── Client/          # Desktop PyQt5 GUI client (remote control + video display)
│   ├── Libs/            # External C library (rpi-ws281x LED driver with SWIG bindings)
│   ├── setup.py         # Linux/RPi setup script (installs deps, configures /boot/firmware/config.txt)
│   ├── setup_macos.py   # macOS client setup
│   └── setup_windows.py # Windows client setup
├── Application/         # Pre-built client executables (Windows .exe, macOS binary)
├── Datasheet/           # Hardware datasheets (PCA9685, ADS7830, PCF8591)
├── Picture/             # Documentation images
└── *.pdf                # Tutorials and battery info
```

## Architecture

### Client-Server Model

```
Desktop Client (PyQt5)              Raspberry Pi Server
─────────────────────               ──────────────────
Main.py (entry point)               main.py (entry point, Qt5 UI)
Client_Ui.py (GUI)                  server.py (TCP listener)
Video.py (video display,            camera.py (picamera2 JPEG streaming)
  face detection)                   car.py (orchestrates all hardware)
Command.py (protocol constants)     command.py (protocol constants)
```

**TCP Connections:**
- **Port 5000** — Command channel (text-based protocol)
- **Port 8000** — Video channel (JPEG frames with size headers)

### Message Protocol

Commands use `#`-delimited format: `COMMAND#param1#param2#...#paramN`

Examples:
- `CMD_MOTOR#2000#2000#2000#2000` — Set all 4 motor duty cycles
- `CMD_SERVO#0#90` — Set servo channel 0 to 90 degrees
- `CMD_LED#0#255#0#0#0` — Set LED 0 to red
- `CMD_BUZZER#1` — Buzzer on

### Hardware Components

| Component | Interface | Driver File | Details |
|-----------|-----------|-------------|---------|
| 4x DC Motors | I2C (PCA9685) | `motor.py` | 2 PWM pins per motor |
| Pan/Tilt Servos | I2C (PCA9685) | `servo.py` | Channels 8-15 |
| 8x WS2812B LEDs | GPIO18 or SPI | `led.py`, `rpi_ledpixel.py`, `spi_ledpixel.py` | Dual implementation |
| Camera | CSI | `camera.py` | picamera2, ov5647/imx219 |
| Ultrasonic (HC-SR04) | GPIO27/22 | `ultrasonic.py` | gpiozero DistanceSensor |
| 3x IR Sensors | GPIO14/15/23 | `infrared.py` | Line following |
| 2x Photoresistors | I2C (ADS7830) | `photoresistor.py`, `adc.py` | Channels 0-1 |
| Power Monitor | I2C (ADS7830) | `adc.py` | Channel 2 |
| Buzzer | GPIO17 | `buzzer.py` | On/off control |

**I2C Addresses:** PCA9685 @ `0x40`, ADS7830 @ `0x48`

## Languages and Dependencies

**Languages:** Python 3 (primary), C (LED library internals)

**Key Python packages:**
- `PyQt5` — GUI framework (both server and client)
- `opencv-python` (cv2) — Video processing, face detection
- `gpiozero` — GPIO abstraction
- `picamera2` — Raspberry Pi camera
- `numpy` — Image/numerical operations
- `spidev` — SPI interface
- `smbus` — I2C interface
- `rpi-ws281x` — WS2812B LED control (bundled in `Code/Libs/`)

**Setup:** Run `sudo python Code/setup.py` on the Raspberry Pi. It installs all Python and system dependencies via `pip3` and `apt-get`, and configures `/boot/firmware/config.txt` for camera and SPI.

## Key Source Files

### Server (`Code/Server/`)

| File | Purpose |
|------|---------|
| `main.py` | Entry point; Qt5 UI for server management |
| `server.py` | TCP server setup; manages dual-port connections |
| `tcp_server.py` | Low-level TCP socket accept/listen |
| `message.py` | `Message_Parse` class; parses `#`-delimited commands |
| `command.py` | Command constant definitions |
| `car.py` | Central hardware orchestrator |
| `motor.py` | Motor control via PCA9685 PWM |
| `servo.py` | Servo control via PCA9685 |
| `led.py` | LED strip control (auto-selects GPIO or SPI driver) |
| `camera.py` | picamera2 video capture and JPEG streaming |
| `ultrasonic.py` | HC-SR04 distance measurement |
| `infrared.py` | IR line-following sensor reads |
| `adc.py` | ADS7830 analog-to-digital converter |
| `pca9685.py` | PCA9685 I2C PWM driver |
| `parameter.py` | JSON-based hardware config (`params.json`) |
| `test.py` | Manual hardware test functions |

### Client (`Code/Client/`)

| File | Purpose |
|------|---------|
| `Main.py` | Entry point; PyQt5 GUI application |
| `Client_Ui.py` | Generated Qt UI code (from `Client_Ui.ui`) |
| `Video.py` | Video stream receiver; face detection overlay |
| `Command.py` | Command constants (mirrors server) |
| `IP.txt` | Stores server IP address |

## Platform Support

- **Raspberry Pi 3, 4, 5** — Server (auto-detected via `parameter.py`)
- **Camera models:** ov5647 (v1), imx219 (v2)
- **PCB versions:** V1.0, V2.0 (different GPIO mappings for LEDs)
- **Client platforms:** Linux, macOS, Windows

## Testing

There is no automated test framework (no pytest, CI/CD, or linting config).

**Manual hardware testing:** `Code/Server/test.py` contains standalone test functions for each hardware component (LED, motor, ultrasonic, servo, ADC, buzzer). Each module also has `if __name__ == '__main__':` blocks for standalone testing.

To test a specific component on the Pi:
```bash
cd Code/Server
python3 test.py          # Runs all hardware tests
python3 motor.py         # Test motors standalone
python3 led.py           # Test LEDs standalone
python3 ultrasonic.py    # Test ultrasonic sensor
```

## Development Conventions

- **No automated linting or formatting** is enforced. Follow existing code style (PEP 8 loosely).
- **Command constants** must stay synchronized between `Code/Server/command.py` and `Code/Client/Command.py`. Any new command must be added to both files.
- **Hardware drivers** are one-file-per-component in `Code/Server/`. Each driver wraps an I2C/GPIO/SPI interface.
- **Configuration** is stored in `Code/Server/params.json` (auto-generated at runtime). It tracks `Connect_Version`, `Pcb_Version`, and `Pi_Version`.
- **UI changes** to the client should be made in `Client_Ui.ui` (Qt Designer) and regenerated to `Client_Ui.py`.
- **LED control** has two implementations: GPIO-based (`rpi_ledpixel.py`) and SPI-based (`spi_ledpixel.py`). The `led.py` module selects the appropriate driver based on PCB version.
- **Thread management** uses a custom `Thread.py` utility for async operations in both server and client.

## Running the Project

**Server (on Raspberry Pi):**
```bash
cd Code/Server
sudo python3 main.py
```
The server must run as root (`sudo`) for GPIO/I2C/SPI hardware access.

**Client (on desktop):**
```bash
cd Code/Client
python3 Main.py
```
Enter the Raspberry Pi's IP address when prompted (or edit `IP.txt`).

## License

Creative Commons Attribution-NonCommercial-ShareAlike 3.0 Unported (CC BY-NC-SA 3.0).
