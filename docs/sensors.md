# Sensor Guide

Add cameras, lidar, IMUs, contact, and force/torque sensors to your
robot for simulation in NVIDIA Isaac Sim.

## Overview

Two components work together:

1. **Sensor Type Library** — right-click to pick a preset (e.g.
   `realsense_d435`, `velodyne_vlp16`, `imu`). The preset carries
   frequency, FOV, range, and resolution. Full list in
   [presets.md](presets.md#sensor-types).
2. **Sensor** — gives the sensor a name, attaches it to a parent link,
   and places it with a Rhino plane.

| Category | Presets | Output data |
|---|---|---|
| RGB camera | `camera_rgb`, `camera_rgb_hd`, `camera_rgb_fhd`, `camera_rgb_wide`, `camera_rgb_fisheye` | Color images |
| Depth camera | `camera_depth`, `camera_depth_short`, `camera_depth_long`, `realsense_d435`, `kinect` | Depth maps |
| Semantic camera | `camera_semantic` | Segmented images |
| 2D lidar | `lidar_2d`, `lidar_2d_360`, `hokuyo_ust10`, `rplidar_a1` | Planar range scans |
| 3D lidar | `lidar_3d`, `velodyne_vlp16`, `velodyne_hdl32`, `ouster_os1` | Point clouds |
| IMU | `imu`, `imu_high_rate`, `imu_low_power` | Acceleration, angular rates |
| Contact | `contact`, `bumper` | Touch / force events |
| Force/torque | `force_torque`, `force_torque_wrist`, `load_cell` | 6-axis wrench |

---

## Frame conventions

The Sensor Frame plane defines where the sensor sits and where it
points. **Select the Sensor component in Grasshopper to see a live
preview in the Rhino viewport** — the view direction, image
orientation, FOV frustum (cameras) or scan arc and spin axis (lidar)
are drawn for you. Trust the preview.

### Cameras

- **Origin** — at the lens.
- **Z-axis** — the viewing direction (where the camera looks).
- **X-axis** — image *right*.
- **Image up is −Y.** A right-handed plane with Z forward and X right
  necessarily has +Y pointing *down* in the image, so the top of the
  rendered picture is the −Y direction. The viewport preview draws an
  "img up" arrow and a tick on the top edge of the frustum so roll is
  unambiguous.

The exporter converts this optical convention to the USD-native camera
frame automatically — you never need to pre-rotate the plane.

### 2D lidar

- **Origin** — at the scanner's optical center.
- **Z-axis** — the spin axis, pointing UP (perpendicular to the scan
  plane). The scan sweeps in the plane's XY.
- **X-axis** — the scan-zero direction (0° ray). The preview draws the
  scan arc spanning the configured FOV (a full circle for 360°
  presets).

### 3D lidar

Same as 2D: Z is the spin axis, X is scan zero. The vertical channel
fan comes from the preset.

### IMU / contact / force-torque

Align the plane axes with the measurement axes you want — typically the
robot body frame. The preview draws a plain XYZ triad. Place IMUs at
(or near) the link's center of mass with rigid mounting.

---

## Typical setup (Grasshopper)

```
Sensor Type Library  (right-click → realsense_d435)
        │
        ▼
Sensor ── Name:   "front_depth"
       ── Parent: tilt_link        (a Link / Joint Child Link output)
       ── Frame:  plane at the lens, Z pointing forward
        │
        ▼
Assembly (Sensors input)
```

Multiple sensors: feed each Sensor output into a Merge, then into the
Assembly's Sensors input.

### Example sensor suite (JetAuto-class mobile manipulator)

```
Chassis:
├── imu          (imu, 100 Hz)          — odometry / stability
└── base_lidar   (lidar_2d, 270°, 30 m) — navigation

Pan/tilt head:
├── front_camera (camera_rgb_hd)        — main vision
└── front_depth  (realsense_d435)       — obstacle / grasp sensing

Gripper:
└── wrist_depth  (camera_depth_short)   — close-range grasp positioning
```

---

## Choosing presets

- **Match your real hardware** when one of the style presets fits
  (`realsense_d435`, `hokuyo_ust10`, `rplidar_a1`, `velodyne_vlp16`,
  `kinect`, `ouster_os1`). Sim-to-real works best when the simulated
  sensor matches the bench.
- **Resolution costs simulation speed.** Start with 640 × 480 during
  development; raise it once the pipeline works. Rough guidance:

| Resolution | Typical sim impact | Use |
|---|---|---|
| 640 × 480 | light | development, RL training |
| 1280 × 720 | moderate | HD perception |
| 1920 × 1080 | heavy | final quality |

- **Update rates:** cameras 30 Hz, 2D lidar 10–40 Hz, 3D lidar 5–20 Hz,
  IMU 100–400 Hz, F/T 500–1000 Hz — the presets encode sensible values.

---

## Verifying in Isaac Sim

1. Open the exported USD; sensors appear in the Stage panel under their
   parent links.
2. Press Play and use Isaac Sim's sensor tooling (camera viewports,
   lidar debug visualization, IMU readouts) to confirm data flows.
3. Check orientation first — a camera looking backwards or a lidar
   scanning vertically is almost always a plane-axis mistake. Re-check
   against the Grasshopper preview.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Sensor missing in Isaac Sim | Wrong parent link, or sensor not wired to Assembly | Check the Sensor's Parent Link and the Assembly's Sensors input |
| Camera image black | Z-axis not pointing at the scene | Reorient the plane; select the component to preview the frustum |
| Image rotated / upside-down | Plane X not aligned with image-right | Rotate the plane about its Z axis; the preview's top-edge tick shows image-up |
| Lidar returns nothing | Z-axis not the spin axis, or range too short | Z must be perpendicular to the intended scan plane; check Max Range |
| Validation error "Sensor parent link not found" | Parent link not present in the assembly | Make sure the parent link object is also wired into the Assembly |
| Sim runs slowly | Too many / too high-res sensors | Reduce resolution and rates; remove unused sensors |

## Best practices

- Place cameras with a clear line of sight; avoid self-occlusion by the
  robot's own geometry.
- Mount 2D lidars above chassis clutter with an unobstructed sweep.
- Put the IMU at the center of mass, axes aligned with the body frame.
- Test one sensor at a time before wiring the full suite.
- Keep sensor names unique — duplicate names are rejected at export.
