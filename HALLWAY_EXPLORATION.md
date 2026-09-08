---
layout: page
title: Hallway Exploration Approach
---

# Hallway Exploration Approach for F1-Prime

## Problem

Hallways challenge the default exploration stack:
- **Narrow clearance** — walls are close on both sides
- **Long straight walls** — poor SLAM constraints, risk of drift
- **Doors / openings** — side frontiers that may not be desirable
- **Dead ends** — must recognize and back out
- **Long thin frontiers** — WFD may detect unreachable wall edges

## Minimum Requirements

| Requirement | Value | Rationale |
|-------------|-------|-----------|
| **Hallway width** | > 1.10 m | Car width 0.40 m + inflation 0.35 m each side = 1.10 m |
| **Floor** | Flat, smooth | Nav2 assumes flat ground; thresholds / cables will block |
| **Walls** | Solid, opaque to LiDAR | Glass / mirrors will appear as free space |
| **Ceiling height** | > LiDAR mount height | Ensure 360° scan clears obstacles |
| **Lighting** | Even, no direct sun on LiDAR | Hokuyo UST-10LX is IR-based but ambient light affects range |

## Parameter Overrides for Hallway Mode

Create `nav2_params_hallway.yaml` — a copy of `nav2_params.yaml` with these changes:

```yaml
controller_server:
  ros__parameters:
    FollowPath:
      desired_linear_vel: 0.3          # Reduce from 0.5
      max_linear_vel: 0.3
      lookahead_dist: 0.4              # Reduce from 0.6 for tighter control
      min_lookahead_dist: 0.2
      max_lookahead_dist: 0.6
      curvature_lookahead_dist: 0.5    # Reduce from 1.0 — hallways have high curvature at turns
      min_turning_radius: 0.25         # Tighter turns
      use_cost_regulated_linear_velocity_scaling: true  # Slow near walls
      regulated_linear_scaling_min_radius: 0.3

local_costmap:
  local_costmap:
    ros__parameters:
      width: 4                         # Reduce from 6 — tighter local view
      height: 4
      inflation_layer:
        inflation_radius: 0.25         # Reduce from 0.35 — needed for narrow hallways
        cost_scaling_factor: 15.0      # Sharper cost falloff
```

**Critical:** Pin `inflation_radius` to `(hallway_width - 0.40) / 2`. For a 1.00 m hallway, that's 0.30 m. Reducing inflation below the measured clearance leaves the planner no safety margin — it will route the 0.70 m × 0.40 m chassis through gaps it cannot physically fit.

## Operational Procedure

### 1. Pre-Flight Check

```bash
# SSH into f1-prime
ssh f1-prime@192.168.1.209

# Verify hallway width with a tape measure or by walking the car through
# Place car in hallway, power on, verify LiDAR sees both walls:
ros2 topic echo /scan --once | jq '.ranges | length'
# Should show ~1080 range readings (Hokuyo UST-10LX @ 0.25° resolution)
```

### 2. Launch with Hallway Parameters

```bash
# Copy hallway params to the config location
cp ~/autoware/nav2_config/nav2_params_hallway.yaml ~/autoware/nav2_config/nav2_params.yaml

# Launch exploration as normal
~/launch_f1tenth_exploration.sh
# Type 'explore' when prompted
```

### 3. Monitor via VNC

```bash
# On local machine
./connect_f1prime_vnc.sh
```

Watch for:
- **Local costmap** shows both walls as obstacles (red)
- **Global path** stays centered between walls
- **Frontier goals** appear at hallway end or doorways

### 4. If the Car Gets Stuck

| Symptom | Cause | Fix |
|---------|-------|-----|
| Wobbles side-to-side | Lookahead too long for hallway width | Reduce `lookahead_dist` to 0.3 |
| Stops at doorway | Frontier explorer chose side door | Increase `inflation_radius` to block doorways |
| Scrapes wall | Inflation too small or footprint wrong | Verify footprint matches actual car, increase inflation |
| SLAM drifts | Feature-poor hallway | Drive slower, add temporary landmarks (boxes) |
| Dead end, won't turn | `allow_reversing: false` | Set `allow_reversing: true` or manually back out |

## Doorway Handling

By default, the frontier explorer will detect doorways as frontiers (unknown space behind the door frame). To control this:

**Option A: Block doorways with costmap**
- Keep `inflation_radius: 0.35` — doors < 1.10 m wide will be treated as walls
- Only doorways > 1.10 m will be explored

**Option B: Allow doorway exploration**
- Reduce inflation to 0.20 — doorways > 0.80 m are passable
- The car will enter doorways and explore rooms
- Risk: may get stuck in narrow doorways (< 0.80 m)

**Option C: Manual override**
- Attach to tmux explorer window
- When explorer sends goal to doorway, cancel with `Ctrl-C`
- The explorer will pick the next-best frontier (down the hallway)

## Dead-End Recovery

If the car reaches a dead end:

1. **Explorer detects no frontiers** — exploration stops
2. **Kill the stack**: `~/launch_f1tenth_exploration.sh --kill`
3. **Manually reverse** (joystick deadman's switch) or re-launch with `allow_reversing: true`

To enable reversing in `nav2_params_hallway.yaml`:

```yaml
FollowPath:
  allow_reversing: true   # Default: false
```

This lets the planner back out of dead ends autonomously.

## Hallway-Specific RViz Config

Add these to `exploration.rviz` for hallway monitoring:

| Display | Topic | Why |
|---------|-------|-----|
| **Polygon** | `/local_costmap/published_footprint` | See if footprint clips walls |
| **Map** | `/local_costmap/costmap` | Check clearance on both sides |
| **Path** | `/plan` | Verify path stays centered |

## High-Resolution Mapping

For more accurate hallway maps, increase SLAM and costmap resolution from 0.05 m to 0.02 m:

```bash
# On f1-prime — copy high-res SLAM config
scp f1tenth_online_async_highres.yaml f1-prime@192.168.1.209:~/autoware/src/universe/autoware.universe/f1tenth/f1tenth_system/f1tenth_stack/config/

# The nav2_params.yaml in this repo already uses 0.02 m for costmaps
scp nav2_params.yaml f1-prime@192.168.1.209:~/autoware/nav2_config/
```

**Tradeoffs:**
- **Accuracy:** 2.5× finer grid — door jambs, wall edges, and corners resolve sharply
- **Memory:** Map size increases ~6× (25 cells/m² → 2500 cells/m²); monitor with `htop`
- **CPU:** SLAM scan matching runs slower; Jetson Orin Nano handles it but watch thermal throttling
- **Map saving:** `ros2 run nav2_map_server map_saver_cli` produces larger `.pgm` files

## Summary Checklist

Before hallway exploration:

- [ ] Hallway measured > 1.10 m wide at narrowest point
- [ ] Floor checked for thresholds, cables, slippery patches
- [ ] `nav2_params_hallway.yaml` created and copied over
- [ ] VNC viewer ready for real-time monitoring
- [ ] Physical stop method accessible (hallway may be too narrow to walk alongside)
- [ ] Joystick paired in case manual override needed
- [ ] Understand how to `--kill` the stack quickly

---

*This approach reuses the existing exploration stack with parameter overrides. No code changes required.*
