# Web‑controlled RC Car (ESP32‑CAM)

Turn an ESP32‑CAM into a Wi‑Fi video streaming, web‑controlled RC platform. This project combines a live MJPEG camera stream with a simple web UI and HTTP API to drive motors and servos for steering and camera pan/tilt. It’s based on the excellent esp32‑cam‑webserver and extends it with RC controls.

![Breadboard design](./Wifi%20controlled%20car%20with%20esp32cam%20design_bb.png)

## What you get

- Live MJPEG video stream and snapshots from ESP32‑CAM
- Web UI with “simple” and “full” views
- RC controls:
  - DC motor forward/stop/reverse (H‑bridge)
  - Steering servo (left/center/right)
  - Camera tilt and pan servos
- Lamp/flash LED control, status LED feedback
- Wi‑Fi Station + fallback Access Point with captive portal
- mDNS hostname, optional NTP clock
- Save/restore camera preferences (SPIFFS)
- OTA firmware updates

## Repository layout

- `esp32-cam-webserver/` — firmware (Arduino/PlatformIO). Contains core web server, UI, camera, and RC extensions.
  - `esp32-cam-webserver.ino` — main sketch
  - `app_httpd.cpp` — HTTP routes, stream handler, API controls (incl. motor/servos)
  - `camera_pins.h` — board pin mappings (incl. motor/servo pins for AI‑Thinker)
  - `myconfig.h` — your local Wi‑Fi, ports, features, and board selection
  - `API.md` — base API documentation (extended here for RC)
  - `platformio.ini` — PlatformIO configuration
- Project media: wiring images and docs under `Docs/` and repo root

## Hardware

Minimum:

- AI‑Thinker ESP32‑CAM (or compatible ESP32 camera board with PSRAM)
- 5V 2A power source for ESP32‑CAM (stable power is critical)
- H‑bridge motor driver (e.g., L298N) for the DC motor(s)
- Standard RC servos for steering and camera pan/tilt (as used)
- Optional: high‑power LED on GPIO 4 (AI‑Thinker lamp pin) for illumination

Default pin mapping for AI‑Thinker (see `esp32-cam-webserver/camera_pins.h`):

- Lamp: GPIO 4
- Status LED: GPIO 33 (active‑low)
- Pan servo: GPIO 12
- Tilt servo: GPIO 13
- Steering servo: GPIO 2
- Motor driver: GPIO 14, GPIO 15 (direction control)

Notes
- Power servos and motors from a suitable external supply; share GND with ESP32‑CAM.
- PSRAM must be available and enabled for higher camera resolutions.

## Configure

Open `esp32-cam-webserver/myconfig.h` and adjust:

- Wi‑Fi: add your SSIDs/passwords in `stationList[]`.
- AP fallback: define `WIFI_AP_ENABLE`. The AP SSID/password are taken from the first `stationList[]` entry.
- Hostname: `MDNS_NAME` and `URL_HOSTNAME`.
- Ports: `HTTP_PORT` (default 80), `STREAM_PORT` (default 81).
- Features: enable/disable lamp, LED, filesystem, OTA, etc.
- Board: select exactly one `CAMERA_MODEL_*` (defaults to AI_THINKER).
- Optional: static IP for STA mode, NTP server/timezone, default resolution, rotation.

Keep secrets out of version control. This repo already includes a `myconfig.h` for demo—replace the credentials with your own before flashing.

## Build and upload

You can use Arduino IDE or PlatformIO. Ensure ESP32 board support is installed and PSRAM is enabled.

\n### Arduino IDE
1. Install ESP32 core (v2.x recommended) via Boards Manager.
2. Board: “ESP32 Dev Module” or “ESP32 Wrover Module”. Enable PSRAM.
3. Partition scheme: “Minimal SPIFFS (1.9MB APP with OTA/190KB SPIFFS)”.
4. Open `esp32-cam-webserver/esp32-cam-webserver.ino`.
5. Connect a 3.3V USB‑to‑serial adapter. For AI‑Thinker, pull GPIO0 to GND on power‑up to enter flashing mode.
6. Upload. Open Serial Monitor at 115200 baud to see the assigned IP or AP address.

Tip: After the first serial upload, you can use OTA updates if enabled in `myconfig.h`.

\n### PlatformIO (VS Code)
1. Open the folder `esp32-cam-webserver/` as the project root.
2. Select the `env:esp32dev` environment from the status bar.
3. Build and upload via PlatformIO. The config pins the Arduino framework (2.0.2) and includes `ESP32Servo`.

If you prefer the CLI from Windows PowerShell, run these inside `esp32-cam-webserver/` (optional):

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command "& { pio run }"
powershell -NoProfile -ExecutionPolicy Bypass -Command "& { pio run -t upload }"
```

## First run

- If STA Wi‑Fi connects: browse to `http://<device-ip>/` or `http://esp32-cam/` via mDNS.
- If STA fails and AP is enabled: connect to the AP SSID from the first `stationList[]` entry, then open the AP IP (default often `192.168.4.1`, or the `AP_ADDRESS` you set in `myconfig.h`).
- Use the web UI to start the stream and adjust settings. A minimal “simple” view and a “full” control view are available.

Stream viewer:

- MJPEG stream: `http://<host>:81/`
- Embedded viewer page: `http://<host>:81/view`
- Snapshot (JPEG): `http://<host>/capture`

## Web API overview

The project speaks plain HTTP. You can automate RC actions and camera settings with simple GETs. Full base API is documented in `esp32-cam-webserver/API.md`.

Important routes

- App: `GET /`, `GET /?view=simple|full|portal`
- Snapshot: `GET /capture`
- Status JSON: `GET /status`
- Control: `GET /control?var=<key>&val=<value>`
- Stream: `GET :81/` and `GET :81/view`
- Kill stream: `GET /stop`

Common camera keys (subset)

- `framesize`, `quality`, `brightness`, `contrast`, `saturation`, `hmirror`, `vflip`, `min_frame_time`, `lamp`, `autolamp`, `rotate`, etc.

RC‑specific controls (extensions)

- Pan servo: `var=horizontalServoSlider&val=0..180`
- Tilt servo: `var=verticalServoSlider&val=0..90`
- Steering servo: `var=steeringServoSlider&val=0..360` (50 ≈ straight, 0/100 ≈ right/left)
- Motor: `var=motorController&val=-1|0|1` (−1 back, 0 stop, 1 forward)

Examples

- Set VGA resolution: `http://<host>/control?var=framesize&val=8`
- Turn lamp to 50%: `http://<host>/control?var=lamp&val=50`
- Center steering: `http://<host>/control?var=steeringServoSlider&val=50`
- Drive forward: `http://<host>/control?var=motorController&val=1`

## Troubleshooting

- Power: Brown‑outs cause reboots, Wi‑Fi drops, and corrupted frames. Use a solid 5V supply; power motors/servos separately from the ESP32‑CAM and share ground.
- One stream at a time: The MJPEG stream supports a single client. Use `/stop` to free the stream.
- PSRAM required: High resolutions need PSRAM; enable it in board settings.
- Wi‑Fi: Short power leads, decent antenna, and non‑congested channels help stability.
- Pins: Verify your board profile in `camera_pins.h` matches your hardware.

## License and credits

This project includes and extends code from:

- easytarget/esp32‑cam‑webserver — Apache‑2.0. See `esp32-cam-webserver/LICENSE`.

Additional libraries:

- ESP32 Arduino core by Espressif
- ESP32Servo (via PlatformIO `lib_deps`)

Unless otherwise noted, project additions are provided under the same license as the upstream component.

## Acknowledgments

Thanks to the ESP32 community and upstream maintainers for the solid foundation this project builds on.

---

Happy hacking and safe driving!
