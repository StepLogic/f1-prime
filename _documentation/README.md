# F1TENTH Autonomous Exploration — documentation

The full setup and debugging guide, verified against the real car (f1-prime,
Jetson arm64, ROS 2 Humble, Nav2 1.1.20).

This folder is a Jekyll collection (`collections.documentation` in the site
`_config.yml`), published at **`/documentation/`**. `index.html` carries front
matter and is rendered into `_layouts/doc.html`, so it is no longer a standalone
file — to read it locally, build the site rather than opening it directly:

```
bundle exec jekyll serve   # then open http://127.0.0.1:4000/documentation/
```

```
index.html        the guide (front matter + body; served at /documentation/)
css/styles.css    styles (light + dark), served at /documentation/css/styles.css
config/           nav2_params.yaml, slam_params.yaml, ekf.yaml, vesc_ekf.yaml,
                  exploration_bt.xml — the exact files running on the car
scripts/          launcher, bridges, and every diagnostic tool referenced
```

## Calibrated values for THIS car

| parameter | value | notes |
|---|---|---|
| `steering_angle_to_servo_offset` | `0.58` | stock 0.5304 veered left; higher offset = right |
| `servo_max` | `0.90` | raised from 0.85 — right lock needs 0.8955 |
| motor start threshold | `0.20 m/s` (923 eRPM) | below this the VESC will not turn the motor |
| `min_move_speed` | `0.25 m/s` | measured floor + margin |
| footprint | `0.56 x 0.28 m` | base_link at the REAR AXLE, not the centre |
| min turning radius | `0.94 m` | cannot rotate in place — never enable rotate_to_heading |

## Non-obvious things that cost the most time

1. **DDS** — foreign participants on a shared LAN broke costmap delivery entirely.
   `ROS_DOMAIN_ID=42` + `ROS_LOCALHOST_ONLY=1`.
2. **The VESC IMU is not REP-145** — empty frame_id, zero covariances, gyro in
   deg/s and accel in g. `vesc_imu_conditioner.py` fixes all three.
3. **Loose cables mimic software faults.** A partly-seated servo cable gives
   perfect commands all the way down the chain and a dead servo. It also produced
   bimodal steering measurements that looked like mechanical backlash.
4. **Trace the chain, and ask what each signal PROVES.**
   `/commands/servo/position` is the topic the driver subscribes to — correct
   values there prove nothing about the VESC. Reading `/sensors/core` back proves
   the serial link works both ways.
