# VineBot ROS2 Project

This repository contains the ROS2 Humble software stack for the VineBot platform developed at THWS. It includes the Roboteq motor controller integration, Livox Mid-360 LiDAR setup, localization and mapping tools, and launch commands for running the system. Roboteq motor controller code was ported from ROS1 to ROS2 by THWS. 

## Overview

The project is designed to support robot control, perception, and SLAM on the VineBot platform. The main components include:

- Roboteq motor controller integration
- Livox Mid-360 LiDAR driver and configuration
- GNSS support via Ublox - ROS2 code avialable. Sensor to be integrated and tested.
- IMU support via BNO055 - ROS2 code available. Sensor to be integrated and tested.
- SLAM and mapping workflows using Vineslam and RTAB-Map - To be setup

---

## Roboteq Controller Integration

The Roboteq controller integration was developed for the VineBot robot.

1. Install the Serial library by following the instructions in the upstream repository: https://github.com/wjwwood/serial
2. A plugin for roboteq hardware interface is created.
3. Configure the serial port and baud rate in roboteq.launch.py in roboteq_controller_ros2.  

This setup allows the robot to communicate with the Roboteq controller and expose the required hardware interfaces to ROS2.

---

## Connecting Lidar

### 1. Set up the Livox Mid-360 sensor

There are two valid ways to complete the initial setup.

#### Option A: Use the official Livox software

1. Download the sensor software from the official Livox page:
   - https://www.livoxtech.com/mid-360/downloads
2. Disable the firewall:

```bash
sudo ufw disable
```

3. Set a static IP manually on the host machine:
   - IP address: `192.168.2.1`
   - Subnet mask: `255.255.255.0`
   - Gateway: `192.168.1.1`

4. Launch the viewer:

```bash
./LivoxViewer2.sh
```

The host IP and LiDAR IP are printed by the program and can be used to verify the connection.

#### Option B: Follow the community setup guide

- https://github.com/pboon09/Livox-MID-360-LiDAR-Setup

This is an alternative setup path for configuring the sensor and establishing communication.

### 2. Install Livox-SDK2 and run the sample script

Install the SDK from:

- https://github.com/Livox-SDK/Livox-SDK2/tree/master

Then configure the IP in the sample configuration file:

```bash
samples/livox_lidar_quick_start/mid360_config.json
```

Run the sample script:

```bash
cd samples/livox_lidar_quick_start && ./livox_lidar_quick_start ../../../samples/livox_lidar_quick_start/<config-file>
```

### 3. Clone the Livox ROS2 driver and configure it

Clone the Livox ROS2 driver package and adjust the configuration file:

```bash
/src/livox_ros_driver2/config/MID360_config.json
```

Then launch the RViz launch file provided in the Livox ROS2 driver package.

### 4. Current project usage

Currently, VineBot uses RTAB-Map for mapping and localization. The system relies on a 3D camera for visual perception, and the relevant launch files are started from the ROS1 bringup configuration:

```bash
roboteq_control/bringup_vinesalm.launch
```

This is the current setup used in the project workflow for LiDAR and mapping integration.

---

## Build Commands

Use the following commands to build the required ROS2 packages:

To build all packages:
```bash
colcon build --cmake-args -DROS_EDITION=ROS2 -DHUMBLE_ROS=humble
```

To build specific packages:
```bash
colcon build --packages-select livox_ros_driver2 --cmake-args -DROS_EDITION=ROS2 -DHUMBLE_ROS=humble

colcon build --packages-select vineslam vineslam_ros --cmake-args -DCMAKE_BUILD_TYPE=Release -DLIDAR_TYPE=4

colcon build --packages-select ps4_ros2 roboteq serial vinebot_description vineslam vineslam_ros
```

---

## Runtime Launch Commands

### Roboteq Controller

```bash
ros2 launch roboteq roboteq.launch.py
ros2 run joy joy_node
ros2 run ps4_ros2 ps4_ros2
```
OR

Configure the path to source the ROS2 workspace and run the startup script:
```bash
./startup_vinebot.sh
```

Calibrate the PS4 controller - After running the ps4_ros2 node, follow the instructions printed in the terminal to calibrate the controller. The calibration process involves pressing specific buttons.

### Robot setup
After running the above nodes, to move the robot, first remove the emergency stop button from the orange remote. To remove the emergencystop, unscrew the black button on the top and then press the green button on the top. 
Very Important: Always keep the emergency stop remote with you when operating the robot. In case of any emergency, press the emergency stop button to immediately stop the robot.


### Livox and sensor nodes

```bash
ros2 launch livox_ros_driver2 msg_MID360_launch.py
ros2 run bno055 bno055 --ros-args --params-file ./src/bno055/bno055/params/bno055_params.yaml
ros2 launch ublox_gps ublox_gps_node-launch.py
ros2 launch src/vineslam_latest/vineslam/test/test_slam_node/run.launch.py 
```

### Fix output permissions

```bash
sudo chown -R $USER:$USER /home/irobot/Mansi/VineBot/VineSlamOutput
```

---

## RTAB-Map Mapping Workflow

### RTAB-Map SLAM

```bash
ros2 launch realsense2_camera rs_launch.py unite_imu_method:=2 enable_rgbd:=true enable_color:=true enable_depth:=true enable_sync:=true align_depth.enable:=true unite_imu_method:=2 enable_gyro:=true enable_accel:=true

ros2 run imu_filter_madgwick imu_filter_madgwick_node --ros-args -p use_mag:=false -p publish_tf:=false -p world_frame:="enu" --remap /imu/data_raw:=/camera/camera/imu

ros2 launch rtabmap_launch rtabmap.launch.py rtabmap_args:="--delete_db_on_start" rgb_topic:=/camera/camera/color/image_raw depth_topic:=/camera/camera/rgbd camera_info_topic:=/camera/camera/color/camera_info frame_id:=camera_base_link approx_sync:=true wait_imu_to_init:=true imu_topic:=/imu/data rviz:=true
```

### RTAB-Map visual odometry

```bash
ros2 launch realsense2_camera rs_launch.py enable_depth:=true enable_color:=true align_depth.enable:=true

ros2 run rtabmap_odom rgbd_odometry --ros-args -r /rgb/image:=/camera/camera/color/image_raw -r /depth/image:=/camera/camera/aligned_depth_to_color/image_raw -r /rgb/camera_info:=/camera/camera/color/camera_info
```

---

## Notes

- This project is intended for ROS2 Humble.
- The Livox Mid-360 sensor should be connected to the expected network configuration before launching the driver.
- Autonomous navigation not functioning at the moment.


