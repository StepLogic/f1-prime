---
layout: page
title: F1-Prime Project Overview
---

# F1-Prime: F1Tenth Autonomous Exploration Setup

> **Platform:** NVIDIA Jetson Orin Nano 8GB (JetPack R35.6.4) running ROS 2 Galactic  
> **Vehicle:** F1Tenth 1:10 scale autonomous racecar — inspect f1-prime for chassis model  
> **Competition Context:** [RoboRacer](https://roboracer.ai/) / [F1TENTH](https://f1tenth.org/)

This repository documents the complete autonomous exploration stack running on `f1-prime` (192.168.1.209), a Jetson Orin-powered F1Tenth vehicle configured for frontier-based autonomous mapping and navigation.

---

## Table of Contents

- [Hardware Overview](#hardware-overview)
- [Software Stack](#software-stack)
- [Repository Structure](#repository-structure)
- [Configuration Files](#configuration-files)
- [Quick Start: Launch Exploration](#quick-start-launch-exploration)
- [VNC Remote Desktop Access](#vnc-remote-desktop-access)
- [ROS 2 Topics & Architecture](#ros-2-topics--architecture)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Hardware Overview

| Component | Verified Spec | Source |
|-----------|---------------|--------|
| **Compute** | NVIDIA Jetson Orin Nano 8GB (aarch64, JetPack R35.6.4) | `cat /proc/device-tree/model`, `uname -a` |
| **LiDAR** | Hokuyo UST-10LX (Ethernet, 192.168.0.10:10940) | `sensors.yaml`, `ping` |
| **Motor Controller** | VESC-compatible (STM32 USB VID:PID 0483:5740, `/dev/sensors/vesc`) | `lsusb`, udev rules |
| **Joystick** | Logitech F710 (DirectInput, VID:PID 046d:c219) | `lsusb`, `joy_teleop.yaml` |
| **Vehicle Dimensions** | 0.70 m × 0.40 m (length × width) | Physical measurement |
| **Chassis / Battery** | Inspect physical f1-prime | Physical inspection required |

### LiPo Battery Safety

- **Never leave connected** when not in use — batteries discharge below safe voltage
- **Monitor during charging** — use fireproof bags on non-flammable surfaces
- **Disconnect immediately** if you hear popping, smell burning, or see smoke/bloating
- **Never short leads** or plug in backwards

---

## Software Stack

| Layer | Technology |
|-------|-----------|
| **OS** | Ubuntu 20.04 (JetPack R35.6.4) |
| **ROS Distro** | ROS 2 Galactic |
| **SLAM** | SLAM Toolbox (online async) |
| **Navigation** | Nav2 (Regulated Pure Pursuit controller) |
| **Exploration** | Custom Wavefront Frontier Detector (WFD) |
| **Recording** | ros2 bag (all topics) |
| **VNC Access** | x11vnc + SSH tunnel |

### ROS 2 Workspace

The autonomy stack lives in `/home/f1-prime/autoware/` with the following key directories:

```
/home/f1-prime/autoware/
├── launch/                          # Launch scripts & exploration nodes
│   ├── launch_f1tenth_exploration.sh   # Main exploration launcher
│   ├── frontier_explorer.py            # WFD frontier explorer
│   ├── f1tenth_exploration_launch.py   # Nav2 + SLAM launch
│   └── exploration.rviz                # RViz2 config
├── nav2_config/
│   ├── nav2_params.yaml                # Nav2 parameters
│   └── exploration_bt.xml              # Behavior tree
└── src/universe/autoware.universe/f1tenth/
    └── f1tenth_system/f1tenth_stack/
        ├── config/
        │   ├── sensors.yaml              # LiDAR/VESC config
        │   └── f1tenth_online_async.yaml   # SLAM Toolbox config
        └── launch/
            └── bringup_launch.py         # Hardware bringup
```

---

## Repository Structure

```
.
├── README.md                          # This file
├── connect_f1prime_vnc.sh            # VNC connection helper
├── launch_f1tenth_exploration.sh     # Exploration launcher (copied to f1-prime)
├── frontier_explorer.py              # Frontier explorer node
├── f1tenth_exploration_launch.py     # Nav2 + SLAM launch file
├── nav2_params.yaml                  # Nav2 configuration
├── wait_for_tf.py                    # TF wait utility
├── exploration_bt.xml                # Nav2 behavior tree
├── visualize_bag.py                  # Bag visualization helper
├── joy_teleop_fixed.yaml             # Joystick teleop config
├── fastrlap_setup/                   # FastRLAP pre-trained encoder
│   ├── setup_fastrlap.sh
│   └── ros2_inference_node.py
└── docs/
    └── superpowers/
```

---

## Configuration Files

### 1. Sensors (`sensors.yaml`)

```yaml
urg_node:
  ros__parameters:
    angle_max: 3.14
    angle_min: -3.14
    ip_address: "192.168.0.10"       # Hokuyo UST-10LX ethernet
    ip_port: 10940
    # serial_port: "/dev/sensors/hokuyo"  # USB fallback
    # serial_baud: 115200
    laser_frame_id: "laser"
    cluster: 1
    skip: 0
```

### 2. Nav2 Parameters (`nav2_params.yaml`)

Key exploration-specific settings:

```yaml
controller_server:
  ros__parameters:
    controller_plugins: [FollowPath]
    FollowPath:
      plugin: nav2_regulated_pure_pursuit_controller::RegulatedPurePursuitController
      desired_linear_vel: 0.5          # Speed cap: 0.5 m/s
      max_linear_vel: 0.5
      max_angular_vel: 1.0
      lookahead_dist: 0.6
      min_turning_radius: 0.3
      use_cost_regulated_linear_velocity_scaling: true
      footprint: "[[0.30, 0.15], [0.30, -0.15], [-0.20, -0.15], [-0.20, 0.15]]"

local_costmap:
  local_costmap:
    ros__parameters:
      update_frequency: 5.0
      width: 6
      height: 6
      resolution: 0.05
      plugins: [static_layer, voxel_layer, obstacle_layer, inflation_layer]
      inflation_layer:
        inflation_radius: 0.35
        cost_scaling_factor: 10.0

global_costmap:
  global_costmap:
    ros__parameters:
      update_frequency: 1.0
      resolution: 0.05
      track_unknown_space: true
```

### 3. SLAM Toolbox (`f1tenth_online_async.yaml`)

```yaml
slam_toolbox:
  ros__parameters:
    mode: mapping                    # Online async mapping
    solver_plugin: solver_plugins::CeresSolver
    ceres_linear_solver: SPARSE_NORMAL_CHOLESKY
    ceres_preconditioner: SCHUR_JACOBI
    ceres_trust_strategy: LEVENBERG_MARQUARDT
    ceres_dogleg_type: TRADITIONAL_DOGLEG
    map_update_interval: 1.0
    resolution: 0.05
    max_laser_range: 20.0
    minimum_travel_distance: 0.2
    minimum_travel_heading: 0.2
    scan_buffer_size: 10
    scan_buffer_maximum_scan_distance: 10.0
    link_match_minimum_response_fine: 0.1
    loop_search_maximum_distance: 3.0
    do_loop_closing: true
    loop_match_minimum_chain_size: 10
```

### 4. Exploration Behavior Tree (`exploration_bt.xml`)

The behavior tree defines NavigateToPose with recovery behaviors:
- Clear costmap on failure
- Spin recovery
- BackUp recovery

### 5. udev Rules (on f1-prime)

```bash
# /etc/udev/rules.d/99-hokuyo.rules
KERNEL=="ttyACM[0-9]*", ACTION=="add", ATTRS{idVendor}=="15d1", \
  MODE="0666", GROUP="dialout", SYMLINK+="sensors/hokuyo"

# /etc/udev/rules.d/99-vesc.rules
KERNEL=="ttyACM[0-9]*", ACTION=="add", ATTRS{idVendor}=="0483", \
  ATTRS{idProduct}=="5740", MODE="0666", GROUP="dialout", SYMLINK+="sensors/vesc"

# /etc/udev/rules.d/99-joypad-f710.rules
KERNEL=="js[0-9]*", ACTION=="add", ATTRS{idVendor}=="046d", \
  ATTRS{idProduct}=="c219", SYMLINK+="input/joypad-f710"
```

Activate with:
```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

---

## Quick Start: Launch Exploration

### Prerequisites

1. **Vehicle on ground** in flat, enclosed, hazard-free area
2. **LiDAR connected** and powered
3. **Joystick paired** (deadman's switch enabled)
4. **SSH access** to f1-prime: `ssh f1-prime@192.168.1.209`

### Launch Command

```bash
# On f1-prime:
~/launch_f1tenth_exploration.sh
# Type 'explore' when prompted for safety confirmation
```

Or non-interactively:
```bash
echo "explore" | bash ~/launch_f1tenth_exploration.sh
```

### What It Launches (8 tmux windows)

| Window | Purpose |
|--------|---------|
| `bringup` | Hardware: LiDAR, VESC, odometry, joystick |
| `odom-relay` | `/ego_racecar/odom` → `/odom` |
| `cmd-bridge` | `/cmd_vel` → `/drive` (Ackermann) |
| `prime` | Stationary `/drive` to prime vesc_to_odom |
| `nav2` | SLAM Toolbox + Nav2 stack |
| `explorer` | Wavefront Frontier Detector |
| `rviz` | RViz2 with exploration config |
| `record` | ros2 bag record (all topics) |

### Stop Exploration

```bash
~/launch_f1tenth_exploration.sh --kill
```

### Save Map

```bash
ros2 run nav2_map_server map_saver_cli -f ~/autoware/maps/explored_map
```

### Replay Bag

```bash
ros2 bag play ~/f1tenth_runs/YYYYMM_DD_HHMMSS_explore/bag
```

---

## VNC Remote Desktop Access

For remote monitoring via RViz2, use the provided connection script:

```bash
# On local machine:
./connect_f1prime_vnc.sh
```

This script will:
1. Start x11vnc on f1-prime (if not running)
2. Create SSH tunnel: `localhost:5901 → f1-prime:5901`
3. Launch Remmina with no-password profile
4. Wait for Enter before disconnecting

### Manual Setup

```bash
# On f1-prime (one-time):
nohup env XAUTHORITY=/run/user/1000/gdm/Xauthority DISPLAY=:0 \
  x11vnc -display :0 -nopw -listen localhost -forever -rfbport 5901 \
  -noxdamage > /tmp/x11vnc.log 2>&1 < /dev/null &

# On local machine:
ssh -L 5901:localhost:5901 f1-prime@192.168.1.209 -N
# Then connect VNC viewer to localhost:5901
```

---

## ROS 2 Topics & Architecture

### Published Topics

| Topic | Type | Source |
|-------|------|--------|
| `/scan` | `LaserScan` | Hokuyo LiDAR |
| `/odom` | `Odometry` | VESC odometry |
| `/map` | `OccupancyGrid` | SLAM Toolbox |
| `/drive` | `AckermannDriveStamped` | cmd_vel bridge |
| `/tf` | `TFMessage` | Multiple nodes |
| `/local_costmap/costmap` | `OccupancyGrid` | Nav2 |
| `/global_costmap/costmap` | `OccupancyGrid` | Nav2 |

### Subscribed Topics (Explorer)

| Topic | Type | Purpose |
|-------|------|---------|
| `/map` | `OccupancyGrid` | Frontier detection |
| `/global_costmap/costmap` | `OccupancyGrid` | Reachability verification |
| `/tf` | `TFMessage` | Robot pose lookup |

### Actions

| Action | Type | Purpose |
|--------|------|---------|
| `/navigate_to_pose` | `NavigateToPose` | Send frontier goals |

### Coordinate Frames

```
map → odom → base_link → laser
```

- `map` → `odom`: Published by SLAM Toolbox
- `odom` → `base_link`: Published by vesc_to_odom

---

## Troubleshooting

### "Cannot connect to VNC server"

- Ensure x11vnc is running: `ssh f1-prime "pgrep x11vnc"`
- Ensure tunnel is up: `ss -tlnp | grep 5901`
- Try restarting: `./connect_f1prime_vnc.sh`

### "No odom->base_link TF"

- VESC may not be publishing odometry
- Check `prime` tmux window — it sends a stationary `/drive` command to prime vesc_to_odom
- Verify VESC connection: `ls /dev/sensors/vesc`

### "Costmaps are blind" (no scan raytracing)

- Nav2 launched before odom->base_link TF was available
- The launch script waits 120s for this TF — check `nav2` tmux window logs

### "Frontier explorer not sending goals"

- Check `explorer` tmux window — waits 30s then polls `/bt_navigator` lifecycle state
- Ensure `/bt_navigator` is `active [3]`
- Check `/map` is being published

### Remmina zombie processes

```bash
# Clean up defunct Remmina processes
pkill -9 remmina
```

---

## References

- [RoboRacer Website](https://roboracer.ai/)
- [F1TENTH Build Instructions](https://f1tenth.readthedocs.io/)
- [F1TENTH Race Stack (GitHub)](https://github.com/f1tenth/f1tenth_system)
- [ForzaETH Race Stack Paper](https://roboracer.ai/publications/ForzaETH.pdf)
- [Nav2 Documentation](https://navigation.ros.org/)
- [SLAM Toolbox](https://github.com/SteveMacenski/slam_toolbox)

---

## License

This documentation and configuration are provided for the F1Tenth autonomous racing community. Hardware drivers and base software follow their respective upstream licenses (ROS 2, Nav2, SLAM Toolbox).
# f1-prime
