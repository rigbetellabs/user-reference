# User Guide — RBL Standard Firmware
### Rigbetel Labs Research Platforms: Diadem · Cepheus · Acrux
**Firmware Version: 2.02**

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Overview](#2-system-overview)
3. [Powering On & Boot Sequence](#3-powering-on--boot-sequence)
4. [LCD Display Reference](#4-lcd-display-reference)
5. [LED Indicator Reference](#5-led-indicator-reference)
6. [Buzzer Reference](#6-buzzer-reference)
7. [Connecting to ROS 2](#7-connecting-to-ros-2)
8. [ROS Topic Reference](#8-ros-topic-reference)
9. [Velocity Control](#9-velocity-control)
10. [Motor & PID Control](#10-motor--pid-control)
11. [Wheel Encoders](#11-wheel-encoders)
12. [IMU Sensor](#12-imu-sensor)
13. [Battery Management](#13-battery-management)
14. [Navigation States](#14-navigation-states)
15. [Emergency Stop](#15-emergency-stop)
16. [RC Control (Optional)](#16-rc-control-optional)
17. [Network Status Display](#17-network-status-display)
18. [Domain ID & Networking](#18-domain-id--networking)
19. [ROS Namespace Configuration](#19-ros-namespace-configuration)
20. [Safety Warnings](#20-safety-warnings)
21. [Troubleshooting](#21-troubleshooting)

---

## 1. Introduction

The **RBL Standard Firmware** is the embedded software that runs on all Rigbetel Labs research robots — **Diadem**, **Cepheus**, and **Acrux**. It runs on an **ESP32 microcontroller** using the **micro-ROS** framework, which allows the robot to communicate directly with a standard ROS 2 network over USB serial.

This guide covers everything you need to operate your robot: from first power-on, through velocity control and PID tuning, to configuration of the namespace and domain ID.

Throughout this guide, ROS topic examples use the `/acrux` namespace. Replace `acrux` with your robot's namespace (`diadem`, `cepheus`, or any custom name you have configured).

> **Cross-reference:** For quick answers to common questions, see the [FAQ](FAQ.md).

---

## 2. System Overview

Understanding how the robot communicates with your computer helps you get it running quickly and debug problems when they arise.

### 2.1 Communication Stack

```
┌──────────────────────────────────────────────────────┐
│                   Your ROS 2 System                  │
│         (navigation, perception, planning, etc.)     │
└────────────────────────┬─────────────────────────────┘
                         │ ROS 2 topics
                         │
┌────────────────────────▼─────────────────────────────┐
│              micro-ROS Agent (on your PC)            │
│          ros2 run micro_ros_agent micro_ros_agent ... │
└────────────────────────┬─────────────────────────────┘
                         │ Serial (USB)
                         │
┌────────────────────────▼─────────────────────────────┐
│            ESP32 Microcontroller (on robot)          │
│               RBL Standard Firmware                  │
│  • Motor control with encoder feedback               │
│  • IMU data publishing (older versions only)         │
│  • Battery monitoring                                │
│  • LED animations & status indicators                │
│  • LCD display                                       │
│  • Buzzer feedback                                   │
│  • RC receiver input (if equipped)                   │
│  • micro-ROS communication                           │
└──────────────────────────────────────────────────────┘
```

The **micro-ROS agent** is a bridge process that runs on your computer. The robot connects to it automatically on startup. Once connected, all ROS 2 topics become available on your network.

> **Note:** IMU data publishing via the microcontroller applies to **older robot versions only**. On newer versions, the IMU is connected directly to the companion PC via USB for better performance and is not part of the firmware communication stack above.

### 2.2 Key Design Principle

Motor control, encoder reading, and battery monitoring run continuously on the robot — **they do not depend on the ROS 2 connection being active**. If the connection drops, motors stop safely and the robot waits to reconnect on its own.

---

## 3. Powering On & Boot Sequence

When you turn on the robot, it goes through a fixed startup sequence. Knowing this sequence lets you confirm that everything is working correctly before you start driving.

### Step-by-Step Boot Sequence

| Step | What You See / Hear |
|---|---|
| 1 | All LEDs light up **solid purple** briefly |
| 2 | LCD displays **"Rigbetel Labs"** on the top row and the robot model name (e.g., `Acrux`) on the bottom row |
| 3 | **Rigbetel Classic tone** plays from the buzzer — 7 beeps in a signature rhythm |
| 4 | LCD transitions to **"Booting..."** |
| 5 | A short initialization period runs (~15 seconds) |
| 6 | LEDs switch to a **pulsing orange breathing animation** — the robot is now searching for a micro-ROS agent |

> Once you see the pulsing orange LEDs, the robot is fully booted and ready for you to start the micro-ROS agent on your computer. See [Section 7](#7-connecting-to-ros-2).

---

## 4. LCD Display Reference

The robot uses a **16×2 I2C LCD display** that shows live status at all times. It works without any ROS 2 connection.

### 4.1 Layout

```
┌──────────────────┐
│  Primary message │   ← Row 0 (top): Status, navigation messages, SSID, IP
├──────────────────┤
│▓ 85% 22.4V ⚠ 025│   ← Row 1 (bottom): Battery, voltage, emergency icon, domain ID
└──────────────────┘
```

### 4.2 Top Row — Primary Messages

The top row shows a context-sensitive message depending on what the robot is doing:

| Robot State | Top Row Content |
|---|---|
| Searching for ROS agent | `"No Input / Retrying..."` |
| Connected — idle | Alternates every 3 seconds between WiFi SSID and IP address |
| RC mode active (no ROS) | `"RC Connected"` |
| Moving to goal | `"Way to goal"` |
| Goal reached | `"Reached Goal"` |
| Goal cancelled | `"Goal Cancelled"` |
| Costmap cleared | `"Costmap Cleared"` |
| Goal stored | `"Goal Stored"` |

Navigation messages are displayed for **3 seconds**, then the top row reverts to its previous idle content automatically.

### 4.3 Bottom Row — Status Information

| Position | Content | Example |
|---|---|---|
| 0 | Battery icon (fills with charge level) | `▓` |
| 1–3 | Battery percentage | `85%` |
| 6–10 | Battery voltage | `22.4V` |
| 12 | Emergency icon — visible only when e-stop is active | `⚠` |
| 13–15 | Current ROS 2 domain ID (3 digits) | `025`, `  0` |

### 4.4 Battery Icon

The battery icon changes in 8 increments to visually represent the current charge level — useful for a quick glance without reading numbers.

> **FAQ Cross-reference:** [Q6–Q10 — Display & Status Indicators](FAQ.md#2-display--status-indicators)

---

## 5. LED Indicator Reference

The robot has **50 RGB LEDs** arranged in segments: headlights, tail/brake lights, side strips, and status indicators at each corner. Each group conveys different information.

### 5.1 Global LED States

These states take priority over all other LED behavior:

| LED State | Meaning |
|---|---|
| All LEDs **solid purple** | Boot initialization in progress |
| All LEDs **pulsing orange** (breathing effect) | Waiting for micro-ROS agent — not yet connected |
| All LEDs **solid red** | Emergency stop is active |

### 5.2 Normal Runtime Colors (Connected, No Emergency)

Once the robot is connected and operational:

| LED Segment | Color | Meaning |
|---|---|---|
| Headlights (front) | White | Robot is on and connected |
| Tail / brake lights (rear) | Red | Indicates rear of robot |
| Side strips | Blue | Idle and ready |
| Status indicators (corners) | Blue | Idle and ready |

### 5.3 Navigation Event Colors

When your navigation system reports a state change via `/nav_status`, the **status indicator LEDs** at the corners change temporarily (3 seconds, then revert):

| Navigation Event | LED Color | Pattern |
|---|---|---|
| Moving to goal | Yellow | Solid |
| Goal reached | Green | Blinks 3× |
| Goal cancelled | Red | Blinks 1× |
| Costmap cleared | Orange | Blinks |
| Goal stored | Purple | Blinks |

### 5.4 Turn Indicator LEDs

The firmware automatically detects motion direction from `cmd_vel` and controls turn indicators — no extra topic needed:

| Motion | Indicator Behavior |
|---|---|
| Turning left | Left-side LEDs blink **orange** |
| Turning right | Right-side LEDs blink **orange** |
| Reversing | Rear brake lights blink **yellow** |
| Stationary or moving straight | No blinking |

> **Low battery override:** When battery drops below 15%, turn indicators switch to **red** as an additional alert.

### 5.5 RC Control Colors

When RC mode is active, the robot signals this clearly:

| LED Segment | Color |
|---|---|
| All status indicators | Solid blue |
| Headlights | White |
| Brake lights | Red |
| Turn indicators | Disabled |

> **FAQ Cross-reference:** [Q2 — What do the LED colors mean?](FAQ.md#1-general--getting-started)

---

## 6. Buzzer Reference

The buzzer provides audio feedback for all major system events. Patterns are queued — multiple events can stack without interfering with each other. If you hear no beeps at all, the buzzer may have been disabled via ROS (see [Section 8.1](#81-topics-you-can-publish-to-subscribers)).

### 6.1 Boot Tone

The **Rigbetel Classic Tone** plays once on every startup:

```
Beep — pause — Beep — Beep — Beep — pause — Beep · · · Beep — Beep — Beep
```

7 beeps total in a distinctive rhythmic signature.

### 6.2 Complete Buzzer Pattern Reference

#### micro-ROS Connection

| Pattern | Event |
|---|---|
| 2 beeps — 200 ms on / 100 ms off | micro-ROS agent **connected** |
| 4 rapid beeps — 100 ms on / 100 ms off | micro-ROS agent **disconnected** |

#### Navigation

| Pattern | Event |
|---|---|
| 1 beep — 500 ms | Robot is **moving to goal** |
| 3 beeps — 200 ms on / 200 ms off | **Goal reached** |
| 1 long beep — 1000 ms | **Goal cancelled** |
| 4 rapid beeps — 100 ms on / 100 ms off | **Costmap cleared** |
| 1 beep — 700 ms | **Goal stored** |

#### Battery

| Pattern | Frequency | Event |
|---|---|---|
| 5 rapid beeps — 100 ms on / 100 ms off | Every 15 seconds | **Battery below 15%** — recharge immediately |

#### PID Control

| Pattern | Event |
|---|---|
| 1 long beep — 1000 ms | **PID disabled** (open-loop mode activated) |
| 1 short beep — 200 ms | **Adaptive PID** mode activated |
| 2 short beeps | **Custom PID** mode activated |

#### Encoder Fault

| Pattern | Event |
|---|---|
| 1 beep — 500 ms, repeating | **Encoder fault detected** — stop and inspect the robot |

> **FAQ Cross-reference:** [Q45 — Complete buzzer patterns](FAQ.md#10-buzzer-indications)

### 6.3 Enabling / Disabling the Buzzer

```bash
# Disable buzzer
ros2 topic pub -1 /acrux/buzzer/enable std_msgs/msg/Bool "{data: false}"

# Re-enable buzzer
ros2 topic pub -1 /acrux/buzzer/enable std_msgs/msg/Bool "{data: true}"
```

> **Note:** The buzzer state **resets to enabled on every reboot**. Disabling is a runtime-only setting.

> **WARNING:** Disabling the buzzer is **not recommended**. The buzzer is the primary alert mechanism for critical events — low battery, encoder faults, and connection changes. Disabling it means you will miss these alerts, which can result in battery damage, unexpected robot behavior, or safety hazards.

---

## 7. Connecting to ROS 2

The robot communicates via **micro-ROS**. You need to run the **micro-ROS agent** on your computer for the robot's topics to appear on ROS 2.

### 7.1 Prerequisites

- **ROS 2 Humble** or later installed on your computer.
- The `micro_ros_agent` package:
  ```bash
  sudo apt install ros-humble-micro-ros-agent
  ```

### 7.2 Starting the micro-ROS Agent

The robot connects to the micro-ROS agent via **USB serial**. Connect the robot to your computer using a USB cable, then start the agent:

```bash
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyUSB0 --baudrate 921600
```

Replace `/dev/ttyUSB0` with your robot's actual serial port (e.g., `/dev/ttyUSB1`, `/dev/ttyACM0`). The baud rate must be **921600**.

> **Tip:** To find the correct port, run `ls /dev/tty*` before and after plugging in the robot and compare the output.

### 7.3 What Happens After Connection

Once the agent is detected (within ~500 ms):

1. **2 short beeps** confirm the connection.
2. LEDs switch from pulsing orange → normal runtime colors (white headlights, blue sides, red tail).
3. All ROS topics become available on your network.
4. The display top row begins cycling between SSID and IP address (after you publish network status — see [Section 17](#17-network-status-display)).

### 7.4 Verify the Connection

```bash
# Should show all /acrux/... topics
ros2 topic list

# Check firmware version
ros2 param get /acrux/microros_node version
```

### 7.5 Connection State Machine

The robot continuously manages the agent connection. Understanding these states helps diagnose issues:

```
WAITING_AGENT ──(agent found)──► AGENT_AVAILABLE ──(ready)──► AGENT_CONNECTED
      ▲                                                               │
      │                                                        (connection lost)
      │                                                               ▼
      └───────────────────── AGENT_DISCONNECTED ◄────────────────────
```

| State | LED | Behavior |
|---|---|---|
| `WAITING_AGENT` | Pulsing orange | Searching for agent every 500 ms |
| `AGENT_AVAILABLE` | — | Setting up ROS publishers, subscribers, and parameters |
| `AGENT_CONNECTED` | Normal runtime | Fully operational |
| `AGENT_DISCONNECTED` | Pulsing orange | Motors stopped; 4-beep alert; auto-reconnecting |

> **FAQ Cross-reference:** [Q3, Q5, Q44, Q46, Q47, Q56](FAQ.md)

---

## 8. ROS Topic Reference

All topics are prefixed with the robot's namespace. This section uses `/acrux` as the example — replace it with your robot's namespace.

### 8.1 Topics You Can Publish TO (Subscribers)

These are topics the robot **listens to**. Send messages here from your terminal or software.

---

#### `/acrux/cmd_vel` — Velocity Command

- **Type:** `geometry_msgs/Twist`
- **Description:** The primary motion control topic. Sends velocity commands to drive the robot. The robot applies closed-loop PID control to match the commanded speed on each wheel.

  | Field | Unit | Description |
  |---|---|---|
  | `linear.x` | m/s | Forward (+) / backward (−) speed |
  | `angular.z` | rad/s | Counter-clockwise (+) / clockwise (−) rotation |
  | `linear.y` | m/s | Lateral movement (Cepheus mecanum configuration only) |

- **Example — move forward at 0.3 m/s:**
  ```bash
  ros2 topic pub /acrux/cmd_vel geometry_msgs/msg/Twist \
    "{linear: {x: 0.3, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
  ```

- **Example — stop:**
  ```bash
  ros2 topic pub /acrux/cmd_vel geometry_msgs/msg/Twist \
    "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
  ```

> **Important:** The robot stops automatically if the micro-ROS agent disconnects while it is moving.

---

#### `/acrux/nav_status` — Navigation State

- **Type:** `std_msgs/Int32`
- **Description:** Tells the robot's display, LEDs, and buzzer what your navigation system is doing. This topic controls **feedback only** — it does not move the robot.

  | Value | State | Display | LEDs | Buzzer |
  |---|---|---|---|---|
  | `0` | Idle | SSID / IP | Normal colors | — |
  | `1` | Moving to goal | `"Way to goal"` | Yellow | 1 beep (500 ms) |
  | `2` | Goal reached | `"Reached Goal"` | Green blink 3× | 3 beeps |
  | `3` | Goal cancelled | `"Goal Cancelled"` | Red blink | 1 long beep |
  | `4` | Costmap cleared | `"Costmap Cleared"` | Orange blink | 4 beeps |
  | `5` | Goal stored | `"Goal Stored"` | Purple blink | 1 beep |

- **Example:**
  ```bash
  ros2 topic pub -1 /acrux/nav_status std_msgs/msg/Int32 "{data: 1}"
  ```

> After states 2–5, the display and LEDs automatically revert to idle after **3 seconds**.

---

#### `/acrux/pid/mode` — PID Mode Selection

- **Type:** `std_msgs/Int8`
- **Description:** Switches the motor controller between three operating modes.

  | Value | Mode | When to use |
  |---|---|---|
  | `0` | PID Off | Bench testing only — no encoder feedback, robot will drift |
  | `1` | Adaptive (default) | All normal use — factory-tuned for your robot model |
  | `2` | Custom | When you have saved your own tuned PID values |

- **Example:**
  ```bash
  ros2 topic pub -1 /acrux/pid/mode std_msgs/msg/Int8 "{data: 1}"
  ```

---

#### `/acrux/pid/constants` — PID Gains

- **Type:** `std_msgs/Float32MultiArray`
- **Description:** Updates the PID gains immediately. Array format: `[Kp, Ki, Kd]`. Applies to all wheels simultaneously.
- **Example:**
  ```bash
  ros2 topic pub -1 /acrux/pid/constants std_msgs/msg/Float32MultiArray \
    "{data: [0.5, 40.0, 0.0]}"
  ```

> **Safety:** Always lift the robot off the ground before sending custom PID values. See [Section 20](#20-safety-warnings).

---

#### `/acrux/pid/custom/save` — Save Custom PID to Flash

- **Type:** `std_msgs/Empty`
- **Description:** Saves the current Kp, Ki, Kd values to non-volatile storage on the robot. They are automatically loaded on the next boot when Custom PID mode is selected.
- **Example:**
  ```bash
  ros2 topic pub -1 /acrux/pid/custom/save std_msgs/msg/Empty "{}"
  ```

---

#### `/acrux/wheel/ticks_reset` — Reset Encoder Counts

- **Type:** `std_msgs/Empty`
- **Description:** Resets the wheel encoder tick counters to zero. Use this before starting an odometry-tracked session.
- **Example:**
  ```bash
  ros2 topic pub -1 /acrux/wheel/ticks_reset std_msgs/msg/Empty "{}"
  ```

---

#### `/acrux/network_status` — Network Info for Display

- **Type:** `std_msgs/String`
- **Description:** Sends your companion PC's network information to the robot's LCD. Publish this **once** after each connection. The message is a JSON string.

  | JSON Field | Description |
  |---|---|
  | `"mode"` | Network type (e.g., `"WiFi"`) |
  | `"status"` | Connection status (e.g., `"Connected"`) |
  | `"info"` | WiFi SSID or network name |
  | `"ip"` | IP address |

- **Example:**
  ```bash
  ros2 topic pub -1 /acrux/network_status std_msgs/msg/String \
    "{data: '{\"mode\": \"WiFi\", \"status\": \"Connected\", \"info\": \"MyNetwork\", \"ip\": \"192.168.1.100\"}'}"
  ```

---

#### `/acrux/buzzer/enable` — Buzzer On/Off

- **Type:** `std_msgs/Bool`
- **Description:** Enables (`true`) or disables (`false`) the buzzer. Default is enabled. Resets to enabled on reboot.
- **Example:**
  ```bash
  ros2 topic pub -1 /acrux/buzzer/enable std_msgs/msg/Bool "{data: false}"
  ```

---

#### `/acrux/microros_domain_id` — Change Domain ID

- **Type:** `std_msgs/Int8`
- **Description:** Changes the robot's ROS 2 domain ID, saves it permanently, and **automatically reboots** the robot. Valid range: 0–101.
- **Example:**
  ```bash
  ros2 topic pub -1 /acrux/microros_domain_id std_msgs/msg/Int8 "{data: 25}"
  ```

---

#### `/acrux/microros_namespace` — Change Namespace

- **Type:** `std_msgs/String`
- **Description:** Changes the robot's ROS namespace, saves it permanently, and **automatically reboots**. All topics then appear under the new namespace.
- **Namespace rules:** Maximum 15 characters; lowercase letters and `/` only; no digits, uppercase, or special characters.
- **Example:**
  ```bash
  ros2 topic pub -1 /acrux/microros_namespace std_msgs/msg/String "{data: 'my_robot'}"
  ```

---

### 8.2 Topics the Robot Publishes

These topics carry live data from the robot. Subscribe to them from your software.

---

#### `/acrux/cmd_vel` — Active Velocity Echo

- **Type:** `geometry_msgs/Twist` | **Rate:** ~10 Hz
- **Description:** The robot echoes back the velocity command currently being executed. Use this to confirm commands are being received and acted upon.

---

#### `/acrux/wheel/rpm` — Wheel RPM

- **Type:** `std_msgs/Float32MultiArray` | **Rate:** ~10 Hz
- **Description:** Current RPM of each wheel, measured from encoders.
  - **2WD (Acrux, Cepheus 2WD):** `[left, right]`
  - **4WD (Diadem, Cepheus 4WD):** `[left_front, left_back, right_front, right_back]`

---

#### `/acrux/wheel/vel` — Wheel Velocity

- **Type:** `std_msgs/Float64MultiArray` | **Rate:** ~10 Hz
- **Description:** Current velocity of each wheel in m/s, derived from encoder readings. Same array order as `wheel/rpm`.

---

#### `/acrux/wheel/ticks` — Encoder Ticks

- **Type:** `std_msgs/Int64MultiArray` | **Rate:** ~500 Hz
- **Description:** Cumulative encoder tick count since last reset. Used for odometry. Same array order as above.

> **Note:** The robot publishes raw tick data. Your ROS stack is responsible for computing `/odom` and TF transforms from these ticks.

---

#### `/acrux/imu/raw_data` — IMU Data

- **Type:** `sensor_msgs/Imu` | **Rate:** ~500 Hz
- **Description:** Sensor data from the onboard IMU. Contains:
  - Orientation (quaternion): `orientation.x/y/z/w`
  - Angular velocity (rad/s): `angular_velocity.x/y/z`
  - Linear acceleration (m/s²): `linear_acceleration.x/y/z`
  - All timestamps are synchronized to ROS time

> **Note (older versions only):** On robots manufactured **before 2026**, the IMU is connected to the ESP32 and data is published over this topic. On **newer versions**, the IMU is connected directly to the companion PC via USB for better performance — this topic will not be active on those robots. Refer to your robot's hardware documentation.

---

#### `/acrux/battery/percentage` — Battery Percentage

- **Type:** `std_msgs/Int8` | **Rate:** ~1 Hz
- **Description:** Estimated battery charge (0–100%). Based on a linear voltage estimate.

> **Note:** This is a voltage-based approximation. It may be less accurate as the battery ages. For reliable monitoring, prefer watching the **voltage** topic directly. If your robot includes a smart BMS, use the BMS topics instead.

---

#### `/acrux/battery/voltage` — Battery Voltage

- **Type:** `std_msgs/Float32` | **Rate:** ~1 Hz
- **Description:** Real-time battery voltage in volts (accuracy ±0.2 V).

  | Voltage | State |
  |---|---|
  | 25.2 V | Fully charged |
  | 19.8 V | Minimum safe — stop and recharge now |
  | Below 19.8 V | Risk of permanent battery damage |

---

#### `/acrux/estop/status` — Emergency Stop Status

- **Type:** `std_msgs/Bool` | **Rate:** ~5 Hz
- **Description:** Current state of the physical emergency stop button.
  - `true` → E-stop is **active**; motors are halted.
  - `false` → Normal operation.

  ```bash
  ros2 topic echo /acrux/estop/status
  ```

---

### 8.3 ROS Parameter Server

| Parameter | Value | Description |
|---|---|---|
| `version` | `2.02` | Firmware version |

```bash
ros2 param get /acrux/microros_node version
```

---

## 9. Velocity Control

### 9.1 How cmd_vel Works

When you publish to `/acrux/cmd_vel`:

1. The robot receives the `(linear.x, angular.z)` command.
2. It converts these into individual per-wheel velocity targets:
   - Left wheel = `linear.x − (angular.z × wheel_separation / 2)`
   - Right wheel = `linear.x + (angular.z × wheel_separation / 2)`
3. Each wheel has its own independent closed-loop controller that matches the actual wheel speed to the target.

### 9.2 Velocity Limits

Each robot has a maximum speed based on its motors and wheel size. Commands beyond the maximum are automatically clamped — not rejected.

The values below are **standard defaults**. Actual maximum velocity may differ depending on the specific hardware configuration ordered.

| Platform | Standard Max Linear Velocity |
|---|---|
| Diadem | ~1.58 m/s |
| Cepheus | ~0.835 m/s |
| Acrux | ~0.628 m/s |

Start with low velocities (0.1–0.2 m/s) and increase gradually, especially in new environments.

### 9.3 RC Mode Priority

If your robot is equipped with an RC receiver and RC mode is active, all `cmd_vel` messages from ROS 2 are **ignored** until RC mode is disengaged. This is intentional for safety. See [Section 16](#16-rc-control-optional).

> **FAQ Cross-reference:** [Q17, Q18 — Velocity commands](FAQ.md#4-ros-topics--communication)

---

## 10. Motor & PID Control

### 10.1 Why PID Matters

Without closed-loop feedback, small mechanical differences between motors cause the robot to drift even when commanded to go straight. The firmware runs an **independent controller per wheel** that continuously adjusts motor power to keep each wheel at the commanded speed.

### 10.2 PID Modes

#### Adaptive Mode — Recommended

Factory-tuned values optimized for your specific robot model. Use this for all normal operation, including autonomous navigation.

```bash
ros2 topic pub -1 /acrux/pid/mode std_msgs/msg/Int8 "{data: 1}"
```

Confirmed by: **1 short beep**

#### Custom Mode

Use your own Kp, Ki, Kd values. Only use this mode if you have carefully tuned and tested your values with the robot lifted off the ground.

```bash
# Switch to Custom mode
ros2 topic pub -1 /acrux/pid/mode std_msgs/msg/Int8 "{data: 2}"

# Send your constants
ros2 topic pub -1 /acrux/pid/constants std_msgs/msg/Float32MultiArray \
  "{data: [0.5, 40.0, 0.0]}"
```

Confirmed by: **2 short beeps**

#### PID Off Mode

Disables feedback control. Motors respond to commands without any encoder correction. **For bench testing only** — the robot will drift noticeably in real use.

```bash
ros2 topic pub -1 /acrux/pid/mode std_msgs/msg/Int8 "{data: 0}"
```

Confirmed by: **1 long beep (1000 ms)**

### 10.3 Saving Custom Values

After tuning, save your values so they persist across reboots:

```bash
# Save current custom values to flash
ros2 topic pub -1 /acrux/pid/custom/save std_msgs/msg/Empty "{}"

# On next boot, select Custom mode to load them
ros2 topic pub -1 /acrux/pid/mode std_msgs/msg/Int8 "{data: 2}"
```

### 10.4 Encoder Health Check

The firmware periodically verifies that each encoder is responding correctly during motion. If an encoder is not detected as working:

- All motors halt immediately.
- A **500 ms single beep repeats** on every health check until the robot is rebooted.
- The robot will not accept motion commands until the encoder is repaired and the robot is restarted.

If you hear repeated single medium-length beeps during operation, stop immediately and inspect the wheel encoders and their connections.

> **FAQ Cross-reference:** [Q31–Q35 — PID Control](FAQ.md#7-pid-control)

---

## 11. Wheel Encoders

### 11.1 Reading Encoder Data

Three topics give you full wheel motion information:

```bash
# Cumulative tick counts (highest rate, ~500 Hz)
ros2 topic echo /acrux/wheel/ticks

# Wheel velocities in m/s (~10 Hz)
ros2 topic echo /acrux/wheel/vel

# Wheel RPM (~10 Hz)
ros2 topic echo /acrux/wheel/rpm
```

Array order:
- **2WD (Acrux, Cepheus 2WD):** `[left, right]`
- **4WD (Diadem, Cepheus 4WD):** `[left_front, left_back, right_front, right_back]`

### 11.2 Resetting Tick Counts

```bash
ros2 topic pub -1 /acrux/wheel/ticks_reset std_msgs/msg/Empty "{}"
```

### 11.3 Using Ticks for Odometry

The firmware publishes raw tick data — it does **not** compute or publish `/odom` or TF transforms. Your ROS stack must integrate tick counts using the robot's wheel diameter and encoder resolution to compute position. A standard differential drive odometry node (e.g., from `nav2_bringup` or `robot_localization`) handles this.

> **FAQ Cross-reference:** [Q19, Q50 — Encoder topics and reset](FAQ.md#4-ros-topics--communication)

---

## 12. IMU Sensor

> **Note (older versions only):** This section applies to robots manufactured **before 2026**, where the IMU is connected to the ESP32 microcontroller. On **newer versions**, the IMU is connected directly to the companion PC via USB for better performance and is published by a separate driver — not through this micro-ROS topic. Refer to your robot's hardware documentation to confirm which applies to your robot.

### 12.1 Reading IMU Data

```bash
ros2 topic echo /acrux/imu/raw_data
```

The message (`sensor_msgs/Imu`) contains:

| Field | Unit | Description |
|---|---|---|
| `orientation` | quaternion | Absolute orientation with onboard sensor fusion |
| `angular_velocity.x/y/z` | rad/s | Gyroscope data |
| `linear_acceleration.x/y/z` | m/s² | Accelerometer data |

### 12.2 Timestamps

All IMU messages carry **timestamps synchronized to ROS time**, which is essential for TF transforms and sensor fusion in your navigation stack.

> **Note (older versions only):** On robots manufactured **2026 or later**, the IMU is connected directly to the companion PC via USB. This micro-ROS topic will not be active on those robots.

> **FAQ Cross-reference:** [Q20 — IMU data](FAQ.md#4-ros-topics--communication)

---

## 13. Battery Management

### 13.1 Monitoring Battery Status

The robot continuously monitors battery voltage and publishes it at ~1 Hz:

```bash
ros2 topic echo /acrux/battery/voltage
ros2 topic echo /acrux/battery/percentage
```

The LCD bottom row shows both the battery percentage and live voltage at all times, even without ROS connected.

### 13.2 Voltage Reference

| Voltage | State |
|---|---|
| 25.2 V | Fully charged |
| ~22 V | Mid charge |
| 19.8 V | Minimum safe level — stop and recharge |
| Below 19.8 V | Risk of permanent battery cell damage |

> Note: Battery percentage is a linear voltage estimate. It may become less accurate as the battery ages. Monitor voltage directly for reliable readings.

### 13.3 Low Battery Alert

When battery drops **below 15%**:

- **5 rapid beeps** sound every 15 seconds.
- Turn indicator LEDs switch to **red**.
- The display continues to show percentage and voltage.
- The robot remains operational — it will not auto-shutdown.

**Stop operation and recharge the robot when this alert triggers.** Do not drain below 10% — doing so can permanently damage the battery cells.

> **FAQ Cross-reference:** [Q11–Q14 — Battery](FAQ.md#3-battery)

---

## 14. Navigation States

### 14.1 Overview

The firmware does not perform path planning — that is your ROS 2 navigation stack's job (e.g., Nav2). The firmware's role is to **display and announce** navigation events through the LCD, LEDs, and buzzer, giving clear feedback to anyone observing the robot.

### 14.2 Publishing Navigation States

Publish an integer to `/acrux/nav_status` from your navigation node:

```bash
ros2 topic pub -1 /acrux/nav_status std_msgs/msg/Int32 "{data: 1}"
```

### 14.3 Each State Explained

#### State 0 — Idle
The default state after connection. The display cycles between SSID and IP address. LEDs show normal runtime colors. No buzzer.

#### State 1 — Moving to Goal
Display: `"Way to goal"` · LEDs: yellow · Buzzer: 1 beep (500 ms).
Publish this when your navigation stack begins driving toward a goal.

#### State 2 — Goal Reached
Display: `"Reached Goal"` · LEDs: green blink 3× · Buzzer: 3 beeps.
Publish this when the robot arrives at its destination. Returns to idle after 3 seconds.

#### State 3 — Goal Cancelled
Display: `"Goal Cancelled"` · LEDs: red blink · Buzzer: 1 long beep.
Publish this when a goal is aborted. Check your navigation system for the reason. Returns to idle after 3 seconds.

#### State 4 — Costmap Cleared
Display: `"Costmap Cleared"` · LEDs: orange blink · Buzzer: 4 beeps.
Publish this when your navigation stack clears a costmap to re-plan. Motion continues uninterrupted. Returns to previous state after 3 seconds.

#### State 5 — Goal Stored
Display: `"Goal Stored"` · LEDs: purple blink · Buzzer: 1 beep.
Publish this when a goal location has been saved for later. Returns to idle after 3 seconds.

### 14.4 When Navigation Feedback Is Active

Navigation state updates are processed **only when**:
- The micro-ROS agent is connected
- The emergency stop is released
- RC control mode is not active

> **FAQ Cross-reference:** [Q22–Q26 — Navigation & States](FAQ.md#5-navigation--states)

---

## 15. Emergency Stop

### 15.1 How It Works

The physical **emergency stop button** on the robot is wired directly to the microcontroller and is always active — even without a ROS 2 connection. It cannot be disabled via software.

### 15.2 When the E-Stop Is Pressed

1. All motors halt immediately.
2. All LEDs turn **solid red**.
3. An emergency icon (⚠) appears on the LCD bottom row.
4. The robot ignores all velocity commands.
5. `/acrux/estop/status` publishes `true`.

### 15.3 Recovery

Release the button. The robot detects the release and:
1. Removes the emergency icon from the LCD.
2. Returns LEDs to normal runtime colors.
3. Accepts `cmd_vel` commands again.

> The robot does **not** resume motion automatically on release. It waits for a new velocity command from ROS 2.

### 15.4 Monitoring via ROS 2

```bash
ros2 topic echo /acrux/estop/status
```

Integrate this into your safety-critical nodes to detect e-stop activation and respond (pause goal execution, alert the operator, log an event).

> **FAQ Cross-reference:** [Q27–Q30 — Emergency Stop](FAQ.md#6-emergency-stop)

---

## 16. RC Control (Optional)

### 16.1 Applicability

RC control is **only available on robots ordered with an RC receiver**. If your robot was not configured with RC support, this entire section does not apply.

### 16.2 Channel Mapping

| Channel | Function | Range |
|---|---|---|
| Channel 1 | Steering / angular velocity | 200 – 1800 |
| Channel 3 | Throttle / linear velocity | 200 – 1800 |
| Channel 5 | RC / ROS mode switch | `1800` = RC mode active; other = ROS mode |
| Channel 7 | PID mode selector | `200`=Off · `1000`=Adaptive · `1800`=Custom |
| Channel 8 | Speed throttle scaler | 200–1800 → 20%–100% of max speed |

### 16.3 Switching Between RC and ROS Mode

Flip **Channel 5** to position `1800` on the transmitter to enter RC mode. Flip back to return to ROS mode.

| Mode | Status LED Color | ROS `cmd_vel` |
|---|---|---|
| RC mode active | Solid **blue** | **Ignored** |
| ROS mode | Normal runtime colors | **Active** |

### 16.4 Input Smoothing

RC input is automatically processed before reaching the motors:
- A rolling average smooths rapid stick movements.
- A center deadband prevents drift when sticks are at rest.
- The throttle channel scales final speed between 20% and 100% of maximum.

### 16.5 Changing PID Mode via RC

Flip **Channel 7** to switch PID mode without needing ROS 2:
- Position `200` → PID Off (1 long beep)
- Position `1000` → Adaptive PID (1 short beep)
- Position `1800` → Custom PID (2 short beeps)

Motors stop briefly during mode transitions before resuming.

> **FAQ Cross-reference:** [Q36–Q40 — RC Control](FAQ.md#8-rc-control)

---

## 17. Network Status Display

By default, after connecting to ROS, the LCD top row shows `"SSID: XX"` and `"IP Address: XX"` — placeholder text, because the firmware does not know your PC's network details.

To show the actual SSID and IP address, publish this **once** after each connection:

```bash
ros2 topic pub -1 /acrux/network_status std_msgs/msg/String \
  "{data: '{\"mode\": \"WiFi\", \"status\": \"Connected\", \"info\": \"YourSSID\", \"ip\": \"192.168.1.100\"}'}"
```

After this, the display cycles between the SSID and IP every 3 seconds while idle. You only need to republish if the robot reboots or reconnects.

> **FAQ Cross-reference:** [Q21 — Network status display](FAQ.md#4-ros-topics--communication)

---

## 18. Domain ID & Networking

### 18.1 What is the ROS 2 Domain ID?

The Domain ID is a number (0–101) that **isolates ROS 2 networks**. Only devices with the same domain ID can see each other's topics — think of it as a channel number. The robot's current domain ID is always shown in the **bottom-right corner of the LCD** (e.g., `025`, `  0`).

### 18.2 Matching the Domain ID on Your Computer

```bash
export ROS_DOMAIN_ID=25
```

Add this to `~/.bashrc` to make it permanent, or set it in your ROS 2 launch files.

### 18.3 Changing the Domain ID

```bash
ros2 topic pub -1 /acrux/microros_domain_id std_msgs/msg/Int8 "{data: 25}"
```

The robot saves the new ID and **automatically reboots**. After reboot:
1. Confirm the new ID appears on the LCD.
2. Update `ROS_DOMAIN_ID` on your computer.
3. Restart the micro-ROS agent.

### 18.4 Persistence

The domain ID is stored permanently and survives all reboots and power cycles. The factory default is `0`.

> **FAQ Cross-reference:** [Q41–Q44 — Domain ID & Networking](FAQ.md#9-domain-id--networking)

---

## 19. ROS Namespace Configuration

### 19.1 What is the Namespace?

Every ROS topic from this robot is prefixed with a namespace (e.g., `/acrux/cmd_vel`). The namespace uniquely identifies your robot on the network and allows multiple robots to run simultaneously without topic collisions.

| Robot | Default Namespace |
|---|---|
| Diadem | `/diadem` |
| Cepheus | `/cepheus` |
| Acrux | `/acrux` |

### 19.2 Changing the Namespace

```bash
ros2 topic pub -1 /acrux/microros_namespace std_msgs/msg/String "{data: 'lab_robot_one'}"
```

The robot validates the name, saves it, and **automatically reboots**. All topics then appear as `/lab_robot_one/...`.

### 19.3 Namespace Rules

| Rule | Detail |
|---|---|
| Maximum length | 15 characters |
| Allowed characters | Lowercase `a–z` and `/` only |
| Invalid characters | Digits, uppercase, spaces, symbols — silently stripped |
| Persists across reboots | Yes |

> **FAQ Cross-reference:** [Q15, Q54 — Namespace](FAQ.md#4-ros-topics--communication)

---

## 20. Safety Warnings

Read and follow these warnings before operating or configuring the robot.

---

### Battery Safety

> **WARNING:** Do not allow the battery to drain below **10%** (approximately 20.5 V). Discharging lithium cells below their minimum voltage causes **permanent, irreversible damage** to the battery pack. The firmware alerts you at 15% — this is your signal to stop and recharge.

> **WARNING:** Do not charge the battery while the robot is operating autonomously and unattended.

---

### PID Tuning Safety

> **WARNING:** Before publishing custom PID values or switching to PID Off mode, **physically lift the robot so its wheels are completely off the ground**. Incorrect values can cause sudden, uncontrolled wheel spin at full speed, which can launch the robot across the floor, damage gearboxes, or injure people nearby.

> **NOTICE:** Any damage to gearboxes, motors, or wheels from improper PID tuning is **not covered under warranty**.

---

### Emergency Stop

> **WARNING:** The physical emergency stop button is your **primary safety mechanism**. Keep it accessible at all times when the robot is operating near people. Never block or disable it.

---

### Velocity Commands

> **WARNING:** Always verify your `cmd_vel` publisher before running in open space. Start at low velocities (0.1–0.2 m/s) and increase gradually. An errant high-speed command can cause collisions or injury.

---

### Encoder Fault

> **NOTICE:** Repeated single medium-length beeps during operation indicate an **encoder fault**. The robot will halt and refuse motion commands. Stop operation immediately, inspect the wheel encoders and wiring, then reboot the robot before resuming.

---

## 21. Troubleshooting

### The robot powers on but no ROS topics appear

1. Confirm the micro-ROS agent is running on your computer.
2. Check that `ROS_DOMAIN_ID` on your computer matches the value shown on the robot's LCD.
3. Verify the USB cable is properly connected and the correct serial port is specified.
4. Observe the LEDs — **pulsing orange** means the robot is still searching for the agent.
5. Restart the micro-ROS agent.

```bash
ros2 topic list   # Should show /acrux/... topics
```

---

### LEDs are pulsing orange

The robot is waiting for the micro-ROS agent. Start or restart the agent on your computer. See [Section 7](#7-connecting-to-ros-2).

---

### All LEDs are solid red

The **emergency stop is active**. Release the physical e-stop button. LEDs and motors return to normal automatically.

---

### The robot drifts or moves erratically

1. Confirm PID mode is **Adaptive**:
   ```bash
   ros2 topic pub -1 /acrux/pid/mode std_msgs/msg/Int8 "{data: 1}"
   ```
2. Listen for repeated medium-length beeps — these indicate an encoder fault. Stop and inspect encoders.
3. Check battery voltage. Low voltage degrades motor performance.

---

### The buzzer is beeping 5 times every 15 seconds

Battery is **below 15%**. Stop operation and recharge the robot immediately.

---

### I changed the Domain ID but topics are not visible

1. Confirm the LCD bottom-right shows the new domain ID.
2. Update your terminal:
   ```bash
   export ROS_DOMAIN_ID=25
   ```
3. Restart the micro-ROS agent.
4. Restart any ROS 2 nodes (they must pick up the new domain ID).
5. Run `ros2 topic list` to verify.

---

### The display shows "SSID: XX" / "IP Address: XX"

Publish the network status once after connecting:
```bash
ros2 topic pub -1 /acrux/network_status std_msgs/msg/String \
  "{data: '{\"mode\": \"WiFi\", \"status\": \"Connected\", \"info\": \"MySSID\", \"ip\": \"192.168.1.100\"}'}"
```

---

### The robot does not respond to cmd_vel

Check these conditions in order:

1. Are LEDs pulsing orange? → micro-ROS agent not connected.
2. Does `/acrux/estop/status` return `true`? → Release the physical e-stop.
3. Are status LEDs solid blue? → RC mode is active; ROS `cmd_vel` is ignored.
4. Do you hear repeated single beeps? → Encoder fault; stop and inspect.

---

### The display is blank or not turning on

Check the physical cable and connector to the LCD display. If connections are secure and the issue persists, contact **Rigbetel Labs** for support.

> **FAQ Cross-reference:** [Q46–Q53 — Troubleshooting](FAQ.md#11-troubleshooting)

---

*For further assistance, contact Rigbetel Labs or refer to the [FAQ](FAQ.md) for quick answers to common questions.*
