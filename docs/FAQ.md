# Frequently Asked Questions (FAQ)
### RigBetel Labs Standard Firmware — Diadem / Cepheus / Acrux / TortoiseBot Pro Max

---

## Table of Contents

1. [General / Getting Started](#1-general--getting-started)
2. [Display & Status Indicators](#2-display--status-indicators)
3. [Battery](#3-battery)
4. [ROS Topics & Communication](#4-ros-topics--communication)
5. [Navigation & States](#5-navigation--states)
6. [Emergency Stop](#6-emergency-stop)
7. [PID Control](#7-pid-control)
8. [RC Control](#8-rc-control)
9. [Domain ID & Networking](#9-domain-id--networking)
10. [Buzzer Indications](#10-buzzer-indications)
11. [Troubleshooting](#11-troubleshooting)
12. [Advanced / Developer](#12-advanced--developer)

---

## 1. General / Getting Started

**Q1. How do I power on the robot and what should I see on startup?**

Power on the robot using the main power switch. On startup you will:
1. See the LCD display show **"RigBetel Labs"** on the top row and the robot's model name (e.g., `Acrux`) on the bottom row.
2. Hear the **Rigbetel Classic boot tone** from the buzzer (see [Buzzer Indications](#10-buzzer-indications)).
3. See all LEDs light up in **Rigbetel purple** for a brief moment.
4. The LEDs will then transition to an **orange pulsing animation**, indicating the robot is searching for a micro-ROS agent to connect to.

---

**Q2. What do the LED colors mean when the robot first boots?**

| LED State | Meaning |
|---|---|
| All LEDs solid **purple** | Boot initialization in progress |
| All LEDs **pulsing orange** (fade in/out) | Waiting for micro-ROS agent connection |
| Normal runtime pattern (white headlights, red tail, blue side strips) | Connected and idle |
| All LEDs **solid red** | Emergency stop is active |
| Status LEDs **blue** | RC control mode is active |

---

**Q3. What is micro-ROS and do I need ROS 2 installed on my computer?**

micro-ROS is a framework that runs ROS 2 directly on the robot's microcontroller (ESP32). To communicate with the robot, you need:
- **ROS 2** installed on your computer (Humble or later recommended).
- The **micro-ROS agent** running on your computer or an onboard companion computer. The agent acts as a bridge between the robot's firmware and your ROS 2 network.

Start the agent with:
```bash
ros2 run micro_ros_agent micro_ros_agent udp4 --port 8888
```
or over serial:
```bash
ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyUSB0
```

---

**Q4. Which ROS 2 version is supported?**

ROS 2 **Humble Hawksbill** or later is recommended. The firmware uses standard micro-ROS Arduino libraries and standard ROS 2 message types, so it is compatible with any ROS 2 distribution that supports micro-ROS.

---

**Q5. How do I connect my computer to the robot?**

The robot communicates via micro-ROS over **UDP (WiFi/Ethernet)** or **Serial (USB)**. Ensure your computer and the robot are on the same network (for UDP), then launch the micro-ROS agent as shown in Q3. Once the agent is running, the robot will auto-connect and all ROS 2 topics will become available.

---

## 2. Display & Status Indicators

**Q6. What information is shown on the LCD display?**

The robot uses a **16×2 I2C LCD**. The layout is:

```
[ Top Row    - Primary segment: status messages, SSID, IP, navigation state ]
[ Bot Row    - [BatIcon][Bat%] [Voltage] [EmgIcon] [DomainID] ]
```

- **Top row** cycles between messages when idle (e.g., SSID and IP address), and shows real-time status during navigation (e.g., "Way to goal", "Reached Goal").
- **Bottom row** always shows battery icon, battery percentage, battery voltage, an emergency icon (if active), and the current micro-ROS domain ID.

---

**Q7. What does the battery icon on the display mean?**

The battery icon in the bottom-left corner of the display visually represents the current charge level. The icon changes its fill as the battery drains — a full icon means fully charged, an empty icon means critically low. The icon updates every time the battery state changes.

---

**Q8. What is the domain ID shown in the bottom-right corner of the display?**

The bottom-right corner (positions 13–15 of the second row) shows the current **micro-ROS Domain ID** — a 3-digit number (e.g., `  0`, ` 25`, `169`). This is the ROS 2 domain your robot is currently broadcasting on. It persists across reboots. See [Domain ID & Networking](#9-domain-id--networking) for how to change it.

---

**Q9. What does the emergency icon on the display mean?**

A custom emergency icon appears in the bottom row (position 12) when the **emergency stop is active**. It disappears once the emergency stop is released and the robot resumes normal operation.

---

**Q10. The top row of the display alternates between two messages — is that normal?**

Yes. When the robot is idle and connected, the top row cycles between two messages every 3 seconds:
- The **WiFi SSID** (or `SSID: XX` if network info has not been published yet)
- The **IP address** (or `IP Address: XX` if network info has not been published yet)

During active navigation states (moving to goal, goal reached, etc.), the top row will show the relevant state message instead.

---

## 3. Battery

**Q11. What is the battery voltage range? When is it fully charged vs. critically low?**

| State | Voltage |
|---|---|
| Fully charged | 25.2 V |
| Minimum safe level | 19.8 V |

Battery percentage is linearly mapped between these two voltage points. The display shows both the percentage and the live voltage reading (accuracy ~0.2 V).

---

**Q12. The robot is beeping 5 times every 15 seconds — what does that mean?**

This is the **low battery alert**. It triggers when battery percentage drops **below 15%**. The pattern is 5 rapid beeps (100 ms on / 100 ms off) repeated every 15 seconds. Stop operation, safely park the robot, and recharge immediately.

---

**Q13. How low can I let the battery drain before it causes damage?**

Do **not** let the battery drain below **10%**. Draining the battery below its minimum voltage can cause permanent, irreversible cell damage. The firmware will warn you at 15% — treat this as your signal to recharge.

---

**Q14. Where can I check the battery percentage and voltage via ROS?**

Two topics are published by the robot:

| Topic | Type | Description |
|---|---|---|
| `/[namespace]/battery/percentage` | `std_msgs/Int8` | Battery percentage (0–100) |
| `/[namespace]/battery/voltage` | `std_msgs/Float32` | Battery voltage in Volts |

Example:
```bash
ros2 topic echo /acrux/battery/percentage
ros2 topic echo /acrux/battery/voltage
```

> **Note:** The battery percentage reported over micro-ROS is a linear estimate based on voltage and may not be accurate as the battery ages or degrades. For day-to-day use, prefer monitoring the **voltage** value directly. If your robot is equipped with a **smart BMS**, it provides accurate State of Charge (SOC) and per-cell voltage information via its companion app and dedicated BMS topics.

---

## 4. ROS Topics & Communication

**Q15. What is the ROS namespace for my robot?**

Each robot has a unique namespace. Topics are prefixed with it:

| Robot | Default Namespace | Example Topic |
|---|---|---|
| Diadem | `/diadem` | `/diadem/cmd_vel` |
| Cepheus | `/cepheus` | `/cepheus/cmd_vel` |
| Acrux | `/acrux` | `/acrux/cmd_vel` |
| TortoiseBot Pro Max | *(none by default)* | `/cmd_vel` |

The namespace can be changed at runtime via the `/[namespace]/namespace` topic. The new namespace is saved to flash memory and persists across reboots.

---

**Q16. How do I list all available ROS topics from the robot?**

With the micro-ROS agent running and the robot connected:
```bash
ros2 topic list
```
All robot topics will appear prefixed with the robot's namespace.

---

**Q17. How do I send a velocity command to move the robot?**

Publish to the `cmd_vel` topic:
```bash
ros2 topic pub /acrux/cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.2, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
```
- `linear.x` — forward/backward velocity in m/s
- `angular.z` — rotational velocity in rad/s

To stop:
```bash
ros2 topic pub /acrux/cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.0, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.0}}"
```

---

**Q18. What is the `cmd_vel` topic and what message type does it use?**

`/[namespace]/cmd_vel` uses `geometry_msgs/Twist`. It is both a **subscriber** (robot receives velocity commands) and a **publisher** (robot echoes back the active velocity). The robot will stop automatically if the micro-ROS agent disconnects.

---

**Q19. How do I check the robot's wheel encoder readings?**

| Topic | Type | Format | Description |
|---|---|---|---|
| `/[namespace]/wheel/ticks` | `std_msgs/Int64MultiArray` | `[lf, lb, rf, rb]` | Cumulative encoder ticks |
| `/[namespace]/wheel/vel` | `std_msgs/Float64MultiArray` | `[lf, lb, rf, rb]` | Wheel velocities in m/s |
| `/[namespace]/wheel/rpm` | `std_msgs/Float32MultiArray` | `[lf, lb, rf, rb]` | Wheel RPM |

For 2-wheel robots (Acrux, Cepheus2), only indices `[0]` (left) and `[1]` (right) are used.

---

**Q20. How do I read IMU data from the robot?**

```bash
ros2 topic echo /acrux/imu/raw_data
```
The topic publishes `sensor_msgs/Imu` with orientation, angular velocity, and linear acceleration data from the onboard IMU.

> **Note:** On robots manufactured **before 2026**, IMU data is read by the microcontroller and published over micro-ROS via this topic. On **newer robot versions**, the IMU is a dedicated external sensor connected directly to the main companion PC via USB — in that case, IMU data is handled by the PC-side driver and this micro-ROS topic may not be active. Refer to your robot's hardware documentation to determine your configuration.

---

**Q21. How do I display the network status (IP address, WiFi name) on the robot's screen?**

Publish a JSON-formatted string to `/[namespace]/network_status` once after the micro-ROS agent connects:

```bash
ros2 topic pub -1 /acrux/network_status std_msgs/msg/String \
  "{data: '{\"mode\": \"WiFi\", \"status\": \"Connected\", \"info\": \"MyNetwork\", \"ip\": \"192.168.1.100\"}'}"
```

The display will then cycle between the SSID and IP address. This only needs to be published **once** after each connection.

---

## 5. Navigation & States

**Q22. What are the different navigation states and what do they look like?**

Publish the state to `/[namespace]/nav_status` (`std_msgs/Int32`):

| Value | State | Display | LED | Buzzer |
|---|---|---|---|---|
| `0` | Idle | Cycles SSID/IP | Normal runtime (white/blue/red) | None |
| `1` | Moving to Goal | "Way to goal" | Status LEDs — **navigation color** | 1 long beep (500 ms) |
| `2` | Goal Reached | "Reached Goal" | Status LEDs blink **3× green** | 3 short beeps (200 ms) |
| `3` | Goal Cancelled | "Goal Cancelled" | Status LEDs blink **1× red/orange** | 1 long beep (1000 ms) |
| `4` | Costmap Cleared | "Costmap Cleared" | Status LEDs blink | 4 rapid beeps (100 ms) |
| `5` | Goal Stored | "Goal Stored" | Status LEDs blink | 1 medium beep (700 ms) |

After states 2, 3, 4, and 5, the display and LEDs automatically revert to the previous state after 3 seconds.

---

**Q23. The display says "Goal Cancelled" — what happened?**

Your navigation stack cancelled the active goal and published `3` to the `nav_status` topic. This can happen due to obstacle detection, a new goal being sent, or a manual cancellation. The robot stops motion. Check your navigation system for the cancellation reason.

---

**Q24. The display says "Costmap Cleared" — what does that mean?**

Your navigation system published `4` to `nav_status`, indicating the local or global costmap was cleared. This is typically done to help the robot re-plan around a stuck situation. The robot will resume its previous navigation state after the 3-second display window.

---

**Q25. How do I tell the robot it has reached a goal?**

Publish `2` to the nav status topic from your navigation system:
```bash
ros2 topic pub -1 /acrux/nav_status std_msgs/msg/Int32 "{data: 2}"
```
This triggers the "Reached Goal" feedback (LED blink + 3 beeps + display message).

---

**Q26. What does "Goal Stored" mean?**

It means a new goal location has been saved by the navigation stack (published `5` to `nav_status`). This is informational feedback — the robot has stored the goal and will navigate toward it. The LED and display will briefly acknowledge it and then revert to the previous state.

---

## 6. Emergency Stop

**Q27. How do I trigger the emergency stop?**

Press the physical **Emergency Stop button** on the robot. The firmware uses hardware debouncing to ensure reliable detection.

---

**Q28. What happens to the motors when emergency stop is activated?**

- All motors **immediately stop**.
- All LEDs turn **solid red**.
- The **emergency icon** appears on the LCD display.
- The robot will not respond to any `cmd_vel` commands while the emergency stop is active.
- Navigation status updates are also suppressed.

---

**Q29. How do I resume operation after an emergency stop?**

Release the emergency stop button. The firmware detects the release and the robot resumes normal operation. LEDs return to their normal runtime state.

---

**Q30. How do I monitor the emergency stop status via ROS?**

```bash
ros2 topic echo /acrux/estop/status
```
- `true` — Emergency stop is **active** (motors halted).
- `false` — Normal operation.

---

## 7. PID Control

**Q31. What is PID control and why does the robot use it?**

PID (Proportional-Integral-Derivative) control is a closed-loop feedback algorithm that regulates each wheel's actual speed to match the desired speed from `cmd_vel`. Without PID, hardware differences between motors would cause the robot to drift. The firmware runs an independent PID loop for each wheel, using encoder feedback.

---

**Q32. What PID modes are available?**

| Mode | Value | Description |
|---|---|---|
| **Adaptive** | `1` | Default mode. Uses factory-tuned Kp, Ki, Kd values optimized for your robot model. |
| **Custom** | `2` | Uses user-defined Kp, Ki, Kd values that can be saved to flash memory. |
| **PID Off** | `0` | Disables PID. Motors run at raw PWM. Use for testing only. |

> **Recommendation:** For accurate velocity operation and straight-line driving, always use **Adaptive mode**. Custom mode is not recommended for general use.
>
> **Safety Warning:** If you choose to tune PID values yourself, **always uplift the robot so the wheels are off the ground** before sending any commands. Incorrect tuning can cause sudden, uncontrolled wheel motion. Any damage to gearboxes or motors resulting from improper PID tuning is **not covered under the provided warranty**.

---

**Q33. How do I switch PID modes via ROS?**

```bash
# Switch to Adaptive (default)
ros2 topic pub -1 /acrux/pid/mode std_msgs/msg/Int8 "{data: 1}"

# Switch to Custom
ros2 topic pub -1 /acrux/pid/mode std_msgs/msg/Int8 "{data: 2}"

# Disable PID
ros2 topic pub -1 /acrux/pid/mode std_msgs/msg/Int8 "{data: 0}"
```

Each mode change is confirmed by a buzzer pattern (see [Buzzer Indications](#10-buzzer-indications)).

---

**Q34. How do I update the PID constants?**

Publish an array of `[Kp, Ki, Kd]` to the constants topic:
```bash
ros2 topic pub -1 /acrux/pid/constants std_msgs/msg/Float32MultiArray \
  "{data: [0.358, 38.15, 0.0]}"
```
This updates the active PID tunings immediately.

> **Safety Warning:** Before sending custom PID constants, **always uplift the robot so the wheels are off the ground**. Incorrect tuning values can cause sudden, uncontrolled wheel motion. Any damage to gearboxes or motors resulting from improper PID tuning is **not covered under the provided warranty**.

---

**Q35. How do I save my custom PID settings so they persist after reboot?**

After tuning, publish an empty message to the save topic:
```bash
ros2 topic pub -1 /acrux/pid/custom/save std_msgs/msg/Empty "{}"
```
The firmware writes the current custom Kp, Ki, Kd values to the ESP32's non-volatile flash memory (Preferences API). They will be automatically loaded on next boot when Custom mode is selected.

---

## 8. RC Control

**Q36. Does the robot support RC (remote control) operation?**

RC control support is optional and hardware-dependent. It is enabled on robots that have an **SBUS receiver** connected and have been configured with RC support at the time of purchase. The information in this section is **only applicable if your robot was provided with an RC receiver and has RC configured**. If RC was not included with your robot, this entire section does not apply.

---

**Q37. How do I switch between RC mode and ROS (autonomous) mode?**

On the RC transmitter, **Channel 5** (a 2-position switch) controls the mode:
- Switch to `RC_MODE_VALUE` position → **RC mode active** (ROS `cmd_vel` is ignored)
- Switch to `TELEOP_MODE_VALUE` position → **ROS mode** (robot responds to `cmd_vel`)

When RC mode is active, the status LEDs turn **blue** to indicate manual control.

---

**Q38. What does the blue LED pattern mean when RC is connected?**

Solid **blue** status LEDs mean the RC controller is connected and RC mode is **active**. If the RC is connected but the mode switch is in the teleop/ROS position, the LEDs return to normal runtime colors.

---

**Q39. How do I adjust the speed throttle using the RC transmitter?**

**Channel 8** on the transmitter acts as a throttle scaler. It scales the final velocity commands between ~20% and 100% of maximum speed. Use it to limit speed in tight spaces or during testing.

---

**Q40. Can I change PID modes from the RC transmitter?**

Yes, via **Channel 7** (a 3-position switch):
- Position 1 (`200`) → **PID Off** (1 long beep)
- Position 2 (`1000`) → **Adaptive PID** (1 short beep)
- Position 3 (`1800`) → **Custom PID** (2 short beeps)

Each mode change stops the motors momentarily before applying the new PID configuration.

---

## 9. Domain ID & Networking

**Q41. What is the micro-ROS Domain ID and why does it matter?**

The **ROS 2 Domain ID** isolates ROS 2 networks. Only devices with the same domain ID can see each other's topics. The robot's domain ID is displayed in the bottom-right of the LCD. It must match the `ROS_DOMAIN_ID` environment variable on your computer.

---

**Q42. How do I change the micro-ROS Domain ID?**

With the micro-ROS agent active, publish the new domain ID:
```bash
ros2 topic pub -1 /acrux/microros_domain_id std_msgs/msg/Int8 "{data: 25}"
```
The robot will apply the new domain ID and **automatically reboot** to activate the change. The new ID is saved to EEPROM and survives power cycles.

---

**Q43. Will the Domain ID reset after reboot?**

No. The domain ID is stored in **EEPROM** and persists across reboots. The robot always boots with the last configured domain ID. You can verify it on the bottom-right of the LCD display.

---

**Q44. The robot is not showing up on my ROS network — what should I check?**

1. Confirm the micro-ROS agent is running on your machine.
2. Check that `ROS_DOMAIN_ID` on your computer matches the domain ID shown on the robot's LCD.
3. Verify the robot and your computer are on the same network subnet.
4. Check the LED state — pulsing orange means the robot is still searching for the agent.
5. Try restarting the micro-ROS agent.

---

## 10. Buzzer Indications

**Q45. What does each buzzer pattern mean?**

The buzzer provides audio feedback for all major robot events. Below is the complete reference:

---

### Boot

| Pattern | Beeps | Timing | Event |
|---|---|---|---|
| Rigbetel Classic Tone | 7 beeps | Signature rhythm: `beep · beep · beep · beep — beep · beep · beep` | Robot has powered on and initialized successfully |

The boot tone plays once at startup after the display shows "RigBetel Labs".

---

### micro-ROS Connection

| Pattern | Beeps | Timing | Event |
|---|---|---|---|
| 2 short beeps | 2 | 200 ms on / 100 ms off | micro-ROS agent **connected** |
| 4 rapid beeps | 4 | 100 ms on / 100 ms off | micro-ROS agent **disconnected** |

---

### Navigation

| Pattern | Beeps | Timing | Event |
|---|---|---|---|
| 1 long beep | 1 | 500 ms on | Robot is **moving to goal** |
| 3 short beeps | 3 | 200 ms on / 200 ms off | **Goal reached** |
| 1 very long beep | 1 | 1000 ms on | **Goal cancelled** |
| 4 rapid beeps | 4 | 100 ms on / 100 ms off | **Costmap cleared** |
| 1 medium beep | 1 | 700 ms on / 100 ms off | **Goal stored** |

---

### Battery

| Pattern | Beeps | Timing | Frequency | Event |
|---|---|---|---|---|
| 5 rapid beeps | 5 | 100 ms on / 100 ms off | Every 15 seconds | **Battery below 15%** — recharge immediately |

---

### PID Control

| Pattern | Beeps | Timing | Event |
|---|---|---|---|
| 1 long beep | 1 | 1000 ms on | **PID disabled** (mode set to Off) |
| 1 short beep | 1 | 200 ms on | **Adaptive PID** mode activated |
| 2 short beeps | 2 | 200 ms on / 10–100 ms off | **Custom PID** mode activated |

---

### Encoder Fault

| Pattern | Beeps | Timing | Event |
|---|---|---|---|
| 1 medium beep | 1 | 500 ms on | **Encoder fault detected** on one or more wheels |

This beep repeats each time the encoder health check runs and detects a fault. If you hear repeated 500 ms single beeps during motion, stop the robot and inspect the wheel encoders and their connections.

---

### Buzzer Muted?

If you hear no beeps at all, the buzzer may have been disabled via ROS:
```bash
# Check current state
ros2 topic echo /acrux/buzzer/enable

# Re-enable buzzer
ros2 topic pub -1 /acrux/buzzer/enable std_msgs/msg/Bool "{data: true}"

# Disable buzzer
ros2 topic pub -1 /acrux/buzzer/enable std_msgs/msg/Bool "{data: false}"
```

---

## 11. Troubleshooting

**Q46. The robot powers on but I see no ROS topics — what could be wrong?**

1. The micro-ROS agent is not running. Start it and wait a few seconds.
2. `ROS_DOMAIN_ID` mismatch — check the LCD bottom-right for the robot's domain ID.
3. Network connectivity issue — robot and computer must be on the same subnet.
4. The LED is still pulsing orange — the robot has not connected yet. Wait or restart the agent.

---

**Q47. The LEDs are pulsing orange slowly — what does that indicate?**

Pulsing orange (warm fade in/out) means the robot is in **WAITING_AGENT** state — it is actively searching for a micro-ROS agent but has not found one. Start or restart the micro-ROS agent on your computer.

---

**Q48. All LEDs turned solid red — what happened?**

All-red LEDs mean the **emergency stop is active**. The physical e-stop button has been pressed. Release the button to resume operation.

---

**Q49. The robot is moving erratically or drifting — how do I fix it?**

1. Check that **Adaptive PID** is enabled (PID Off mode causes open-loop drift).
2. Verify encoders are working — listen for 500 ms single beeps which indicate encoder faults.
3. Ensure the battery voltage is adequate — low voltage causes motor performance degradation.

---

**Q50. How do I reset the wheel encoder counts?**

```bash
ros2 topic pub -1 /acrux/wheel/ticks_reset std_msgs/msg/Empty "{}"
```
This resets the cumulative tick counter for all wheels to zero.

---

**Q51. The buzzer won't stop beeping — how do I disable it?**

```bash
ros2 topic pub -1 /acrux/buzzer/enable std_msgs/msg/Bool "{data: false}"
```
Note: This only suppresses future buzzer patterns. It does not disable safety-critical indications permanently — the setting reverts to enabled on reboot.

---

**Q52. The display is not turning on — what should I check?**

Check the wiring and connectors to the LCD display. If the issue persists after verifying the connections, contact **RigBetel Labs** for support.

---

**Q53. I changed the Domain ID but the robot isn't responding on the new ID — what should I do?**

The robot reboots automatically after a Domain ID change. After reboot:
1. Confirm the new ID is shown on the LCD bottom-right corner.
2. Set `ROS_DOMAIN_ID` on your computer to match:
   ```bash
   export ROS_DOMAIN_ID=25
   ```
3. Restart the micro-ROS agent.
4. Run `ros2 topic list` to confirm topics are visible.

---

## 12. Advanced / Developer

**Q54. Can I change the robot's namespace?**

Yes. Publish the desired namespace string to:
```bash
ros2 topic pub -1 /[current_namespace]/namespace std_msgs/msg/String "{data: 'my_robot'}"
```
The new namespace is saved to flash and the robot **automatically reboots**. All topics will then appear under `/my_robot/...`. Namespace must use valid ROS 2 naming characters (alphanumeric and underscore only).

---

**Q55. What microcontroller (ECU) does the robot use?**

The robot uses an **ESP32** microcontroller running micro-ROS firmware. It runs FreeRTOS under the hood, with multiple concurrent tasks managing motor control, display updates, LED animations, IMU publishing, and ROS communication.

---

**Q56. How does the firmware handle loss of micro-ROS agent connection?**

When the agent connection is lost (`AGENT_DISCONNECTED` state):
1. Motors are **stopped immediately** (if not in RC mode).
2. A **4-beep disconnection alert** sounds.
3. LEDs return to the **pulsing orange** no-connection animation.
4. The firmware continuously polls to re-establish connection (`WAITING_AGENT` state).
5. Once the agent is available again, entities are recreated and the robot **reconnects automatically** with a 2-beep confirmation.

---

*For further assistance, contact RigBetel Labs or refer to the full API documentation.*
