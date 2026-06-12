<p align="center"><img src="icon.png" width="128" alt="Armature"></p>

# Armature

**Sim-ready robot export for Rhino + Grasshopper.**

Author articulated robots in Rhino and export physics-correct USD for
NVIDIA Isaac Sim / Isaac Lab, MuJoCo, and any USD-Physics consumer:
rigid bodies with CAD-derived inertia tensors, joints with realistic
servo dynamics, transmissions, sensors, named frames, and a validator
that catches sim-breaking mistakes before export. Exports validate
against NVIDIA's SimReady Foundation Robot-Body profiles.

## Install

In Rhino 7 or 8 (Windows), run the `_PackageManager` command, search
**armature**, and click Install. Nothing else to set up — the USD
converter ships as a self-contained executable.

- Package: https://yak.rhino3d.com/packages/armature
- Food4Rhino listing: (link follows after review)

## Documentation

- [Getting started](docs/getting-started.md) — 15 minutes from Rhino to Isaac Sim
- [Component reference](docs/components.md) — every component, input, output, and right-click preset
- [Preset libraries](docs/presets.md) — all material, motor, transmission, joint-type, and sensor values
- [Material workflow](docs/materials.md) — mass from geometry, the friction model, gripper pads
- [Plane placement](docs/plane-placement.md) — link, joint, and sensor frame conventions
- [Sensors](docs/sensors.md) — cameras, lidar, IMU, force/torque
- [Arrayed links](docs/arrayed-links.md) — many identical links/joints from a few components
- [SimReady conformance](docs/simready.md) — Robot-Body-Neutral / Runnable profiles
- [Troubleshooting](docs/troubleshooting.md) — exploding robots and other classics

## Support

- **Bugs / questions:** open an [issue](../../issues)
- **Commercial support** (priority fixes, Isaac Sim / Isaac Lab / ROS 2
  integration, custom development): thorsoej@gmail.com

## License

Free for personal, educational, and commercial use. Closed source.
Files you export belong entirely to you.
