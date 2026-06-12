# Plane Placement Guide

Armature uses Rhino **planes** to define coordinate frames: where links
sit, where joints rotate or slide, where sensors look, and where named
frames (TCP, mounts) live. A plane carries both a position (origin) and
an orientation (X/Y/Z axes) — that is exactly what a robot description
needs.

## The one rule that matters

> **Plane origin = where the motion happens.
> Plane Z-axis = the motion axis.**

- **Revolute joint:** origin at the exact center of the bearing/servo
  shaft; Z along the rotation axis.
- **Prismatic joint:** origin at the slide's zero position; Z pointing
  in the direction of positive travel.
- **Fixed joint:** origin at the connection point; orientation is
  irrelevant (no motion).
- **Sensor:** origin at the lens/scanner; Z = view/scan direction (see
  [sensors.md](sensors.md) for the camera image-axes convention).
- **Named frame:** origin and full orientation are exported as-is
  (world space, converted to link-local at export).

Arbitrary axis directions are supported — the exporter orients the
joint's local frames so the joint axis matches your plane exactly. You
do not need to align joints with the world X/Y/Z axes, though doing so
keeps definitions easier to reason about.

---

## Link frames (optional)

By default, link meshes are exported in world coordinates and you don't
need link frames at all — this is the simplest workflow.

If you set **Use Link Frame = True** on Link Enhanced and supply a
plane, the mesh is rebased into that local frame and the link gets an
explicit transform. Use this when you want clean link-local coordinates
(e.g. for downstream tooling that expects the link origin at the joint).

Placement options:

1. **At the joint connecting to the parent** — conventional URDF-style.
2. **At the link's geometric center** — convenient for symmetric parts.
3. **Anywhere meaningful to you** — the math works regardless.

---

## Typical joint axes

| Joint | Typical motion axis | Plane setup |
|---|---|---|
| Pan (yaw) | vertical | Z-axis pointing up |
| Tilt (pitch) | horizontal, sideways | Z-axis pointing left/right |
| Shoulder / elbow | horizontal | Z-axis along the hinge pin |
| Wrist twist | along the arm | Z-axis along the forearm |
| Wheel | wheel axle | Z-axis along the axle |
| Gripper jaw (prismatic) | open/close direction | Z-axis pointing the way the jaw moves |
| Vertical lift (prismatic) | vertical | Z-axis pointing up |

---

## Worked example: mobile manipulator hierarchy

```
base_link (chassis)
  ├── wheel_fl … wheel_rr     (continuous or fixed)
  ├── pan_link                (revolute, Z up)
  │     └── tilt_link         (revolute, Z sideways)
  └── arm_base                (fixed)
        └── shoulder_link     (revolute)
              └── elbow_link  (revolute)
                    └── wrist_link        (revolute)
                          └── gripper_base
                                ├── jaw_left   (prismatic, Z outward)
                                └── jaw_right  (prismatic, Z outward, mimic of left)
```

For each moving joint, place one plane:

- **pan_joint** — origin at the center of the pan bearing, Z up.
- **tilt_joint** — origin on the tilt axis, Z sideways.
- **shoulder/elbow/wrist** — origin at each servo's output shaft
  center, Z along the shaft.
- **jaw joints** — origin at each jaw's closed position, Z pointing in
  the opening direction.

---

## Rhino techniques

- `Plane` — pick origin, X direction, Y direction.
- `PlaneFrom3Points` — when you have reference geometry to snap to.
- `Orient` — copy a plane between repeated features (e.g. four wheel
  mounts).
- **Gumball** — select a plane and verify/adjust its axes visually.
- Use **object snaps** (center, intersection) so joint origins land
  exactly on shaft centers — an offset origin makes the child link orbit
  the wrong point.
- Organize planes on layers (e.g. `joint_frames`, `link_frames`,
  `sensor_frames`) and color-code by type.

For polar-arrayed features (Mecanum rollers), array the plane together
with the geometry so each instance gets a matching joint frame — see
[arrayed-links.md](arrayed-links.md).

---

## Verifying your frames

1. **Joint component preview:** select a Joint component — its frames
   draw in the viewport with RGB axis lines (X red, Y green, Z blue).
2. **Robot Validator:** connect the assembly and drag the Animation
   slider (0–1). Every joint sweeps through its range. Joint origins,
   axes (cyan), and limit arcs draw in the viewport. A joint rotating
   the wrong way or a link orbiting an offset point is immediately
   visible.
3. **Joint Info output:** check the reported origin and axis numbers.

## Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Joint rotates the wrong way | Z-axis flipped | Flip the plane (Rhino `Flip`) or negate with a Grasshopper plane component |
| Child link orbits an offset point | Plane origin not at the shaft center | Snap the origin to the exact rotation center |
| Joint connects at the wrong place | Used the link's center instead of the connection point | Joint frames go at connection points |
| Prismatic jaw moves the wrong direction | Z pointing inward | Point Z in the positive-travel direction |
| Everything piles up at the world origin | Use Link Frame on, but planes missing/invalid | Supply valid planes or turn Use Link Frame off |
