# Preset Libraries

The exact values shipped in each library component. Pick presets via
the component's right-click menu; optional inputs override individual
fields. See [components.md](components.md) for the component reference.

Contents:

- [Physics materials (21)](#physics-materials)
- [Joint drives / motors (10)](#joint-drives--motors)
- [Transmissions (10)](#transmissions)
- [Joint types (16)](#joint-types)
- [Sensor types (27)](#sensor-types)
- [Visual material presets (12)](#visual-material-presets)
- [Named frame roles (9)](#named-frame-roles)
- [Robot types (8)](#robot-types)

---

## Physics materials

Component: **Material Library**. Density drives automatic mass
calculation (mass = volume × density). Friction is split into static
(μs, just-before-slip) and dynamic (μk, during-slip) — real materials
have μk ≈ 0.7–0.9 × μs, and using a single value for both produces
sticky-then-launch contact in PhysX that makes grasping unreliable.

### Metals

| Name | Density (kg/m³) | μs | μk | Restitution | Notes |
|---|---:|---:|---:|---:|---|
| `aluminum` | 2700 | 0.45 | 0.35 | 0.10 | Lightweight metal, good for structural components |
| `steel` | 7850 | 0.55 | 0.42 | 0.20 | Heavy-duty structural metal |
| `stainless_steel` | 8000 | 0.45 | 0.35 | 0.20 | Corrosion-resistant steel |
| `titanium` | 4500 | 0.40 | 0.32 | 0.15 | High strength-to-weight ratio |
| `brass` | 8500 | 0.40 | 0.32 | 0.15 | Decorative metal alloy |
| `copper` | 8960 | 0.40 | 0.32 | 0.15 | Electrical conductor |

### Plastics and rubbers

| Name | Density (kg/m³) | μs | μk | Restitution | Notes |
|---|---:|---:|---:|---:|---|
| `abs_plastic` | 1040 | 0.35 | 0.28 | 0.08 | Common 3D printing plastic |
| `pla_plastic` | 1250 | 0.40 | 0.30 | 0.08 | Biodegradable 3D printing plastic |
| `nylon` | 1150 | 0.30 | 0.22 | 0.10 | Durable engineering polymer |
| `polycarbonate` | 1200 | 0.40 | 0.30 | 0.10 | Impact-resistant engineering plastic |
| `acrylic` | 1180 | 0.45 | 0.35 | 0.10 | Clear plastic (plexiglass) |
| `rubber` | 1100 | 1.10 | 0.90 | 0.05 | Vulcanized rubber — tire / roller material |
| `tpu_rubber` | 1210 | 1.20 | 0.95 | 0.04 | Printable TPU (Shore 95A) — gripper-pad rated |
| `silicone_rubber` | 1100 | 1.40 | 1.05 | 0.05 | Cast silicone gripper pad — maximum grip |
| `foam` | 50 | 0.55 | 0.45 | 0.10 | Lightweight padding material |

`tpu_rubber` and `silicone_rubber` appear under the *Gripper Pads*
submenu — use them on jaws and contact pads.

### Composites

| Name | Density (kg/m³) | μs | μk | Restitution | Notes |
|---|---:|---:|---:|---:|---|
| `carbon_fiber` | 1600 | 0.35 | 0.28 | 0.08 | High-performance lightweight composite |
| `fiberglass` | 1900 | 0.40 | 0.30 | 0.10 | Glass fiber reinforced plastic |

### Wood

| Name | Density (kg/m³) | μs | μk | Restitution | Notes |
|---|---:|---:|---:|---:|---|
| `wood_pine` | 550 | 0.55 | 0.42 | 0.10 | Softwood lumber |
| `wood_oak` | 750 | 0.60 | 0.48 | 0.12 | Hardwood lumber |
| `plywood` | 600 | 0.55 | 0.42 | 0.10 | Layered engineered wood |
| `mdf` | 700 | 0.50 | 0.38 | 0.08 | Medium density fiberboard |

---

## Joint drives / motors

Component: **Joint Drive Library**. Stiffness and damping are tuned for
typical arm-link reflected inertia (targeting roughly a 10–15 Hz
natural frequency at ~70% damping). They are deliberately conservative:
an undertuned joint can be bumped per joint, but an infinitely-stiff
default destroys sim-to-real transfer.

| Name | Mode | Max force (N·m) | Stiffness (N·m/rad) | Damping (N·m·s/rad) | Max velocity (rad/s) | Joint friction (N·m) | Notes |
|---|---|---:|---:|---:|---:|---:|---|
| `hiwonder_lx225` | Position | 1.8 | 60 | 4.0 | 5.0 | 0.02 | Hiwonder LX-225 bus servo (~25 kg·cm, JetAuto arm class) |
| `hiwonder_hx35h` | Position | 2.8 | 200 | 10.0 | 4.0 | 0.04 | Hiwonder HX-35H heavy-duty (~35 kg·cm) |
| `hiwonder_lx15d` | Position | 1.1 | 35 | 2.5 | 6.0 | 0.015 | Hiwonder LX-15D small bus servo (~15 kg·cm) |
| `dynamixel_xl430` | Position | 1.4 | 120 | 5.0 | 6.0 | 0.015 | Robotis Dynamixel XL430-W250-T (~1.5 N·m, 61 rpm) |
| `dynamixel_xm430` | Position | 3.6 | 220 | 10.0 | 4.8 | 0.03 | Robotis Dynamixel XM430-W350-T (~4.1 N·m, 46 rpm) |
| `generic_hobby` | Position | 0.9 | 30 | 2.0 | 5.5 | 0.02 | Generic hobby servo (MG996R-class) |
| `generic_industrial` | Position | 30.0 | 1500 | 60.0 | 3.0 | 0.10 | Generic industrial harmonic-drive joint |
| `velocity_motor` | Velocity | 5.0 | 0 | 50.0 | 8.0 | 0.01 | Velocity-control motor (continuous rotation) |
| `torque_motor` | Torque | 10.0 | 0 | 1.0 | — (unset) | 0.02 | Torque-control motor (soft drive, high effort) |
| `passive` | Velocity | 0 | 0 | 0.05 | — (unset) | 0.01 | Passive joint — no drive, light damping/friction (Mecanum rollers) |

For prismatic joints, max velocity is in m/s and forces in N.

Legacy alias names `position_servo`, `servo`, and `position` resolve to
`hiwonder_lx225` so old definitions keep working.

---

## Transmissions

Component: **Transmission**. When applied to a motor:
torque × ratio × η, speed ÷ ratio, stiffness and damping × ratio².

| Name | Ratio | Efficiency | Notes |
|---|---:|---:|---|
| `direct` | 1.0 | 0.99 | Direct drive (no transmission) |
| `belt_2to1` | 2.0 | 0.92 | GT2 timing belt 2:1 reduction |
| `belt_3to1` | 3.0 | 0.92 | GT2 timing belt 3:1 reduction |
| `belt_4to1` | 4.0 | 0.90 | GT2 timing belt 4:1 reduction |
| `gear_pair_5to1` | 5.0 | 0.90 | Printed spur gear pair 5:1 |
| `gear_pair_10to1` | 10.0 | 0.88 | Printed spur gear pair 10:1 |
| `planetary_27to1` | 27.0 | 0.82 | Common stepper planetary 27:1 |
| `harmonic_50to1` | 50.0 | 0.78 | Strain-wave harmonic drive 50:1 |
| `harmonic_100to1` | 100.0 | 0.78 | Strain-wave harmonic drive 100:1 |
| `harmonic_160to1` | 160.0 | 0.75 | Strain-wave harmonic drive 160:1 |

Reflected inertia and backlash default to 0 and can be set on the
component (backlash is informational — PhysX has no native backlash).

---

## Joint types

Component: **Joint Type Library**. Limits are degrees for revolute,
meters for prismatic. Stiffness/damping/max-force units differ by kind
(N·m/rad, N·m·s/rad, N·m for revolute; N/m, N·s/m, N for prismatic).
A Motor connected on the Joint overrides stiffness, damping, and max
force.

### Revolute

| Name | Limits (deg) | Stiffness | Damping | Max force | Notes |
|---|---|---:|---:|---:|---|
| `revolute` | −180 … 180 | 50 | 3 | 5 | Standard revolute joint, full rotation range (LX-225 class) |
| `revolute_limited` | −90 … 90 | 50 | 3 | 5 | Revolute limited to ±90° |
| `revolute_270` | −135 … 135 | 50 | 3 | 5 | Revolute with 270° range |
| `continuous` | unbounded | 0 | 0.05 | 0 | Continuous rotation (wheel, spinner) — passive damped; adds joint friction 0.02 N·m and child angular damping 2.0 |
| `shoulder` | −180 … 180 | 200 | 10 | 3.5 | High-torque shoulder joint (HX-35H class) |
| `elbow` | −135 … 135 | 60 | 4 | 2 | Elbow joint with typical range (LX-225 class) |
| `wrist` | −180 … 180 | 35 | 2.5 | 1.1 | Low-torque wrist joint (LX-15D class) |
| `finger` | 0 … 90 | 35 | 2.5 | 1.1 | Gripper finger joint (LX-15D class) |

### Prismatic

| Name | Limits (m) | Stiffness | Damping | Max force | Notes |
|---|---|---:|---:|---:|---|
| `prismatic` | 0 … 1.0 | 500 | 30 | 50 | Standard linear actuator (0–1 m) |
| `prismatic_short` | 0 … 0.1 | 500 | 30 | 25 | Short-stroke linear actuator (0–10 cm) |
| `prismatic_long` | 0 … 2.0 | 800 | 50 | 100 | Long-stroke linear actuator (0–2 m) |
| `linear_rail` | −1.0 … 1.0 | 1500 | 80 | 200 | Bidirectional linear rail (±1 m, industrial) |
| `gripper_slide` | 0 … 0.05 | 200 | 15 | 10 | Gripper sliding motion (0–5 cm) |

### Fixed

| Name | Notes |
|---|---|
| `fixed` | Rigid fixed connection |
| `welded` | Permanently welded connection |

Fixed joints have no limits or drive parameters.

---

## Sensor types

Component: **Sensor Type Library**. "Style" presets approximate the
named hardware's published specs for simulation purposes.

### RGB cameras

| Name | Frequency (Hz) | FOV (deg) | Range (m) | Resolution | Notes |
|---|---:|---:|---:|---|---|
| `camera_rgb` | 30 | 60 | 100 | 640 × 480 | Standard RGB camera |
| `camera_rgb_hd` | 30 | 60 | 100 | 1280 × 720 | HD RGB camera (720p) |
| `camera_rgb_fhd` | 30 | 60 | 100 | 1920 × 1080 | Full HD RGB camera (1080p) |
| `camera_rgb_wide` | 30 | 120 | 100 | 640 × 480 | Wide-angle RGB camera |
| `camera_rgb_fisheye` | 30 | 180 | 50 | 640 × 480 | Fisheye RGB camera |

### Depth cameras

| Name | Frequency (Hz) | FOV (deg) | Range (m) | Resolution | Notes |
|---|---:|---:|---:|---|---|
| `camera_depth` | 30 | 60 | 10 | 640 × 480 | Standard depth camera |
| `camera_depth_short` | 30 | 60 | 3 | 640 × 480 | Short-range depth camera (0–3 m) |
| `camera_depth_long` | 30 | 60 | 20 | 640 × 480 | Long-range depth camera |
| `realsense_d435` | 30 | 87 | 10 | 1280 × 720 | Intel RealSense D435 style |
| `kinect` | 30 | 70 | 4.5 | 512 × 424 | Kinect v2 style depth sensor |

### Semantic cameras

| Name | Frequency (Hz) | FOV (deg) | Range (m) | Resolution | Notes |
|---|---:|---:|---:|---|---|
| `camera_semantic` | 30 | 60 | 100 | 640 × 480 | Semantic segmentation camera |

### 2D lidars

Resolution is samples per scan.

| Name | Frequency (Hz) | FOV (deg) | Range (m) | Samples | Notes |
|---|---:|---:|---:|---:|---|
| `lidar_2d` | 10 | 270 | 30 | 1080 | Standard 2D lidar (270°) |
| `lidar_2d_360` | 10 | 360 | 30 | 1440 | 360-degree 2D lidar |
| `hokuyo_ust10` | 40 | 270 | 10 | 1081 | Hokuyo UST-10LX style |
| `rplidar_a1` | 5 | 360 | 12 | 360 | RPLidar A1 style |

### 3D lidars

Resolution is horizontal samples × vertical channels.

| Name | Frequency (Hz) | FOV (deg) | Range (m) | Resolution | Notes |
|---|---:|---:|---:|---|---|
| `lidar_3d` | 10 | 360 | 100 | 1024 × 64 | Standard 3D lidar |
| `velodyne_vlp16` | 20 | 360 | 100 | 1800 × 16 | Velodyne VLP-16 (Puck) style |
| `velodyne_hdl32` | 10 | 360 | 100 | 2200 × 32 | Velodyne HDL-32E style |
| `ouster_os1` | 10 | 360 | 120 | 1024 × 64 | Ouster OS1-64 style |

### IMUs

| Name | Frequency (Hz) | Notes |
|---|---:|---|
| `imu` | 100 | Standard IMU (accelerometer + gyroscope) |
| `imu_high_rate` | 400 | High-rate IMU for fast dynamics |
| `imu_low_power` | 50 | Low-power IMU |

### Contact sensors

| Name | Frequency (Hz) | Notes |
|---|---:|---|
| `contact` | 100 | Contact / touch sensor |
| `bumper` | 50 | Bumper contact sensor |

### Force/torque sensors

| Name | Frequency (Hz) | Notes |
|---|---:|---|
| `force_torque` | 1000 | 6-axis force/torque sensor |
| `force_torque_wrist` | 500 | Wrist-mounted F/T sensor |
| `load_cell` | 100 | Simple load cell |

---

## Visual material presets

Component: **Visual Material**. RGB values are 0–1; metallic,
roughness, and opacity come from the component inputs (defaults 0.0 /
0.5 / 1.0).

| Name | RGB | | Name | RGB |
|---|---|---|---|---|
| `aluminum` | 0.75, 0.75, 0.75 | | `yellow` | 1.00, 1.00, 0.00 |
| `steel` | 0.60, 0.60, 0.65 | | `black` | 0.10, 0.10, 0.10 |
| `brass` | 0.70, 0.60, 0.30 | | `white` | 0.90, 0.90, 0.90 |
| `copper` | 0.70, 0.45, 0.30 | | `gray` | 0.50, 0.50, 0.50 |
| `red` | 1.00, 0.00, 0.00 | | `rubber_black` | 0.15, 0.15, 0.15 |
| `blue` | 0.00, 0.00, 1.00 | | | |
| `green` | 0.00, 1.00, 0.00 | | | |

---

## Named frame roles

Component: **Named Frame**. Roles are semantic hints used by Isaac Lab
/ MoveIt to auto-discover frames:

`tcp`, `end_effector`, `tool_flange`, `camera_mount`, `imu_mount`,
`lidar_mount`, `calibration_target`, `attachment`, `custom`

---

## Robot types

Component: **Robot Type Library**. The eight canonical SimReady tokens
(case-sensitive):

`End Effector`, `Manipulator`, `Humanoid`, `Wheeled`, `Holonomic`,
`Quadruped`, `Mobile Manipulators`, `Aerial`

`Manipulator` and `End Effector` require a pinned (Static) root link
under the SimReady Runnable profile; the other six require a non-pinned
root. See [simready.md](simready.md).
