# TortoiseBot Pro Max

**Features via ROS Topics**

The following is a comprehensive list of topics available for interfacing with the TortoiseBot Pro Max. All topics are relative to the robot's ROS namespace. By default, no namespace prefix is set, so topics are published at the root level (e.g., `/cmd_vel`). If a custom namespace has been configured, all topics will be prefixed accordingly (e.g., `/my_robot/cmd_vel`).

## Node Name

`/microros_node`

With namespace: `/<namespace>/microros_node`

---

## Quick Reference — All Topics

| Topic | Direction | Type |
| ----- | --------- | ---- |
| `/cmd_vel` | Subscribe / Publish | `geometry_msgs/Twist` |
| `/nav_status` | Subscribe | `std_msgs/Int32` |
| `/network_status` | Subscribe | `std_msgs/String` |
| `/pid/mode` | Subscribe | `std_msgs/Int8` |
| `/pid/constants` | Subscribe | `std_msgs/Float32MultiArray` |
| `/pid/custom/save` | Subscribe | `std_msgs/Empty` |
| `/buzzer/enable` | Subscribe | `std_msgs/Bool` |
| `/wheel/ticks_reset` | Subscribe | `std_msgs/Empty` |
| `/microros_domain_id` | Subscribe | `std_msgs/UInt8` |
| `/microros_namespace` | Subscribe | `std_msgs/String` |
| `/wheel/ticks` | Publish | `std_msgs/Int64MultiArray` |
| `/wheel/vel` | Publish | `std_msgs/Float64MultiArray` |
| `/wheel/rpm` | Publish | `std_msgs/Float32MultiArray` |
| `/imu/raw_data` | Publish | `sensor_msgs/Imu` |
| `/battery/percentage` | Publish | `std_msgs/Int8` |
| `/battery/voltage` | Publish | `std_msgs/Float32` |
| `/estop/status` | Publish | `std_msgs/Bool` |
| `/user_button` | Publish | `std_msgs/Int8` |

> **Note:** If a namespace is configured, all topics above are prefixed with `/<namespace>/`.

---

## Robot Specifications

| Parameter | Value |
| --------- | ----- |
| Drive Type | Differential Drive (2-Wheel) |
| Wheel Diameter | 80 mm |
| Wheel Separation | 257 mm |
| Max Linear Velocity | ~0.43 m/s |
| IMU | BNO055 (9-DOF) |
| Battery Voltage Range | 19.8 V (0%) – 25.0 V (100%) |
| Display | 16×2 I2C LCD |

---

## ROS Topics — Subscribers

These are topics the robot **subscribes** to. Publish on these topics to command the robot.

---

**`/cmd_vel`**

- **Type:** `geometry_msgs/Twist`
- **Description:** Sends velocity commands to the robot. Linear velocity is applied along the X axis; angular velocity is applied around the Z axis. This is the primary motion control interface.

  | Field | Description |
  | ----- | ----------- |
  | `linear.x` | Forward/backward velocity in m/s |
  | `angular.z` | Rotational velocity in rad/s (positive = counter-clockwise) |

  Example:
  ```bash
  ros2 topic pub /cmd_vel geometry_msgs/msg/Twist "{linear: {x: 0.2}, angular: {z: 0.0}}"
  ```

---

**`/nav_status`**

- **Type:** `std_msgs/Int32`
- **Description:** Sets the robot's current navigation state. The robot reflects the state via buzzer feedback and the onboard display.

  | Value | State | Description |
  | ----- | ----- | ----------- |
  | `0` | **Idle** | Robot is stationary with no active goal. |
  | `1` | **Moving to Goal** | Robot is navigating toward a goal. |
  | `2` | **Goal Reached** | Robot has arrived at the destination. |
  | `3` | **Goal Cancelled** | Active goal has been aborted. |
  | `4` | **Costmap Cleared** | Navigation costmap was reset for re-planning. |
  | `5` | **Goal Stored** | A new goal location has been saved. |

  Example:
  ```bash
  ros2 topic pub -1 /nav_status std_msgs/msg/Int32 "{data: 1}"
  ```

---

**`/network_status`**

- **Type:** `std_msgs/String`
- **Description:** Displays the network connection status on the robot's LCD. The message must be a serialized JSON string containing the fields below.

  > **Note:** Publish this topic only once the micro-ROS agent is connected.

  **JSON Fields:**

  | Key | Description |
  | --- | ----------- |
  | `mode` | Network mode (e.g., `"WiFi"`, `"Ethernet"`) |
  | `status` | Connection status (e.g., `"Connected"`, `"Disconnected"`) |
  | `info` | Network name or identifier (e.g., SSID) |
  | `ip` | Assigned IP address (e.g., `"192.168.1.100"`) |

  Example:
  ```bash
  ros2 topic pub -1 /network_status std_msgs/msg/String "{data: '{\"mode\": \"WiFi\", \"status\": \"Connected\", \"info\": \"MyNetwork\", \"ip\": \"192.168.1.100\"}'}"
  ```

---

**`/pid/mode`**

- **Type:** `std_msgs/Int8`
- **Description:** Selects the PID control mode for the wheel velocity controller.

  | Value | Mode | Description |
  | ----- | ---- | ----------- |
  | `0` | **Off** | PID disabled; raw velocity commands are used directly. |
  | `1` | **Adaptive** | Default tuned PID — recommended for normal operation. |
  | `2` | **Custom** | Uses the user-provided PID constants set via `/pid/constants`. |

  Example:
  ```bash
  ros2 topic pub -1 /pid/mode std_msgs/msg/Int8 "{data: 1}"
  ```

---

**`/pid/constants`**

- **Type:** `std_msgs/Float32MultiArray`
- **Description:** Updates the custom PID constants used when `/pid/mode` is set to `2` (Custom). Values are applied to both wheels simultaneously.

  **Array format:** `[Kp, Ki, Kd]`

  Example:
  ```bash
  ros2 topic pub -1 /pid/constants std_msgs/msg/Float32MultiArray "{data: [0.5, 30.0, 0.0]}"
  ```

---

**`/pid/custom/save`**

- **Type:** `std_msgs/Empty`
- **Description:** Saves the currently active custom PID constants to persistent storage (NVS). The saved values will be restored on the next boot.

  Example:
  ```bash
  ros2 topic pub -1 /pid/custom/save std_msgs/msg/Empty "{}"
  ```

---

**`/buzzer/enable`**

- **Type:** `std_msgs/Bool`
- **Description:** Enables or disables all buzzer sounds on the robot. Default is `true` (enabled).

  Example:
  ```bash
  ros2 topic pub -1 /buzzer/enable std_msgs/msg/Bool "{data: false}"
  ```

---

**`/wheel/ticks_reset`**

- **Type:** `std_msgs/Empty`
- **Description:** Resets the encoder tick counters for both wheels to zero.

  Example:
  ```bash
  ros2 topic pub -1 /wheel/ticks_reset std_msgs/msg/Empty "{}"
  ```

---

**`/microros_domain_id`**

- **Type:** `std_msgs/UInt8`
- **Description:** Permanently changes the micro-ROS DDS domain ID. The new value is saved to persistent storage and takes effect immediately after an automatic reboot. The current domain ID is shown on the bottom-right of the onboard display.

  > **Note:** After publishing, the robot will automatically restart. Ensure your micro-ROS agent is started with the same domain ID.

  Example:
  ```bash
  ros2 topic pub -1 /microros_domain_id std_msgs/msg/UInt8 "{data: 25}"
  ```

---

**`/microros_namespace`**

- **Type:** `std_msgs/String`
- **Description:** Permanently sets the ROS namespace for the robot's node and all topics. The namespace is saved to persistent storage and applied after an automatic reboot. All topics will then be prefixed with `/<namespace>/`.

  > **Note:** Only alphanumeric characters and underscores are valid. The robot will restart automatically after the change.

  Example:
  ```bash
  ros2 topic pub -1 /microros_namespace std_msgs/msg/String "{data: 'tortoisebot'}"
  ```

  After reboot, topics will be available under `/tortoisebot/cmd_vel`, `/tortoisebot/wheel/ticks`, etc.

---

## ROS Topics — Publishers

These are topics the robot **publishes** to. Subscribe to these topics to receive data from the robot.

---

**`/wheel/ticks`**

- **Type:** `std_msgs/Int64MultiArray`
- **Description:** Cumulative encoder tick counts for each wheel. Resets to zero when `/wheel/ticks_reset` is published.

  **Array format:** `[left, right]`

---

**`/wheel/vel`**

- **Type:** `std_msgs/Float64MultiArray`
- **Description:** Measured linear velocities for each wheel in meters per second, calculated from encoder readings.

  **Array format:** `[left, right]` (m/s)

---

**`/wheel/rpm`**

- **Type:** `std_msgs/Float32MultiArray`
- **Description:** Measured RPM (Revolutions Per Minute) for each wheel, calculated from encoder readings.

  **Array format:** `[left, right]` (RPM)

---

**`/imu/raw_data`**

- **Type:** `sensor_msgs/Imu`
- **Description:** Raw IMU data from the onboard BNO055 sensor. Includes orientation (quaternion), angular velocity, and linear acceleration. The `frame_id` is set to `imu_link`.

  | Field | Description |
  | ----- | ----------- |
  | `orientation` | Quaternion (x, y, z, w) |
  | `angular_velocity` | rad/s (x, y, z) |
  | `linear_acceleration` | m/s² (x, y, z) |

---

**`/cmd_vel`**

- **Type:** `geometry_msgs/Twist`
- **Description:** Echoes the last received velocity command. Useful for monitoring what velocity command the robot is currently executing.

---

**`/battery/percentage`**

- **Type:** `std_msgs/Int8`
- **Description:** Battery charge level as a percentage (0–100%). The robot emits a buzzer alert when the battery falls below 15%.

  | Battery Level | Buzzer Behavior |
  | ------------- | --------------- |
  | 15% – 100% | No alert |
  | Below 15% | 5 beeps every 15 seconds |

  > **Caution:** Do not discharge the battery below 10%. Doing so may permanently damage the battery.

---

**`/battery/voltage`**

- **Type:** `std_msgs/Float32`
- **Description:** Real-time battery voltage in Volts.

  | State | Voltage |
  | ----- | ------- |
  | Fully charged | ~24.7 V |
  | Minimum safe | ~19.8 V |

---

**`/estop/status`**

- **Type:** `std_msgs/Bool`
- **Description:** Publishes the status of the hardware Emergency Stop switch.

  | Value | Meaning |
  | ----- | ------- |
  | `true` | Emergency Stop is **active** — motors are halted. |
  | `false` | Emergency Stop is **inactive** — normal operation. |

---

**`/user_button`**

- **Type:** `std_msgs/Int8`
- **Description:** Publishes feedback events from the onboard user-programmable button. Subscribe to this topic to react to physical button presses.

---

## ROS Parameters

The robot exposes the following parameter via the micro-ROS parameter server.

| Parameter | Type | Description |
| --------- | ---- | ----------- |
| `version` | `double` | Firmware version currently running on the robot. |

To read the firmware version:
```bash
ros2 param get /microros_node version
```

---

## Further Reading

The following documents from the RigBetel Labs reference repo cover topics that also apply to the TortoiseBot Pro Max.

### FAQ

| Topic | Relevant Questions |
| ----- | ------------------ |
| Getting started & micro-ROS agent setup | [Q1–Q5](FAQ.md#1-general--getting-started) |
| LCD display layout & status indicators | [Q6–Q10](FAQ.md#2-display--status-indicators) |
| Battery voltage range, low-battery alert, safe discharge limit | [Q11–Q14](FAQ.md#3-battery) |
| ROS topics, namespaces, velocity commands, encoder & IMU data | [Q15–Q21](FAQ.md#4-ros-topics--communication) |
| Navigation states — display, LED, and buzzer behaviour | [Q22–Q26](FAQ.md#5-navigation--states) |
| Emergency stop — trigger, effect, and recovery | [Q27–Q30](FAQ.md#6-emergency-stop) |
| PID control modes, tuning constants, saving to flash | [Q31–Q35](FAQ.md#7-pid-control) |
| Domain ID — changing, persistence, troubleshooting | [Q41–Q43, Q53](FAQ.md#9-domain-id--networking) |
| Full buzzer pattern reference | [Q45](FAQ.md#10-buzzer-indications) |
| Troubleshooting — no topics, orange LEDs, erratic motion | [Q46–Q53](FAQ.md#11-troubleshooting) |
| Namespace change, firmware internals, agent reconnection | [Q54–Q56](FAQ.md#12-advanced--developer) |

### User Guide

For a complete operational walkthrough — boot sequence, ROS 2 integration, motor control, battery management, and safety procedures — see the [User Guide](user_guide.md).
