# Component Reference

Every Armature component, found in Grasshopper under
**Armature → USD Export**. Components are grouped here as:

- [Authoring](#authoring) — links, joints, sensors, frames, filters
- [Libraries](#libraries) — preset pickers (right-click to choose)
- [Assembly, Validation, and Export](#assembly-validation-and-export)

Conventions used below:

- *Optional* inputs may be left unconnected.
- *Access* is the Grasshopper data access: **item** takes one value per
  solve, **list** takes a list (and most list components broadcast a
  single value across all items — see
  [arrayed-links.md](arrayed-links.md)).
- All lengths are meters, masses are kilograms, revolute angles are
  degrees, prismatic distances are meters, torques are N·m.

---

## Authoring

### Link Enhanced (`Link`)

Creates a robot link from mesh geometry, with mass properties and an
optional physics material. Mass, center of mass, and the full inertia
tensor are computed from the mesh automatically.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Name | Name | Text | list | — | No |
| Mesh | Mesh | Mesh | list | — | No |
| Link Frame | Frame | Plane | list | world origin | Yes |
| Material | Mat | Physics material (from Material Library) | item | — | Yes |
| Visual Material | VisMat | Visual material (from Visual Material) | item | — | Yes |
| Mass Override | Mass | Number (kg) | item | — | Yes |
| Collision Mesh | ColMesh | Mesh | item | visual mesh | Yes |
| Use Link Frame | UseFrame | Boolean | item | `False` | Yes |
| Color | Color | Colour | list | — | Yes |
| Angular Damping | AngDamp | Number | list | `0.0` | Yes |
| Static | Static | Boolean | item | `False` | Yes |
| Collision Fidelity | Fidelity | Text | item | `auto` | Yes |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Link | Link | Link object(s) — connect to Joint or Assembly |
| Info | Info | Mass, volume, COM, inertia, material summary per link |

**Nuances**

- **Mass priority:** if a Material is connected, mass = volume ×
  density and the Mass Override input is **ignored**. With no material,
  the Mass Override is used; with neither, mass defaults to 1.0 kg.
- **List behavior:** Name and Mesh are lists. With one name and many
  meshes, links are auto-named `name_0`, `name_1`, … — this is the
  intended pattern for arrayed parts (wheel rollers, etc.).
- **Inertia:** the full inertia tensor (including off-diagonal terms)
  is computed from the **collision mesh** via volume mass properties.
  This requires a **closed** mesh; open meshes fall back to a
  bounding-box approximation (which over-estimates inertia 2–3× for
  non-box shapes) and the Info output says so.
- **Static flag:** a static link is exported as a static collider with
  **no rigid body and no mass** — it is pinned to the world. Use it for
  a fixed chassis or test rig. Do *not* mark a link static if it is the
  child of a driven joint: the drive has no rigid body to push and is
  silently ignored (the Validator warns about this). Static is also how
  you satisfy the SimReady root-pinning rule for stationary robot types
  (Manipulator / End Effector).
- **Collision Fidelity** accepts: `auto` (default), `bounding_box`,
  `bounding_sphere`, `convex_hull`, `convex_decomposition`,
  `mesh_simplification`, `none`. Guidance:
  - gripper jaws and contact-critical links → `convex_decomposition`
    (a convex hull bridges the gap between jaws and the gripper can't
    close around anything)
  - bulky chassis → `bounding_box` or `convex_hull` for speed
  - `none` and `mesh_simplification` (exact triangle-mesh collision)
    are only valid on **static** links — PhysX rejects exact-mesh
    collision on dynamic bodies at load time. The Validator warns.
  - unrecognized tokens silently fall back to `auto`.
- **Use Link Frame:** when true and a Link Frame plane is supplied, the
  mesh is rebased into the link's local frame and a link transform is
  exported. When false, the mesh is exported in world coordinates.
- **Color:** if no Visual Material is connected, a connected Color (or,
  failing that, the mesh's average vertex color) is exported as a
  simple visual material.
- Warnings are raised for very small volumes (almost always a units
  mistake — model in meters) and for open meshes.

---

### Joint (`Joint`)

Connects a parent link and a child link with a revolute (rotating),
prismatic (sliding), or fixed joint.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Name | Name | Text | list | — | No |
| Parent Link | Parent | Link | list | — | No |
| Child Link | Child | Link | list | — | No |
| Joint Type | Type | Config (from Joint Type Library) | list | — | No |
| Joint Frame | Frame | Plane | list | — | No |
| Pose | Pose | Number (deg / m) | list | `0.0` | Yes |
| Motor | Motor | Drive config (from Joint Drive Library) | list | — | Yes |
| Lower Limit | Lower | Number | list | from joint type | Yes |
| Upper Limit | Upper | Number | list | from joint type | Yes |
| Use Limits | Limits | Boolean | list | from joint type | Yes |
| Joint Friction | Friction | Number (N·m) | list | see below | Yes |
| Transmission | Trans | Transmission (from Transmission) | item | — | Yes |
| Safety Profile | Safety | Safety profile (from Drive Safety Profile) | item | — | Yes |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Joint | Joint | Joint object(s) — connect to Assembly |
| Child Link | ChildOut | **Updated child link with parent reference set** |
| Info | Info | Joint configuration per joint |
| Parent Link | ParentOut | Pass-through of the (deduplicated) parent links |

**Nuances**

- **The child link is re-emitted.** The Joint clones the child link
  with its parent reference attached (and preserves inertia, static
  flag, collision fidelity, materials, etc.). **Always use the joint's
  `Child Link` output downstream** — in the Assembly's Links input and
  as the parent of the next joint. Wiring the original Link output
  instead means the kinematic chain is never recorded.
- **Joint Frame convention:** plane **origin** = the exact point of
  rotation/translation; plane **Z-axis** = the motion axis (rotation
  axis for revolute, slide direction for prismatic). Arbitrary axis
  directions are supported — the exporter orients the joint's local
  frames to match. See [plane-placement.md](plane-placement.md).
- **Motor overrides the joint type:** when a Motor is connected, its
  max force always wins, and its stiffness/damping replace the joint
  type's values whenever they are non-zero. This makes a generic
  `revolute` joint with an `hiwonder_lx225` motor behave like an
  LX-225 rather than like the joint-type default. The motor's max
  velocity and joint friction also carry through to the export.
- **Transmission:** if connected, the motor's drive parameters are
  reflected to the joint output side (torque ×ratio×η, speed ÷ratio,
  stiffness and damping ×ratio²) before export. One Transmission input
  applies to **all** joints produced by this component instance.
- **Joint Friction default:** revolute joints *without* a motor inherit
  the joint type's passive friction (e.g. the `continuous` preset adds
  light friction for free-spinning wheels). Joints with a motor use the
  motor's joint friction. The Joint Friction input overrides either.
- **Child angular damping:** if the child link authored its own angular
  damping it is kept; otherwise the joint type's child-angular-damping
  default applies (e.g. `continuous` adds 2.0 to damp free spinners).
- **Pose** is the joint's rest/preview value (degrees for revolute,
  meters for prismatic). It is used by the Assembly/Validator preview
  and exported as the authored rest state.
- **Lists broadcast:** a single joint type, motor, or frame is reused
  across all joints; a single Name with multiple joints auto-suffixes
  `name_0`, `name_1`, ….
- Fixed joints ignore limits and drive parameters (the Validator warns
  if you set them anyway).

---

### Mimic Joint (`Mimic`)

Creates a PhysX mimic relationship between two existing revolute
joints — the standard way to model coupled gripper jaws and parallel
linkages.

PhysX enforces `q_target + gearing × q_reference + offset = 0`,
equivalently `q_target = −(gearing × q_reference + offset)`.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Name | Name | Text | list | `mimic` | Yes |
| Target Joint | Target | Joint | list | — | No |
| Reference Joint | Reference | Joint | list | — | No |
| Gearing | Gear | Number | list | `1.0` | Yes |
| Offset | Offset | Number (deg) | list | `0.0` | Yes |
| Allow Target Motor | AllowMotor | Boolean | list | `False` | Yes |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Mimic Joint | Mimic | Mimic relationship(s) — connect to Assembly's Mimic Joints input |
| Info | Info | Equation and configuration per relationship |

**Nuances**

- **Gearing sign:** because of the PhysX sign convention above,
  same-direction coupling in joint coordinates usually needs
  **gearing = −1**, not +1.
- Both target and reference must be **revolute joints with limits** —
  anything else is rejected.
- **The target joint's motor is removed at export** unless Allow Target
  Motor is true. Under the SimReady Runnable profile, a kept motor on a
  mimic target gets its drive stiffness and damping force-zeroed in the
  USD (the spec requires it).
- The Validator checks mimic rest-pose consistency: if the target's
  authored rest pose disagrees with what the mimic equation predicts
  from the reference's rest pose, or a coupled joint rests at/near its
  limit, you get a warning — both situations are classic causes of
  gripper solver lock.

---

### Sensor (`Sensor`)

Attaches a sensor to a link. Pick the sensor model with a **Sensor Type
Library** component. See [sensors.md](sensors.md) for placement
conventions and the full preset list.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Name | Name | Text | item | — | No |
| Sensor Type | Type | Config (from Sensor Type Library) | item | — | No |
| Parent Link | Link | Link | item | — | No |
| Sensor Frame | Frame | Plane | item | — | No |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Sensor | Sensor | Sensor object — connect to Assembly |
| Info | Info | Type, parent, position, frequency, FOV, resolution |

**Nuances**

- **Frame convention:** plane origin = sensor position; plane
  **Z-axis = view/sensing direction**. For cameras, plane X = image
  right, and image-up is therefore **−Y** (right-handed optical
  convention). The exporter converts this to the USD-native camera
  frame automatically — point Z where the camera looks and it works.
- **Select the component to preview**: cameras draw their view
  direction, image-right/image-up arrows, and FOV frustum (with a tick
  marking the top edge of the image, so roll is unambiguous); lidars
  draw the spin axis, scan-zero ray, and scan arc; IMU/contact/F-T
  sensors draw a plain axes triad.
- The Sensor Type input must come from the Sensor Type Library
  component — raw text is rejected.

---

### Named Frame (`Frame`)

Authors a named coordinate frame on a link — TCP, end-effector, tool
flange, camera mount, calibration target. Exported as USD Xform prims
with semantic role labels so Isaac Lab, ROS 2 tf2, and MoveIt can
address them by name without hand-editing the USD.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Name | Name | Text | list | — | No |
| Parent Link | Parent | Link | list | — | No |
| Plane | Plane | Plane | list | world XY | No |
| Role Override | Role | Text | list | menu selection | Yes |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Named Frame | Frame | Frame object(s) — connect to Assembly's Named Frames input |
| Info | Info | Name, parent, role, origin, axes |

**Right-click roles:** `tcp` (default), `end_effector`, `tool_flange`,
`camera_mount`, `imu_mount`, `lidar_mount`, `calibration_target`,
`attachment`, `custom`.

**Nuances**

- The plane is given in **world space**; it is converted to link-local
  space at export.
- Typical hobby use: drop a plane on the gripper tip, name it `tcp`,
  pick the TCP role, done — Isaac sees it.

---

### Collision Filter (`ColFilter`)

Excludes one link pair (or one link versus a list of links) from
collision detection. Use it for non-adjacent links that legitimately
touch or overlap at rest.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Link 1 | Link1 | Link | item | — | No |
| Link 2 | Link2 | Link(s) | list | — | No |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Collision Filter | Filter | Filter object — connect to Assembly |
| Info | Info | The excluded pairs |

**Nuances**

- Parent–child joint pairs are **already excluded automatically** by
  the exporter and the Validator's collision check — you only need
  filters for other pairs (siblings, base-vs-arm, etc.).
- Self-pairs and duplicates are skipped with a warning.

---

### Collision Matrix (`ColMatrix`)

Bulk collision-filter authoring.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Group A | A | Links | list | — | No |
| Group B | B | Links | list | — | Yes |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Collision Filters | Filters | Generated filter objects — connect to Assembly |
| Info | Info | Pair count summary |

**Nuances**

- **Group B empty:** every unique pair *within* Group A is filtered —
  ideal for a cluster of mutually-overlapping parts.
- **Group B provided:** the full A × B cross-product is filtered.
- Duplicate links and A∩B self-pairs are skipped.

---

### Transmission (`Trans`)

Models the gearbox / belt / harmonic drive between motor and joint
output. A transmission is not just torque scaling — it reshapes the
whole drive:

```
torque_out    = ratio × efficiency × torque_motor
speed_out     = speed_motor / ratio
stiffness_out = ratio² × stiffness_motor
damping_out   = ratio² × damping_motor
```

A 5:1 belt on a hobby servo means 25× effective output stiffness — the
simulator needs to know that.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Motor | Motor | Drive config | item | — | Yes |
| Ratio Override | Ratio | Number | item | from preset | Yes |
| Efficiency Override | Eta | Number 0–1 | item | from preset | Yes |
| Reflected Inertia | Jrefl | Number (kg·m²) | item | `0` | Yes |
| Backlash | Lash | Number (deg / m) | item | `0` | Yes |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Transmission | Trans | Transmission object — connect to Joint's Transmission input |
| Motor (geared) | MotorOut | The motor preset with the transmission already applied |
| Info | Info | Ratio, efficiency, and geared-output preview |

**Right-click presets:** `direct`, `belt_2to1`, `belt_3to1`,
`belt_4to1`, `gear_pair_5to1`, `gear_pair_10to1`, `planetary_27to1`,
`harmonic_50to1`, `harmonic_100to1`, `harmonic_160to1` — values in
[presets.md](presets.md).

**Nuances**

- Two equivalent wirings: connect the **Transmission** output to the
  Joint's Transmission input (the joint applies the math), **or**
  connect a Motor here and wire the **Motor (geared)** output to the
  Joint's Motor input directly. Don't do both for the same joint.
- Ratio 0 is an error; efficiency is clamped to [0, 1].
- **Backlash is informational only** — PhysX joints have no native
  backlash. It is recorded in the export for downstream tooling.

---

### Drive Safety Profile (`Safety`)

Per-joint safety envelope: acceleration/jerk caps, soft position stops,
and an e-stop torque ceiling. Authored as USD metadata that Isaac Lab
safety filters can read, so a policy can be clamped to the hardware
envelope from the first training epoch.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Max Acceleration | MaxA | Number (rad/s² or m/s²) | item | `0` = unset | Yes |
| Max Jerk | MaxJ | Number (rad/s³ or m/s³) | item | `0` = unset | Yes |
| Soft Lower Limit | SoftLo | Number (deg / m) | item | unset | Yes |
| Soft Upper Limit | SoftHi | Number (deg / m) | item | unset | Yes |
| E-Stop Torque | EStop | Number (N·m) | item | `0` = use joint MaxForce | Yes |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Safety Profile | Safety | Profile object — connect to Joint's Safety input |
| Info | Info | Which limits are set |

**Nuances**

- Everything left at 0 / unset means "no limit authored" — the sim runs
  at PhysX defaults. Only the fields you set are written to the USD.
- Soft lower must be strictly less than soft upper (error otherwise).

---

### Joint Calibration (`JointCal`)

Adds a calibration offset and an optional home position to joints.
This is how you make the simulated joint angle agree with the real
robot's encoder reading.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Joint | Joint | Joint(s) | list | — | No |
| Calibration Offset | Offset | Number (deg / m) | list | `0.0` | Yes |
| Home Position | Home | Number (deg / m) | list | — | Yes |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Joint | Joint | Updated joint(s) |
| Info | Info | Offset and home per joint |

**Nuances**

- The offset is added to the authored joint limits and the home target
  at export.
- Setting a Home Position also sets the joint's rest pose, so the
  preview and the exported rest state match.
- The component is additive — only joints passed through it are
  affected.

---

### Link Dynamics Override (`DynOverride`)

Force-set mass, center of mass, diagonal inertia, or angular damping on
links where the mesh-derived values aren't trustworthy (open meshes,
parts with internal hardware, etc.).

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Link | Link | Link(s) | list | — | No |
| Mass | Mass | Number (kg) | list | — | Yes |
| Center Of Mass | COM | Point (link-local, doc units) | list | — | Yes |
| Diagonal Inertia | Inertia | Vector (Ixx, Iyy, Izz) | list | — | Yes |
| Angular Damping | AngDamp | Number | list | — | Yes |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Link | Link | Updated link(s) |
| Info | Info | The applied overrides |

**Nuances**

- **The diagonal inertia override clears the CAD-derived full inertia
  tensor**, so your explicit values actually win at export (the writer
  otherwise prefers the full tensor). If you override inertia, you are
  fully responsible for all three diagonal values.
- All other sim fields (static flag, collision fidelity, materials,
  full inertia when *not* overridden) pass through untouched.
- Like Joint Calibration, this is additive — only links passed through
  it are affected.

---

## Libraries

Library components carry a preset database. **Right-click the component
to pick a preset**; the current selection is shown on the component
face. Optional inputs override individual fields of the selected
preset. All preset values are listed in [presets.md](presets.md).

### Material Library (`Material`)

Physics material: density (drives mass), static/dynamic friction, and
restitution.

**Inputs** (all optional overrides)

| Input | Nickname | Type | Default |
|---|---|---|---|
| Custom Density | Density | Number (kg/m³) | preset |
| Custom Friction | Friction | Number | preset — sets **both** static and dynamic |
| Custom Restitution | Restitution | Number 0–1 | preset |
| Custom Static Friction | StaticFrx | Number | preset |
| Custom Dynamic Friction | DynFrx | Number | preset |

**Outputs:** Material (`Mat`) — connect to Link Enhanced; Info.

**Right-click menu:** *Metals* (aluminum, steel, stainless_steel,
titanium, brass, copper), *Plastics* (abs_plastic, pla_plastic, nylon,
polycarbonate, acrylic, rubber, foam), *Gripper Pads* (tpu_rubber,
silicone_rubber), *Composites* (carbon_fiber, fiberglass), *Wood*
(wood_pine, wood_oak, plywood, mdf). Default: `aluminum`.

**Nuances**

- Friction override precedence: explicit static/dynamic overrides win;
  the legacy single Friction input sets both; otherwise the preset's
  split values apply.
- Dynamic friction is clamped to ≤ static friction (PhysX requires
  μk ≤ μs; a violation is non-physical and would be rejected anyway).
- The split matters: presets ship with μk ≈ 0.7–0.9 × μs. Setting them
  equal produces sticky-then-launch contact that makes grasping
  unreliable.

### Visual Material (`VisMat`)

PBR rendering appearance for links. No effect on physics.

**Inputs**

| Input | Nickname | Type | Default | Optional |
|---|---|---|---|---|
| Name | Name | Text | preset name | Yes |
| Color | Color | Colour | preset color | Yes |
| Metallic | Metal | Number 0–1 | `0.0` | Yes |
| Roughness | Rough | Number 0–1 | `0.5` | Yes |
| Opacity | Opacity | Number 0–1 | `1.0` | Yes |

**Outputs:** Visual Material (`VisMat`) — connect to Link Enhanced;
Info.

**Right-click menu:** *Metal Presets* (aluminum, steel, brass, copper)
and *Color Presets* (red, blue, green, yellow, black, white, gray,
rubber_black). Default: `gray`.

### Joint Type Library (`JointType`)

Joint kind + limits + baseline drive parameters. 16 presets covering
revolute, prismatic, and fixed joints — see [presets.md](presets.md).

**Inputs** (all optional overrides)

| Input | Nickname | Type | Default |
|---|---|---|---|
| Lower Limit | Lower | Number (deg / m) | preset |
| Upper Limit | Upper | Number (deg / m) | preset |
| Stiffness | Stiff | Number | preset |
| Damping | Damp | Number | preset |
| Max Force | MaxF | Number | preset |
| Use Limits | Limits | Boolean | `True` |
| Joint Friction | Friction | Number (N·m) | preset |
| Child Angular Damping | AngDamp | Number | preset |

**Outputs:** Joint Type Config (`Config`) — connect to Joint; Info.

**Right-click menu:** grouped *Revolute Joints*, *Prismatic Joints*,
*Fixed Joints*. Default: `revolute`.

**Nuances**

- Fixed joint types always have limits off, regardless of the Use
  Limits input.
- A connected Motor on the Joint overrides the joint type's
  stiffness/damping/max-force (see the Joint component).
- The `continuous` preset is the right choice for wheels and spinners:
  effectively unlimited range, no drive, light passive friction and
  child angular damping so it doesn't spin forever.

### Joint Drive Library (`Drive`)

USD drive (motor/servo) presets: peak torque, PD stiffness and damping,
max velocity, passive friction. Organized by hardware family because
you pick a servo by what's on the robot, not by control mode.

**Inputs** (all optional overrides)

| Input | Nickname | Type |
|---|---|---|
| Max Force | MaxF | Number (N·m) |
| Stiffness | Stiff | Number (N·m/rad) |
| Damping | Damp | Number (N·m·s/rad) |
| Max Velocity | MaxVel | Number (rad/s or m/s) |
| Joint Friction | JFric | Number (N·m) |

**Outputs:** Drive Config (`Drive`) — connect to Joint (or
Transmission); Info.

**Right-click menu:** *Hiwonder (JetAuto)* (hiwonder_lx225,
hiwonder_hx35h, hiwonder_lx15d), *Dynamixel* (dynamixel_xl430,
dynamixel_xm430), *Generic* (generic_hobby, generic_industrial),
*Control modes* (velocity_motor, torque_motor, passive). Default:
`hiwonder_lx225`.

**Nuances**

- The preset values are deliberately conservative and tuned to real
  hardware. An undertuned joint can be bumped per joint; an
  infinitely-stiff default kills sim-to-real, which is why there isn't
  one.
- `passive` is for free-spinning joints (Mecanum rollers): zero drive,
  light damping and friction.

### Sensor Type Library (`SensorType`)

Sensor presets: frequency, FOV, range, resolution. 27 presets including
hardware look-alikes (RealSense D435, Velodyne VLP-16, Hokuyo UST-10LX,
RPLidar A1, Ouster OS1, Kinect) — see [presets.md](presets.md).

**Inputs:** none — selection is purely via the right-click menu,
grouped *RGB Cameras*, *Depth Cameras*, *Semantic Cameras*, *2D
Lidars*, *3D Lidars*, *IMUs*, *Contact Sensors*, *Force/Torque
Sensors*. Default: `camera_rgb`.

**Outputs:** Sensor Type Config (`Config`) — connect to Sensor; Info.

### Robot Type Library (`RobotType`)

Emits one of the eight canonical SimReady robot-type tokens
(case-sensitive): `End Effector`, `Manipulator`, `Humanoid`, `Wheeled`,
`Holonomic`, `Quadruped`, `Mobile Manipulators`, `Aerial`.

**Inputs:** none — right-click to select. Default:
`Mobile Manipulators`.

**Outputs:** Robot Type (`Type`) — connect to the Assembly's Robot Type
input; Info.

**Nuances**

- Required by the SimReady **Runnable** profile; optional otherwise.
- The token interacts with the root-pinning rule: `Manipulator` and
  `End Effector` require the root link to be **Static** (pinned);
  all mobile types require it **not** static. See
  [simready.md](simready.md).

### SimReady Profile Library (`SimReady`)

Emits a canonical SimReady conformance-profile token.

**Inputs:** none — right-click to select: **None** (no declaration),
**Robot-Body-Neutral**, **Robot-Body-Runnable** (default).

**Outputs:** SimReady Profile (`Profile`) — connect to both the Robot
Validator's and Assembly Export's SimReady Profile inputs; Info.

---

## Assembly, Validation, and Export

### Assembly (`Assembly`)

Collects links, joints, sensors, mimic joints, named frames, and
collision filters into one robot assembly. Supports merging
sub-assemblies and previewing the posed robot in the viewport.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Assemblies | Asm | Assembly objects to merge | list | — | Yes |
| Links | Links | Links | list | — | Yes |
| Joints | Joints | Joints | list | — | Yes |
| Sensors | Sensors | Sensors | list | — | Yes |
| Collision Filters | ColFilter | Filters | list | — | Yes |
| Name | Name | Text | item | `Assembly` | No |
| Preview Pose | Preview | Boolean | item | `True` | No |
| Mimic Joints | Mimic | Mimic relationships | list | — | Yes |
| Named Frames | Frames | Named frames | list | — | Yes |
| Robot Type | Type | Text (SimReady token) | item | `""` | Yes |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Assembly | Assembly | Assembly object — connect to Validator / Export / Attach |
| Info | Info | Counts, robot type, duplicate-name warnings |

**Nuances**

- **Use the joints' Child Link outputs** in the Links input so parent
  references survive (see the Joint component).
- Robot Type is validated against the eight allowed SimReady tokens —
  the comparison is **case-sensitive** (`"wheeled"` is not
  `"Wheeled"`). Use the Robot Type Library component to avoid typos.
- When merging sub-assemblies, the first non-empty Robot Type wins.
- The Info output flags duplicate link and joint names — fix them;
  joints, filters, sensors, and mimics all resolve targets **by name**.
- The viewport preview applies each joint's authored Pose, resolving
  mimic relationships, so you see the robot in its rest state.

### Assembly Attach (`Attach`)

Attaches child assemblies (or single links) to a parent assembly with a
joint defined in the parent's frame — the way to compose, say, a
gripper assembly onto an arm assembly, or one wheel assembly onto a
chassis four times.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Parent Assembly/Link | Parent | Assembly or Link | item | — | No |
| Child Assembly/Link | Child | Assembly(s) or Link(s) | list | — | No |
| Parent Link | ParentLink | Link or exact link name | item | — | No |
| Child Link | ChildLink | Link(s) or exact link name(s) | list | — | No |
| Joint Type | Type | Config (from Joint Type Library) | item | — | No |
| Joint Frame | Frame | Plane(s), in the parent assembly frame | list | — | No |
| Pose | Pose | Number(s) | list | `0.0` | Yes |
| Joint Name | Name | Text | list | `attach_joint` | Yes |
| Motor | Motor | Drive config(s) | list | — | Yes |
| Namespace Child Instances | Namespace | Boolean | item | `True` | Yes |
| Lower Limit | Lower | Number(s) | list | from joint type | Yes |
| Upper Limit | Upper | Number(s) | list | from joint type | Yes |
| Use Limits | Limits | Boolean(s) | list | from joint type | Yes |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Assembly | Assembly | Combined assembly with the attach joint(s) |
| Info | Info | Attachment summary |
| Parent Link | ParentLink | Resolved parent link (pass-through for cleaner graphs) |
| Parent Link Name | ParentName | Resolved parent link name |

**Nuances**

- **Namespace Child Instances** (default on) prefixes each attached
  child instance's link and joint names. Leave it on whenever you
  attach the *same* assembly more than once — otherwise every instance
  has duplicate names and the export is blocked.
- Parent Link / Child Link accept either the link object or its exact
  name as text.

### Robot Validator (`Validate`)

Validates the assembly and previews animated joint motion in the
viewport. Errors block export; warnings are advisory.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Links | Links | Links | list | — | Yes |
| Joints | Joints | Joints | list | — | Yes |
| Animation | Anim | Number 0–1 | item | `0.5` | No |
| Joint Values | JVals | Numbers 0–1, per joint | list | — | Yes |
| Show Axes | Axes | Boolean | item | `True` | No |
| Show Limits | Limits | Boolean | item | `True` | No |
| Axis Scale | Scale | Number | item | `0.1` | No |
| Collision Filters | ColFilter | Filters | list | — | Yes |
| Check Collisions | Collide | Boolean | item | `False` | No |
| Isolate Joint | IsoJoint | Text (name or index) | item | — | Yes |
| Isolate | Iso | Boolean | item | `False` | No |
| Assembly | Assembly | Assembly object (**recommended**) | item | — | Yes |
| SimReady Profile | SimReady | Text | item | `""` | Yes |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Valid | Valid | True when there are no errors |
| Report | Report | Full validation report |
| Joint States | States | Current pose per joint |
| Preview Meshes | Meshes | Transformed meshes at the current animation state |
| Collisions | Collisions | Detected collision pairs |
| Errors | Errors | Error list for on-canvas inspection |
| Warnings | Warnings | Warning list |
| SimReady | SimReady | Per-requirement conformance checklist (see [simready.md](simready.md)) |

**What it checks**

- *Kinematic graph:* duplicate names, missing parent/child links,
  multiple parents, cycles, exactly one root link, disconnected links.
- *Sim fidelity:* non-positive or sub-gram masses (unit mistakes),
  missing/implausible inertia (including the rigid-body triangle
  inequality), μk > μs, restitution out of range, static links in the
  middle of a chain, unsafe collision-fidelity tokens on dynamic
  bodies, legacy-class (≥1e5) stiffness, missing max force, inverted
  limits, >±360° revolute ranges, heavily overdamped drives.
- *Mimic sanity:* rest-pose consistency and rest-near-limit warnings.
- *Visual material consistency:* the same material name with different
  RGB values across links is an error.
- *Document units:* warns when the Rhino document isn't in meters.
- *Rest collisions* (when Check Collisions is on): unfiltered pairs
  overlapping at rest are reported; overlapping **sibling** pairs are
  errors (add Collision Filter / Matrix coverage).
- *SimReady checklist* when a profile is set.

**Nuances**

- **Connect the Assembly input** — it is the authoritative source and
  the Links/Joints pins are then ignored (with a warning if both are
  wired). The pin inputs exist for quick partial checks.
- With nothing wired to Animation or Joint Values, the preview shows
  the robot at its **authored rest poses** (including resolved mimics).
  Wire Animation (0–1) to sweep all joints lower→upper, or Joint Values
  for per-joint control; Isolate Joint + Isolate animates one joint
  while holding the rest.
- The collision check excludes parent–child pairs automatically and
  honors your collision filters.

### Assembly Export (`Export`)

Validates and exports the assembly: writes a self-contained package
folder (USD file, JSON description, and OBJ mesh assets) and runs the
bundled USD converter automatically.

**Inputs**

| Input | Nickname | Type | Access | Default | Optional |
|---|---|---|---|---|---|
| Assembly | Assembly | Assembly object | item | — | No |
| Export Path | Path | Text — file or folder path | item | — | No |
| Robot Name | Name | Text | item | `Robot` | No |
| Export | Export | Boolean — set True to export | item | `False` | No |
| Physics FPS | FPS | Integer | item | `120` | Yes |
| Solver Position Iterations | PosIter | Integer | item | `16` | Yes |
| Restitution Cap | RestCap | Number 0–1 | item | `0.10` | Yes |
| SimReady Profile | SimReady | Text | item | `""` | Yes |

**Outputs**

| Output | Nickname | Description |
|---|---|---|
| Success | Success | True if export succeeded |
| Info | Info | Validation preview, or export paths + converter output |

**Nuances**

- **With Export = False** the component runs a dry-run validation and
  the Info output reports `VALID - ready to export` or the blocking
  issues. Use this as a pre-flight check.
- **Path handling:** a folder path (or trailing slash) produces
  `<folder>\<RobotName>.usda`; a path without extension gets `.usda`
  appended; `.usd` and `.usda` are both accepted. The export creates a
  **folder** named after the file containing the USD, a JSON package,
  and an assets/meshes directory — keep the folder together if you move
  the export.
- **SimReady Profile** accepted tokens (case-insensitive): `""` /
  `none`, `neutral` / `Robot-Body-Neutral`, `runnable` /
  `Robot-Body-Runnable`. Unknown values warn and are treated as none.
  See [simready.md](simready.md) for what each profile enforces.
- Export is **blocked** by unresolved references: missing/duplicate
  link or joint names, joints or sensors pointing at links that aren't
  in the assembly, invalid meshes, mimic targets/references that don't
  exist or aren't limited revolute joints, and zero-range revolute
  joints without a drive (use a fixed joint instead).
- Mimic-target joints have their motor stripped at export unless the
  mimic's Allow Target Motor flag is set.
- **Physics FPS** (min 30) and **Solver Position Iterations** (min 1)
  are authored into the USD physics scene. **Restitution Cap** clamps
  the restitution of all exported robot materials — bouncy robot parts
  destabilize contact, so the default cap is 0.10.
- Joint and sensor prim names are sanitized (non-alphanumeric → `_`)
  and uniquified for USD; the original names are preserved in the JSON.
- Conversion runs the bundled converter; nothing else needs to be
  installed. If conversion fails, the Info output contains the full
  converter log.
