# 🤖 ESP32 Line-Following Robot with Obstacle Detection & Web Control

A WiFi-controlled, autonomous line-following robot built on the **ESP32** platform. The robot navigates a predefined track from **Point A → B → C**, halts at checkpoints, detects and waits for obstacles using an ultrasonic sensor, and exposes a real-time web dashboard for monitoring and control.

---

## 📸 Features

- ✅ **Autonomous line following** using 3 IR sensors (left, center, right)
- ✅ **3-point state machine** — A → B (halt 5 sec) → C (final destination)
- ✅ **Obstacle detection** via HC-SR04 ultrasonic sensor (stops within 15 cm)
- ✅ **Auto-resume** after obstacle is cleared (requires 3 consecutive clear readings)
- ✅ **Real-time web UI** hosted on the ESP32 (auto-refreshes every 1 second)
- ✅ **Static IP** for reliable local network access
- ✅ **Non-blocking sensor reads** — ultrasonic polled every 100ms, won't stall line following
- ✅ **False checkpoint filtering** — 100ms confirmation window avoids spurious stops

---

## 🛠️ Hardware Requirements

| Component | Quantity |
|---|---|
| ESP32 Dev Board | 1 |
| L298N Motor Driver | 1 |
| DC Gear Motors | 2 |
| IR Sensors (Digital) | 3 |
| HC-SR04 Ultrasonic Sensor | 1 |
| Voltage Divider (1kΩ + 2kΩ resistors) | 1 |
| Robot Chassis | 1 |
| Power Supply (7–12V for motors) | 1 |
| 3.3V / 5V Regulated Supply for logic | 1 |

---

## ⚡ Important: Voltage Divider for Ultrasonic Echo Pin

> **This is critical — skipping this will damage your ESP32.**

The **ESP32 GPIO pins are 3.3V tolerant only**. The HC-SR04 ultrasonic sensor is powered externally at **5V**, which means its **ECHO pin outputs 5V logic** — this will exceed the safe input voltage of the ESP32 and can permanently damage the GPIO.

### Solution: Voltage Divider on ECHO Pin

Use a simple resistor voltage divider to step the 5V ECHO signal down to ~3.3V before connecting to the ESP32:

```
HC-SR04 ECHO (5V) ──┬── 1kΩ ──── ESP32 GPIO (echoPin)
                    │
                   2kΩ
                    │
                   GND
```

**Output voltage** = 5V × 2kΩ / (1kΩ + 2kΩ) = **3.33V** ✅

- The **TRIG pin** can be driven directly from the ESP32 (3.3V HIGH is sufficient to trigger the HC-SR04).
- Only the **ECHO pin** needs the voltage divider.
- The VCC of the HC-SR04 connects to your **5V external supply**, not the ESP32's 3.3V rail.

---

## 📌 Pin Configuration

| Signal | ESP32 GPIO |
|---|---|
| Motor ENA (Left PWM) | 13 |
| Motor IN1 | 12 |
| Motor IN2 | 14 |
| Motor IN3 | 27 |
| Motor IN4 | 26 |
| Motor ENB (Right PWM) | 25 |
| IR Sensor Left | 34 |
| IR Sensor Center | 35 |
| IR Sensor Right | 32 |
| Ultrasonic TRIG | 23 |
| Ultrasonic ECHO | 22 |

> GPIO 34, 35 are **input-only** on ESP32 — perfect for IR sensors.

---

## 🌐 Network Configuration

Edit the following constants in the sketch before uploading:

```cpp
const char* ssid     = "Your_WiFi_Name";
const char* password = "Your_WiFi_Password";

IPAddress local_IP(192, 168, 1, 100);   // Desired static IP
IPAddress gateway(192, 168, 1, 1);      // Your router IP
IPAddress subnet(255, 255, 255, 0);
```

Once connected, open a browser and navigate to:

```
http://192.168.1.100
```

---

## 🗺️ State Machine

```
[IDLE] ──(START)──► [TO_B] ──(all 3 IR HIGH, confirmed)──► [AT_B] ──(5 sec)──► [TO_C] ──(all IR LOW)──► [FINISHED]
           ▲                                                                                                    │
           └──────────────────────────────────(RESET)──────────────────────────────────────────────────────────┘

Any moving state ──(obstacle < 15cm)──► [OBJECT_DETECTED] ──(3× clear readings)──► (resume previous state)
```

### State Descriptions

| State | Behaviour |
|---|---|
| `IDLE` | Stationary at Point A. Awaiting START from web UI. |
| `TO_B` | Line following toward Point B. Obstacle detection active. |
| `AT_B` | Stopped at Point B for 5 seconds. |
| `TO_C` | Line following to final destination. |
| `FINISHED` | Stopped. RESET button available. |
| `OBJECT_DETECTED` | All motion halted. Waits for obstacle to clear (3 consecutive clear reads = ~300ms). |

---

## 🖥️ Web Dashboard

The built-in web server provides:

- Live **status message** (updates every 1 second via meta-refresh)
- **Obstacle distance** display when an object is within range
- **START** button (shown in IDLE state)
- **RESET** button (shown in FINISHED state)
- Color-coded alerts for obstacle detection

---

## 🔧 Line Following Logic

The robot uses **pivot steering** — one motor drives while the other stops — for sharp, responsive corrections.

| IR Left | IR Center | IR Right | Action |
|---|---|---|---|
| LOW | HIGH | LOW | Forward |
| HIGH | LOW | — | Turn Left |
| — | LOW | HIGH | Turn Right |
| HIGH | HIGH | — | Turn Left |
| — | HIGH | HIGH | Turn Right |
| LOW | LOW | LOW | Forward (lost line, continue) |
| HIGH | HIGH | HIGH | Stop (checkpoint detected) |

> A **100ms confirmation window** prevents false checkpoint detection from noise.

---

## 🚧 Obstacle Detection Details

- Ultrasonic sensor is polled **every 100ms** (non-blocking)
- Robot stops immediately if an object is detected **within 15cm**
- Resume requires **3 consecutive clear readings** (~300ms) to avoid flickering
- Previous state is saved and fully restored on resume, including remaining halt time at Point B
- Detection is **disabled in IDLE and FINISHED** states

---

## 📦 Software Dependencies

- **Arduino IDE** with ESP32 board support
- Libraries (all built-in with ESP32 core):
  - `WiFi.h`
  - `WebServer.h`

### Installing ESP32 Board Support

In Arduino IDE → Preferences → Additional Board Manager URLs, add:

```
https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
```

Then: Tools → Board → Board Manager → search **esp32** → Install.

---

## 🚀 Getting Started

1. Clone or download this repository.
2. Open the `.ino` file in Arduino IDE.
3. Set your WiFi credentials and static IP.
4. Wire up hardware per the pin table above.
5. **Add the voltage divider** on the ECHO pin (see above — mandatory).
6. Upload to your ESP32.
7. Open Serial Monitor at **115200 baud** to see connection status and debug logs.
8. Navigate to the robot's IP in a browser and press **START JOURNEY**.

---

## 🔩 Wiring Overview

```
                    ┌─────────────┐
       Motors ◄─────┤    L298N    ├────► External Battery (7-12V)
                    │             │
  ESP32 GPIO ───────┤ ENA,IN1-IN4 │
  GPIO 25,13        │    ENB      │
                    └─────────────┘

  IR Sensors ──────► GPIO 34, 35, 32 (3.3V logic, direct connection OK)

  HC-SR04 TRIG ────► GPIO 23 (3.3V output from ESP32 is sufficient)
  HC-SR04 ECHO ────► 1kΩ ──┬── GPIO 22
                            │
                           2kΩ
                            │
                           GND
  HC-SR04 VCC  ────► 5V external supply
```

---

## 📄 License

MIT License — free to use, modify, and distribute.

---

## 🙌 Contributing

Pull requests welcome! If you improve the line-following algorithm, add PID control, or extend the web UI, please open a PR.
