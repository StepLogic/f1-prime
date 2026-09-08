---
layout: page
title: Student Replication Guide
---

# F1-Prime Student Replication Guide

> **Goal:** Reproduce the f1-prime autonomous exploration platform.  
> **Verified Platform:** NVIDIA Jetson Orin Nano 8GB (JetPack R35.6.4 / Ubuntu 20.04) with ROS 2 Galactic. Physical chassis and power hardware must be inspected on the actual vehicle.

---

## Table of Contents

1. [Verified Hardware Platform](#1-verified-hardware-platform)
2. [Hardware Assembly & Wiring (Inspect f1-prime)](#2-hardware-assembly--wiring-inspect-f1-prime)
3. [Jetson OS Setup](#3-jetson-os-setup)
4. [Network Configuration](#4-network-configuration)
5. [ROS 2 Galactic Installation](#5-ros-2-galactic-installation)
6. [Workspace Setup & Build](#6-workspace-setup--build)
7. [udev Rules](#7-udev-rules)
8. [VESC Calibration](#8-vesc-calibration)
9. [Odometry Calibration](#9-odometry-calibration)
10. [Exploration Stack Setup](#10-exploration-stack-setup)
11. [Launching Autonomous Exploration](#11-launching-autonomous-exploration)
12. [VNC Remote Desktop](#12-vnc-remote-desktop)
13. [Troubleshooting](#13-troubleshooting)
14. [Safety Checklist](#14-safety-checklist)

---

## 1. Verified Hardware Platform

These components are confirmed by inspecting the running f1-prime system:

| Component | Verified Spec | How Verified |
|-----------|---------------|--------------|
| **Compute** | NVIDIA Jetson Orin Nano 8GB (Engineering Reference Developer Kit Super) | `cat /proc/device-tree/model` |
| **OS** | Ubuntu 20.04.6 LTS (kernel 5.10.216-tegra aarch64) | `cat /etc/lsb-release`, `uname -a` |
| **JetPack** | R35.6.4 | `cat /etc/nv_tegra_release` |
| **RAM** | 7.6 GB | `cat /proc/meminfo` |
| **Storage** | 931 GB NVMe SSD (`/dev/nvme0n1`) | `lsblk` |
| **LiDAR** | Hokuyo UST-10LX (Ethernet, IP 192.168.0.10:10940) | `sensors.yaml` + `ping 192.168.0.10` |
| **Motor Controller** | VESC-compatible (STM32 USB VID:PID 0483:5740) | `lsusb` → `/dev/ttyACM0` → `/dev/sensors/vesc` |
| **Joystick** | Logitech F710 (DirectInput, VID:PID 046d:c219) | `lsusb` + udev rules + `joy_teleop.yaml` |
| **WiFi** | Built-in Orin WiFi (wlan0 at 192.168.1.209) | `ip addr show wlan0` |
| **Ethernet** | eth0 at 192.168.0.15/24 (LiDAR subnet) | `ip addr show eth0` |

### Components Requiring Physical Inspection of f1-prime

The following cannot be determined remotely — inspect the physical vehicle:

| Component | What to Verify |
|-----------|----------------|
| **Chassis** | Manufacturer, model, wheelbase (for `wheelbase` param) |
| **Battery** | Chemistry, cell count, nominal voltage, capacity, connector type |
| **Power to Jetson** | How Jetson is powered — inspect power cable, regulator, barrel jack / USB-C / header |
| **VESC Model** | Exact model and firmware version (check VESC label or VESC Tool) |
| **Motor** | Brushless DC type, KV rating, sensorless or sensored |
| **Servo** | Model, PWM range, mounting geometry |
| **LiDAR Mount** | Height and offset from `base_link` for URDF / static TF |
| **Physical Mounting** | How Jetson and LiDAR are attached (vibration isolation? standoffs? plate?) |

### Software Stack (Free)

- Ubuntu 20.04 (JetPack R35.6.4)
- ROS 2 Galactic (380 packages installed)
- SLAM Toolbox
- Nav2
- F1TENTH driver stack (autoware.universe fork)

---

## 2. Hardware Assembly & Wiring (Inspect f1-prime)

### 2.1 What the Config Files Confirm

The software configuration reveals the following connectivity:

```
+----------------------------+         +-------------------------+
|     NVIDIA Jetson Orin     |         |      Hokuyo UST-10LX    |
|     Nano                   |         |      LiDAR              |
|                            |         |      IP: 192.168.0.10   |
|  eth0: 192.168.0.15/24    |<------->|      Port: 10940        |
|  USB: STMicro VESC         |         |                         |
|  (ttyACM0 / sensors/vesc)  |         +-------------------------+
|  USB: Logitech F710        |
|  (js0 / input/joypad-f710) |
+----------------------------+
```

### 2.2 Minimum Wiring Requirements (From Config + `lsusb`)

1. **VESC → Jetson via USB** — The VESC enumerates as STMicroelectronics Virtual COM Port (`/dev/ttyACM0`). This single USB connection carries:
   - Motor speed commands (ERPM)
   - Servo position commands
   - Odometry feedback (from motor encoder / ERPM)
   - IMU telemetry (if VESC firmware supports it)
2. **Hokuyo LiDAR → Jetson via Ethernet** — Static IP `192.168.0.10` on LiDAR; Jetson eth0 must be `192.168.0.15/24`.
3. **Logitech F710 → Jetson via USB** — Wireless receiver dongle.
4. **Power → Jetson** — Inspect f1-prime to determine power input method (barrel jack, USB-C, or header pins) and regulator.

### 2.3 Physical Inspection Checklist

Before replicating, physically inspect f1-prime and record:

- [ ] **Chassis model** (manufacturer nameplate, serial number)
- [ ] **Wheelbase** (measure center-front-axle to center-rear-axle in meters)
- [ ] **Vehicle footprint** (length × width for Nav2 `footprint` param)
- [ ] **VESC model** (label on VESC case — needed for VESC Tool calibration)
- [ ] **Battery type** (LiPo? cell count? nominal voltage? capacity?)
- [ ] **Power regulator** (part number, input voltage, output voltage, current rating)
- [ ] **LiDAR mount** (height and offset from `base_link`)
- [ ] **Jetson mount** (vibration isolation? standoff height?)

### 2.4 Update Config After Inspection

After measuring the physical vehicle, update these parameters:

| Parameter | Location | What to Measure |
|-----------|----------|----------------|
| `wheelbase` | `vesc.yaml` → `vesc_to_odom_node` | Front axle to rear axle distance |
| `footprint` | `nav2_params.yaml` | Vehicle bounding box |
| `speed_to_erpm_gain` | `vesc.yaml` | Calibrate per motor (see Section 8) |
| `steering_angle_to_servo_gain` | `vesc.yaml` | Calibrate per servo (see Section 8) |

---

## 3. Jetson OS Setup

### 3.1 Flash JetPack

1. Download **NVIDIA SDK Manager** on a host PC (Ubuntu 18.04+).
2. Connect Jetson Orin Nano in recovery mode:
   - Hold **REC** button, press and release **PWR**, release **REC**.
3. In SDK Manager, select:
   - **Target Hardware:** Jetson Orin Nano
   - **JetPack Version:** 5.1.4 (R35.6.4) or newer
   - **OS:** Ubuntu 20.04
4. Flash the OS image.

### 3.2 Initial System Configuration

```bash
# On the Jetson (after first boot):
sudo apt update && sudo apt upgrade -y
sudo apt install -y \
    build-essential git wget curl vim htop \
    python3-pip python3-venv python3-dev \
    net-tools iputils-ping

# Set hostname
sudo hostnamectl set-hostname f1-prime

# Add user to dialout group for USB serial access
sudo usermod -a -G dialout $USER
# Log out and back in for group changes to take effect
```

### 3.3 Verify Hardware Detection

```bash
# Verify VESC detected
lsusb | grep STMicroelectronics
# Should show: Bus 001 Device 004: ID 0483:5740 STMicroelectronics Virtual COM Port

# Verify LiDAR reachable
ping -c 3 192.168.0.10
# Should respond from Hokuyo UST-10LX
```

---

## 4. Network Configuration

### 4.1 LiDAR Ethernet (eth0)

The Hokuyo UST-10LX ships with factory IP `192.168.0.10`. Configure the Jetson eth0:

```bash
# Via nmcli (recommended)
nmcli connection add type ethernet \
    ifname eth0 \
    con-name "Hokuyo-LiDAR" \
    ipv4.method manual \
    ipv4.addresses 192.168.0.15/24

nmcli connection up "Hokuyo-LiDAR"

# Verify
ping -c 3 192.168.0.10
```

### 4.2 WiFi (wlan0)

```bash
# Connect to your lab network
nmcli device wifi list
nmcli device wifi connect "YourLabSSID" password "YourPassword"

# Verify IP
ip addr show wlan0
# Should show IP on your lab subnet (e.g., 192.168.1.x)
```

### 4.3 IP Assignment Reference

| Device | Interface | IP Address | Subnet |
|--------|-----------|------------|--------|
| Hokuyo UST-10LX | Ethernet | 192.168.0.10 | /24 |
| Jetson (LiDAR) | eth0 | 192.168.0.15 | /24 |
| Jetson (WiFi) | wlan0 | DHCP (e.g., 192.168.1.209) | /24 |

---

## 5. ROS 2 Galactic Installation

### 5.1 Add ROS Repositories

```bash
sudo apt install -y curl gnupg lsb-release
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
sudo apt update
```

### 5.2 Install ROS 2 Galactic Base

```bash
# Install ROS 2 Galactic base (no GUI tools — saves space on Jetson)
sudo apt install -y ros-galactic-ros-base

# Install essential tools
sudo apt install -y python3-colcon-common-extensions python3-rosdep python3-vcstool

# Initialize rosdep
sudo rosdep init
rosdep update
```

### 5.3 Source ROS in `.bashrc`

```bash
echo "source /opt/ros/galactic/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### 5.4 Verify Installation

```bash
ros2 --version
# Should show: usage: ros2 [-h] ...

ros2 topic list
# Should show empty list (no nodes running yet)
```

---

## 6. Workspace Setup & Build

### 6.1 Create Workspace

```bash
cd ~
mkdir -p autoware/src
cd autoware/src
```

### 6.2 Clone F1TENTH Driver Stack

```bash
# Clone the F1TENTH system repo
git clone --branch galactic-devel https://github.com/f1tenth/f1tenth_system.git
cd f1tenth_system
git submodule update --init --recursive --remote
```

### 6.3 Clone Autoware Universe (Optional — for full autoware stack)

```bash
# If using autoware.universe (as on f1-prime)
cd ~/autoware/src
mkdir -p universe
cd universe
git clone --branch galactic https://github.com/autowarefoundation/autoware.universe.git
# Note: This is a large repo. Only clone the f1tenth packages if disk space is limited.
```

### 6.4 Install Dependencies

```bash
cd ~/autoware
rosdep install --from-paths src --ignore-src -y --rosdistro galactic
```

### 6.5 Install Additional ROS Packages

```bash
# Nav2 and SLAM Toolbox (critical for exploration)
sudo apt install -y \
    ros-galactic-nav2-bringup \
    ros-galactic-slam-toolbox \
    ros-galactic-tf2-tools \
    ros-galactic-ackermann-msgs \
    ros-galactic-joy \
    ros-galactic-joy-teleop

# Python dependencies
pip3 install numpy scipy
```

### 6.6 Build Workspace

```bash
cd ~/autoware
source /opt/ros/galactic/setup.bash
colcon build --symlink-install

# Source the workspace
echo "source ~/autoware/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

### 6.7 Verify Build

```bash
# Check that packages are available
ros2 pkg list | grep f1tenth
# Should show: f1tenth_stack, f1tenth_gym_ros, etc.

ros2 pkg list | grep nav2
# Should show: nav2_bringup, nav2_controller, etc.
```

---

## 7. udev Rules

Create persistent device names for VESC, LiDAR, and joystick.

### 7.1 Create Rule Files

```bash
# Hokuyo LiDAR (USB fallback — not needed for ethernet LiDAR, but good to have)
sudo tee /etc/udev/rules.d/99-hokuyo.rules << 'EOF'
KERNEL=="ttyACM[0-9]*", ACTION=="add", ATTRS{idVendor}=="15d1", MODE="0666", GROUP="dialout", SYMLINK+="sensors/hokuyo"
EOF

# VESC
sudo tee /etc/udev/rules.d/99-vesc.rules << 'EOF'
KERNEL=="ttyACM[0-9]*", ACTION=="add", ATTRS{idVendor}=="0483", ATTRS{idProduct}=="5740", MODE="0666", GROUP="dialout", SYMLINK+="sensors/vesc"
EOF

# Logitech F710 Joystick
sudo tee /etc/udev/rules.d/99-joypad-f710.rules << 'EOF'
KERNEL=="js[0-9]*", ACTION=="add", ATTRS{idVendor}=="046d", ATTRS{idProduct}=="c219", SYMLINK+="input/joypad-f710"
EOF
```

### 7.2 Activate Rules

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger

# Reboot to ensure rules take effect
sudo reboot
```

### 7.3 Verify After Reboot

```bash
# Check device symlinks exist
ls -la /dev/sensors/
# Should show: hokuyo, vesc

ls -la /dev/input/joypad-f710
# Should show symlink to js0 or js1
```

---

## 8. VESC Calibration

The VESC must be calibrated to your specific motor and steering servo. This is **critical** — wrong parameters will cause erratic behavior.

### 8.1 VESC Tool Setup

1. Download **VESC Tool** from [vesc-project.com](https://vesc-project.com/) (free, requires account).
2. Connect VESC to host PC via USB-C.
3. Run VESC Tool and connect.

### 8.2 Motor Detection (Required)

1. In VESC Tool: **Motor > Setup Motors FOC > Large** (for car-sized motors).
2. Click **Run Detection** — the motor will spin. Ensure wheels are off the ground.
3. Save parameters to VESC.

### 8.3 Input Calibration

1. **APP Settings > PPM / ADC > Mapping:**
   - Connect your RC transmitter or test with VESC Tool's virtual controls.
   - Set `PPM Control Type` to `Current No Reverse with Brake`.
2. **Servo Output:**
   - Connect steering servo to VESC servo output.
   - Test servo range and center in VESC Tool.

### 8.4 VESC Configuration (`vesc.yaml`)

After calibration, copy the detected values to `~/autoware/src/f1tenth_system/f1tenth_stack/config/vesc.yaml`:

```yaml
ros__parameters:
  # ERPM = speed_to_erpm_gain * speed (m/s) + speed_to_erpm_offset
  speed_to_erpm_gain: 7700.0    # Calibrate with your motor
  speed_to_erpm_offset: 0.0

  # Servo = steering_angle_to_servo_gain * angle (rad) + offset
  steering_angle_to_servo_gain: -1.2135
  steering_angle_to_servo_offset: 0.45

  port: /dev/sensors/vesc

  # Safety limits
  servo_min: 0.15
  servo_max: 0.85
  speed_min: -23250.0
  speed_max: 23250.0

vesc_to_odom_node:
  ros__parameters:
    odom_frame: odom
    base_frame: base_link
    publish_tf: true
    use_servo_cmd_to_calc_angular_velocity: true
    wheelbase: 0.25   # Distance between front and rear axles (meters)

throttle_interpolator:
  ros__parameters:
    max_acceleration: 2.5          # m/s²
    throttle_smoother_rate: 75.0   # msgs/sec
    max_servo_speed: 3.2           # rad/s
    servo_smoother_rate: 75.0      # msgs/sec
```

### 8.5 Wheelbase Measurement

Measure the actual wheelbase of your car (center of front axle to center of rear axle). The overall vehicle is 0.70 m long; wheelbase is the distance between the front and rear axles, which is shorter than total length. Update the `wheelbase` parameter in `vesc_to_odom_node`:

```bash
# Measure front axle center to rear axle center
# Update in vesc.yaml → vesc_to_odom_node → wheelbase
```

---

## 9. Odometry Calibration

Accurate odometry is required for SLAM and navigation.

### 9.1 ERPM Gain Calibration

The `speed_to_erpm_gain` converts meters/second to electrical RPM. Calibrate:

1. Place car on flat ground.
2. Run bringup: `ros2 launch f1tenth_stack bringup_launch.py`
3. Publish a slow command: `ros2 topic pub /drive ackermann_msgs/msg/AckermannDriveStamped "{drive: {speed: 1.0, steering_angle: 0.0}}"`
4. Measure actual speed with a stopwatch over a known distance (e.g., 2 meters).
5. Adjust `speed_to_erpm_gain`:
   - If actual speed < 1.0 m/s → decrease gain
   - If actual speed > 1.0 m/s → increase gain

### 9.2 Steering Calibration

1. With bringup running, use joystick to steer full left and full right.
2. Check `/sensors/core` topic for servo position.
3. Adjust `steering_angle_to_servo_gain` and `steering_angle_to_servo_offset` so that:
   - Center steering → servo position ~0.5
   - Full left → matches your car's mechanical limit
   - Full right → matches your car's mechanical limit

### 9.3 Verify Odometry

```bash
# Monitor odometry
ros2 topic echo /odom

# Drive forward 2 meters, check position.x
# Should be approximately 2.0
```

---

## 10. Exploration Stack Setup

### 10.1 Copy Launch Scripts

Copy these files from this repository to `~/autoware/launch/` on the Jetson:

```
launch_f1tenth_exploration.sh    # Main launcher
f1tenth_exploration_launch.py    # Nav2 + SLAM launch
frontier_explorer.py             # Frontier detection node
wait_for_tf.py                   # TF wait utility
cmd_vel_to_ackermann.py          # Twist → Ackermann bridge
odom_relay.py                    # Topic relay
exploration.rviz                 # RViz2 config
```

### 10.2 Copy Configuration Files

```
nav2_params.yaml                 # Nav2 parameters
exploration_bt.xml               # Behavior tree
```

### 10.3 Configure Nav2 Parameters

Edit `nav2_params.yaml` for your car:

```yaml
controller_server:
  ros__parameters:
    FollowPath:
      desired_linear_vel: 0.5   # Max speed during exploration (m/s)
      max_linear_vel: 0.5
      max_angular_vel: 1.0
      lookahead_dist: 0.6
      min_turning_radius: 0.3
      footprint: "[[0.30, 0.15], [0.30, -0.15], [-0.20, -0.15], [-0.20, 0.15]]"

local_costmap:
  local_costmap:
    ros__parameters:
      footprint: "[[0.30, 0.15], [0.30, -0.15], [-0.20, -0.15], [-0.20, 0.15]]"

global_costmap:
  global_costmap:
    ros__parameters:
      footprint: "[[0.30, 0.15], [0.30, -0.15], [-0.20, -0.15], [-0.20, 0.15]]"
```

**Footprint dimensions:** Measure your car's bounding box:
- Length: ~0.50m (front bumper to rear)
- Width: ~0.30m (left to right)
- Adjust footprint to match your actual vehicle.

### 10.4 Configure SLAM Toolbox

Edit `f1tenth_online_async.yaml`:

```yaml
slam_toolbox:
  ros__parameters:
    mode: mapping
    odom_frame: odom
    map_frame: map
    base_frame: base_link
    scan_topic: /scan
    resolution: 0.02              # 2.5× finer grid for accurate LiDAR mapping
    max_laser_range: 20.0
    minimum_travel_distance: 0.5
    map_update_interval: 2.0     # Faster updates to match higher resolution
```

### 10.5 Configure Sensors

Edit `sensors.yaml` for your LiDAR:

```yaml
urg_node:
  ros__parameters:
    angle_max: 3.14
    angle_min: -3.14
    ip_address: "192.168.0.10"    # Hokuyo UST-10LX
    ip_port: 10940
    # For USB LiDAR, comment ip_address and uncomment:
    # serial_port: "/dev/sensors/hokuyo"
    # serial_baud: 115200
    laser_frame_id: "laser"
```

---

## 11. Launching Autonomous Exploration

### 11.1 Pre-Launch Checklist

- [ ] Car is **on the ground** in a flat, enclosed area
- [ ] No stairs, drop-offs, or cables nearby
- [ ] LiDAR powered and connected (ping 192.168.0.10)
- [ ] VESC connected (`ls /dev/sensors/vesc`)
- [ ] Joystick paired (Logitech F710 — switch set to "D")
- [ ] Physical stop method ready (hand on car, kill switch, or power cutoff)

### 11.2 Launch Command

```bash
# SSH into the Jetson
ssh f1-prime@192.168.1.209

# Run the exploration launcher
~/launch_f1tenth_exploration.sh

# Type 'explore' when prompted for safety confirmation
```

### 11.3 What Happens

The script creates 8 tmux windows:

| Window | Purpose |
|--------|---------|
| `bringup` | Hardware: LiDAR, VESC, odometry, joystick |
| `odom-relay` | `/ego_racecar/odom` → `/odom` |
| `cmd-bridge` | `/cmd_vel` (Twist) → `/drive` (Ackermann) |
| `prime` | Sends one stationary `/drive` to prime vesc_to_odom |
| `nav2` | SLAM Toolbox + Nav2 stack (waits for odom→base_link TF) |
| `explorer` | Frontier explorer (waits 30s, then sends goals) |
| `rviz` | RViz2 with exploration config |
| `record` | ros2 bag record (all topics) |

### 11.4 Monitor Progress

```bash
# Attach to tmux session
tmux attach -t f1tenth-explore

# Navigate windows: Ctrl+B, then window number (0–7)
# Detach: Ctrl+B, then D
```

### 11.5 Stop Exploration

```bash
~/launch_f1tenth_exploration.sh --kill
```

### 11.6 Save Map

```bash
ros2 run nav2_map_server map_saver_cli -f ~/autoware/maps/explored_map
# Creates: explored_map.yaml + explored_map.pgm
```

---

## 12. VNC Remote Desktop

The Jetson runs a headless GNOME desktop at 1920×1080 via an EDID fake display. An `x11vnc` systemd service starts automatically on boot.

### 12.1 Jetson Headless Setup (One-Time)

Based on [Mauro Arcidiacono's Jetson Headless VNC Guide](https://mauroarcidiacono.github.io/jetson-headless-vnc/):

```bash
# SSH into f1-prime
ssh f1-prime@192.168.1.209

# 1. Install x11vnc and set a VNC password
sudo apt-get update
sudo apt-get install -y x11vnc
x11vnc -storepasswd   # creates ~/.vnc/passwd

# 2. Ensure graphical.target
systemctl get-default   # should be "graphical.target"
# If not: sudo systemctl set-default graphical.target && sudo reboot

# 3. Enable GDM auto-login in /etc/gdm3/custom.conf
#    [daemon]
#    AutomaticLoginEnable=true
#    AutomaticLogin=f1-prime

# 4. Install EDID fake display for 1920×1080
sudo mkdir -p /lib/firmware/edid
sudo cp EDID_1920x1080.bin /lib/firmware/edid/EDID_1920x1080.bin
# Update /etc/X11/xorg.conf with ConnectedMonitor + CustomEDID options
# (see repo file: f1tenth_system/f1tenth_stack/config/xorg.conf)

# 5. Enable x11vnc systemd service
sudo systemctl enable x11vnc.service
sudo reboot

# 6. Verify after reboot
systemctl status x11vnc.service
DISPLAY=:0 xrandr   # should show 1920x1080
```

### 12.2 On Local Machine

Use the provided `connect_f1prime_vnc.sh` script:

```bash
./connect_f1prime_vnc.sh
```

Or manually:

```bash
# Create SSH tunnel
ssh -L 5900:localhost:5900 f1-prime@192.168.1.209 -N

# In another terminal, connect VNC viewer
vncviewer localhost::5900
# or: remmina -c ~/.config/remmina/f1prime_vnc.remmina
```

---

## 13. Troubleshooting

### "No odom→base_link TF"

- VESC not publishing odometry.
- Check `prime` window — it sends a stationary `/drive` to activate vesc_to_odom.
- Verify VESC connection: `ls /dev/sensors/vesc`

### "Costmaps are blind" (no obstacles)

- Nav2 launched before odom→base_link TF was ready.
- The launch script waits 120s by default — check `nav2` window logs.
- Restart with longer wait: `NAV2_TF_WAIT=180 ~/launch_f1tenth_exploration.sh`

### "Frontier explorer not sending goals"

- Check `explorer` window — waits for `/bt_navigator` lifecycle state `active [3]`.
- Verify `/map` is published: `ros2 topic hz /map`

### "LiDAR not detected"

- Verify ethernet IP: `ping 192.168.0.10`
- Check LiDAR power LED (should be solid green).
- For USB LiDAR: verify udev rule and `/dev/sensors/hokuyo`.

### "VESC not responding"

- Check USB connection: `lsusb | grep STMicroelectronics`
- Verify udev rule activated (reboot if needed).
- Check VESC is powered (LED on VESC should blink).

### "Joystick not working"

- Verify receiver is plugged in.
- Check switch on F710 is set to **"D"** (DirectInput), not "X".
- Verify udev rule: `ls /dev/input/joypad-f710`

### "Remmina zombie processes"

```bash
pkill -9 remmina
```

---

## 14. Safety Checklist

### Before EVERY Autonomous Run

- [ ] **Car is on the ground** — never run on blocks or stands
- [ ] **Flat, enclosed area** — no stairs, drop-offs, or obstacles
- [ ] **LiPo battery secured** — no loose wires or connectors
- [ ] **Physical stop ready** — hand on car or power cutoff accessible
- [ ] **No bystanders in path**
- [ ] **Joystick in range** — deadman's switch works if needed

### LiPo Battery Safety

- **Never leave connected** when not in use
- **Monitor during charging** — fireproof bag required
- **Disconnect immediately** on popping, bloating, smoke, or burning smell
- **Never short leads** or plug in backwards
- **Storage voltage:** 3.7–3.85V per cell (use LiPo checker)

### Emergency Stop

| Method | Action |
|--------|--------|
| Software | `~/launch_f1tenth_exploration.sh --kill` |
| Joystick | Release deadman's switch (RB button) |
| Physical | Grab car, block wheel, or disconnect battery |
| Remote | SSH in and run `pkill -f "ros2 launch"` |

---

## Appendix A: File Locations Reference

| File | Path on Jetson |
|------|----------------|
| Main launcher | `~/launch_f1tenth_exploration.sh` |
| Nav2 params | `~/autoware/nav2_config/nav2_params.yaml` |
| SLAM config | `~/autoware/src/f1tenth_system/f1tenth_stack/config/f1tenth_online_async.yaml` |
| Sensors config | `~/autoware/src/f1tenth_system/f1tenth_stack/config/sensors.yaml` |
| VESC config | `~/autoware/src/f1tenth_system/f1tenth_stack/config/vesc.yaml` |
| Joystick config | `~/autoware/src/f1tenth_system/f1tenth_stack/config/joy_teleop.yaml` |
| Mux config | `~/autoware/src/f1tenth_system/f1tenth_stack/config/mux.yaml` |
| Exploration launch | `~/autoware/launch/f1tenth_exploration_launch.py` |
| Frontier explorer | `~/autoware/launch/frontier_explorer.py` |
| RViz config | `~/autoware/launch/exploration.rviz` |
| udev rules | `/etc/udev/rules.d/99-*.rules` |
| Bag recordings | `~/f1tenth_runs/` |
| Saved maps | `~/autoware/maps/` |

## Appendix B: ROS 2 Topic Reference

| Topic | Type | Publisher |
|-------|------|-----------|
| `/scan` | `sensor_msgs/LaserScan` | Hokuyo LiDAR |
| `/odom` | `nav_msgs/Odometry` | vesc_to_odom |
| `/map` | `nav_msgs/OccupancyGrid` | SLAM Toolbox |
| `/drive` | `ackermann_msgs/AckermannDriveStamped` | cmd_vel bridge |
| `/tf` | `tf2_msgs/TFMessage` | Multiple nodes |
| `/local_costmap/costmap` | `nav_msgs/OccupancyGrid` | Nav2 |
| `/global_costmap/costmap` | `nav_msgs/OccupancyGrid` | Nav2 |
| `/navigate_to_pose` | `nav2_msgs/NavigateToPose` | explorer |

## Appendix C: TF Tree

```
map
 └── odom
      └── base_link
           └── laser
```

- `map → odom`: Published by SLAM Toolbox
- `odom → base_link`: Published by vesc_to_odom
- `base_link → laser`: Static transform (LiDAR mount offset)

---

**Document Version:** 1.0  
**Last Updated:** 2026-08-03  
**Platform:** f1-prime (Jetson Orin Nano, JetPack R35.6.4, ROS 2 Galactic)
