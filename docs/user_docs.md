# ROS Topic Reference

**Features via ROS Topics**

The following is a comprehensive list of topics essential for low-level control. Each topic is prefixed with the robot's namespace, which in this case is set to diadem—resulting in all topics being structured as /diadem/topic_name.

> **Note:** The namespace (`diadem` in the examples below) is configurable. Replace it with your robot's assigned namespace (e.g., `/acrux`, `/cepheus`) wherever applicable. All topic paths follow the pattern `/<your_namespace>/topic_name`.

---

## Topic Index

### Subscribers

| Topic | Type | Description |
|-------|------|-------------|
| [`/diadem/nav_status`](#diademnavstatus) | `std_msgs/Int32` | Set LED indicators and buzzer for navigation state |
| [`/diadem/network_status`](#diademnetworkstatus) | `std_msgs/String` | Display network info on robot's screen |
| [`/diadem/pid/mode`](#diampidmode) | `std_msgs/Int8` | Set PID control mode |
| [`/diadem/pid/constants`](#diampidconstants) | `std_msgs/Float32MultiArray` | Update PID constants |
| [`/diadem/pid/custom/save`](#diampidcustomsave) | `std_msgs/Empty` | Save custom PID configuration |
| [`/diadem/cmd_vel`](#diademcmdvel) | `geometry_msgs/Twist` | Send velocity commands |
| [`/diadem/buzzer/enable`](#diadembuzzerenable) | `std_msgs/Bool` | Enable/disable buzzer |
| [`/diadem/wheel/ticks_reset`](#diademwheelticksreset) | `std_msgs/Empty` | Reset encoder ticks |
| [`/diadem/microros_domain_id`](#diademicrorosdomainid) | `std_msgs/Int8` | Change micro-ROS domain ID |

### Publishers

| Topic | Type | Description |
|-------|------|-------------|
| [`/diadem/battery/percentage`](#diambatteryperentage) | `std_msgs/Int8` | Remaining battery percentage |
| [`/diadem/battery/voltage`](#diambatteryvoltage) | `std_msgs/Float32` | Current battery voltage |
| [`/diadem/wheel/ticks`](#diademwheelticks) | `std_msgs/Int64MultiArray` | Encoder ticks for all wheels |
| [`/diadem/wheel/vel`](#diademwheelvel) | `std_msgs/Float64MultiArray` | Current wheel velocities |
| [`/diadem/wheel/rpm`](#diademwheelrpm) | `std_msgs/Float32MultiArray` | Wheel RPM values |
| [`/diadem/cmd_vel`](#diademcmdvel-1) | `geometry_msgs/Twist` | Publishes velocity commands |
| [`/diadem/imu/raw_data`](#diademimrawdata) | `sensor_msgs/Imu` | IMU orientation and acceleration data |
| [`/diadem/estop/status`](#diademestopstatus) | `std_msgs/Bool` | Emergency stop button status |

---

## Node Name

`/diadem/microros_node`

## **Low-Level ROS Topics Subcriber**

**`/diadem/nav_status`**

- **Type:** `std_msgs/Int32`
- **Description:** Publising on this topic sets the LED indicators and buzzer to display the robot's current navigation state, such as idle, moving to goal, goal reached, goal canceled, costmap cleared, or goal stored.
  | `robot_status` Value | Description |
  | -------------------- | ---------------------------------------------------------------------------------------------------- |
  | `0` | **Idle State:** The robot is stationary. |
  | `1` | **Moving to Goal:** The robot is navigating towards a goal. |
  | `2` | **Goal Reached:** The robot has successfully arrived at its destination and stops motion. |
  | `3` | **Goal Cancelled:** The robot's mission is aborted, indicated by status indicators and a buzzer. |
  | `4` | **Costmap Cleared:** The robot has cleared its navigation costmap, which helps in re-planning paths. |
  | `5` | **Goal Stored:** A new goal location has been saved for future navigation. |

**`/diadem/network_status`**

- **Type:** `std_msgs/String`
- **Description:** This topic is responsible for displaying the network status of the robot on robot’s display, including mode, connection status, WiFi network name, and assigned IP address. The message is serialized as a JSON string.
  **Note :: Data on`/diadem/network_status` needs to be published only once micro-ROS agent is connected.**
- **Message Format:** The message follows the JSON structure:
  Publishing Example:
  ```bash
  {
  	"mode": "WiFi",  // Network mode (e.g., WiFi, Ethernet)
  	"status": "Connected",  // Connection status
  	"info": "MyWiFiNetwork",  // WiFi network name
  	"ip": "192.123.255.255"  // Assigned IP address
  }
  ```
- To manually publish a network status message, using command:
  ```bash
  ros2 topic pub /diadem/network_status std_msgs/msg/String "{data: '{\"mode\": \"WiFi\", \"status\": \"Connected\", \"info\": \"MyWiFiNetwork\", \"ip\": \"192.123.4.12\"}'}"
  ```

**`/diadem/pid/mode`**

- **Type:** `std_msgs/Int8`
- **Description:** This topic is of type `int` and is used to control the Proportional-Integral-Derivative (PID) controller.

  - `0` - Stop PID control
  - `1` - Adaptive PID control

    Default pid mode is `Adaptive`.

Here's an example:

```
ros2 topic pub -1 /diadem/pid/mode std_msgs/msg/Int8 "{data: 1}"
```

**`/diadem/pid/constants`**

- **Type:** `std_msgs/Float32MultiArray`
- **Description:** Updates the PID constants.

**`/diadem/pid/custom/save`**

- **Type:** `std_msgs/Empty`
- **Description:** Saves custom PID configurations.

**`/diadem/cmd_vel`**

- **Type:** `geometry_msgs/Twist`
- **Description:** Provides velocity commands in linear and angular dimensions.The `/diadem/cmd_vel` topic is responsible for receiving velocity commands for the robot. These commands can be generated by teleoperation or the `move_base` module, instructing the robot on how fast to move in different directions.

**`/diadem/buzzer/enable`**

- **Type:** `std_msgs/Bool`
- **Description:** Controls the enabling/disabling of buzzer functions. Default is `true`.

**`/diadem/wheel/ticks_reset`**

- **Type:** `std_msgs/Empty`
- **Description:** Reset encoder ticks for all the wheels.

**`/diadem/microros_domain_id`**

- **Type:** `std_msgs/Int8`
- **Description:** The /microros_domain_id enables you to change your firmware's domain id permanently. Once changed, the firmware remembers your last set value and always boots up with the same domain id. You can also see current domain id on the display (bottom-right).

  To change the domain id, make sure microros agent is active, Open a new terminal and publish your custom domain id:

  ```
  ros2 topic pub -1 /diadem/microros_domain_id std_msgs/msg/Int8 "{data: 169}"
  ```

  The robot will change the domain ID and reset the ECU automatically to make the necessary changes. You will able to see the topics on your custom domain id now.

## **Low-Level ROS Topics Publisher**

**`/diadem/battery/percentage`**

- **Type:** `std_msgs/Int8`
- **Description:** This topic provides information about the remaining battery percentage of the robot.
  | Battery Percentage | Beeping Sounds |
  | ---------------------- | --------------------- |
  | 100 - 15 | No beeping |
  | Below 15 | 5 Beeps Every 15 Seconds |

> **Tip: To ensure you are aware of the robot's battery status, pay attention to the beeping sounds, especially as the battery percentage decreases.**

> **Caution :: Do not drain the battery below `10 %`, doing so can damage the battery permanently.**

**`/diadem/battery/voltage`**

- **Type:** `std_msgs/Float32`
- **Description:** This topic reports the current battery voltage, ranging from 25.2V at maximum charge to 19.8 V at minimum charge.

**`/diadem/wheel/ticks`**

- **Type:** `std_msgs/Int64MultiArray`
- **Description:** This topic provides an array of ticks for all four wheels of the robot, in the format `[lf, lb, rf, rb]`. These values represent the encoder readings of the wheel ticks.

**`/diadem/wheel/vel`**

- **Type:** `std_msgs/Float64MultiArray`
- **Description:** The `/diadem/wheel/vel` topic sends an array of calculated current velocities for each wheel on the robot, received via encoders. The format of the array is `[lf, lb, rf, rb]`, representing the actual velocity at which each wheel is moving.

**`/diadem/wheel/rpm`**

- **Type:** `std_msgs/Float32MultiArray`
- **Description:** Provides the RPM (Revolutions Per Minute) of the wheels in the format `[lf, lb, rf, rb]`.

**`/diadem/cmd_vel`**

- **Type:** `geometry_msgs/Twist`
- **Description:** Publishes the robot's velocity commands.

**`/diadem/imu/raw_data`**

- **Type:** `sensor_msgs/Imu`
- **Description:** Provides IMU (Inertial Measurement Unit) data including orientation, angular velocity, and linear acceleration.

**`/diadem/estop/status`**

- **Type:** `std_msgs/Bool`
- **Description:** Publishes the status of the Emergency Stop button.
