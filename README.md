> **About this fork**
>
> Fork of [pollen-robotics/mobile_base](https://github.com/pollen-robotics/mobile_base), modified for the Gazebo simulation used in my Master's thesis ([reachy2_teleop_grasping_and_simulation](https://github.com/mrtelisa/reachy2_teleop_grasping_and_simulation), branch `new_protocol`), where the Reachy 2 mobile base is driven through `/cmd_vel` by a body-machine interface.
>
> Changes on `develop`:
> - **Velocity control by default:** `CMD_VEL` is the default mode of `zuuu_hal`, and the Gazebo planar-move plugin runs in velocity mode.
> - **Simulation without the real lidar:** the lidar driver is not started when running in Gazebo, MuJoCo or fake mode.
> - **Collision counter:** a contact sensor on `base_link` publishes the collisions of the base on `/reachy/contacts_state`, used to count collisions with obstacles during the experiments.
> - **Documentation:** a section below explains how a `/cmd_vel` message reaches the simulated base, with checks for debugging.
>
> The rest of this README is the original documentation by Pollen Robotics.

---

# Packages to control the mobile base Zuuu in ROS2
<p align="center">
  <img src="img/mobile_base.png" alt="Mobile Base" />
</p>

## Packages

* [zuuu_hal](./zuuu_hal/) - Hardware Abstraction Layer for the mobile base, ROSifies the inputs and outputs.
* [zuuu_interfaces](./zuuu_interfaces/) - Defines custom services for the mobile base
* [rplidar_ros](./rplidar_ros/) - RPLIDAR ROS2 Package.


See the readme of each package for more information and specific installation needs.

See [`/cmd_vel` → simulated base](#cmd_vel--simulated-base-gazebo) for how a `/cmd_vel` message flows through to the simulated base in Gazebo.

## Installation

* Clone the repository into your ROS2 workspace
* Build the workspace with `colcon build --symlink-install`

## `/cmd_vel` → simulated base (Gazebo)

How a `Twist` on `/cmd_vel` ends up moving Zuuu in the Gazebo sim
(`gazebo:=true`). Applies to any publisher (e.g. `reachy_bomi`), not just one.

```
/cmd_vel → zuuu_hal.cmd_vel_callback() → control_tick() [mode dispatch]
             │
             ├─ CMD_GOTO (default) or CMD_VEL: both consume it
             ▼
        process_velocity_goals_and_send_wheel_commands()
             │  fake_hardware=True (gazebo/mujoco/fake)
             ▼
        publish_fake_robot_speed() → /cmd_vel_gazebo
             │
             ▼
   libgazebo_planar_move_plugin.so (base_link, zuuu.gazebo.xacro)
             │
       moves the robot in Gazebo
   (plugin also publishes odom_gazebo, read back by zuuu_hal)
```

- `zuuu_hal` subscribes to `cmd_vel` (global topic, no remap).
- Mode gate: `zuuu_hal`'s `ZuuuModes` (`utils.py`) — `CMD_VEL`, `CMD_GOTO`, `GOTO`, `SPEED`, `BRAKE`, `FREE_WHEEL`, `EMERGENCY_STOP`. Both `CMD_VEL` and the hard-coded default `CMD_GOTO` use `/cmd_vel` (`CMD_GOTO` adds odometry-corrected smoothing on top) — **no mode change needed**. The other modes ignore it; check with `GetZuuuMode` if the base won't move.
- In sim, `zuuu_hal` republishes the processed velocity as a **new** `Twist` on **`cmd_vel_gazebo`** (not `/cmd_vel`) — that's what the Gazebo plugin actually listens to (`<command_topic>cmd_vel_gazebo</command_topic>` in `zuuu_description/urdf/zuuu.gazebo.xacro`).

**For this to work:** URDF built with `use_gazebo:=true` (set automatically by `reachy_bringup` when `gazebo:=true`), `mobile_base.enable: true` in `reachy_config`, and the `/cmd_vel` publisher actually running/connected.

**Sanity checks:**
```sh
ros2 topic echo /cmd_vel          # is anything arriving?
ros2 topic echo /cmd_vel_gazebo   # is zuuu_hal forwarding it?
ros2 service call /GetZuuuMode zuuu_interfaces/srv/GetZuuuMode
```
Data on `/cmd_vel` but not `/cmd_vel_gazebo` → wrong mode. Neither has data → problem is upstream of `zuuu_hal`.
