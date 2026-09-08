---
layout: home
title: "F1-Prime: Autonomous Exploration"
description: "F1Tenth autonomous racing platform — frontier-based exploration with SLAM Toolbox, Nav2, and ROS 2 Galactic on NVIDIA Jetson Orin Nano."
---

## Quick Links

| Guide | Description |
|-------|-------------|
| [🧭 Autonomous Exploration Guide]({{ '/documentation/' | relative_url }}) | The illustrated runbook — every command, parameter and fix verified on the real car |
| [🚗 Project Overview](README.md) | Hardware, software stack, ROS 2 topics, VNC access, quick start |
| [📚 Student Replication Guide](STUDENT_REPLICATION_GUIDE.md) | Step-by-step build, install, calibrate, and launch |
| [🚪 Hallway Exploration](HALLWAY_EXPLORATION.md) | Parameter overrides and procedure for narrow hallway navigation |

---

```bash
ssh f1-prime@192.168.1.209
~/launch_f1tenth_exploration.sh
# Type 'explore' when prompted
```

[→ See full setup guide](STUDENT_REPLICATION_GUIDE.md)
