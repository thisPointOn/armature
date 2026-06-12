# Getting Started — 15 minutes from Rhino to Isaac Sim

This walkthrough exports a simple two-link arm. The same workflow scales
to a full 6-DOF robot or a mobile manipulator.

## Prerequisites

1. **Rhino 7 or 8 (Windows)** with Grasshopper.
2. **Armature installed**: run the `_PackageManager` command in Rhino,
   search **armature**, click Install, restart Rhino. That's all — the
   USD converter ships bundled with the plugin, so there is nothing else
   to install.
3. A Rhino file with at least one **closed mesh** per link, with document
   units set to **meters**.

> **Critical: model in meters.** If your file is in millimeters or
> inches, the volume-to-mass calculation and the inertia tensor will be
> off by huge factors. The Robot Validator catches this with a
> "very small volume" warning and a document-units warning. Set units
> via *File → Properties → Units → Meters* and rescale your geometry.

All components live in Grasshopper under the **Armature → USD Export**
tab.

---

## Step 1 — Materials

Drop a **Material Library** component on the canvas. Right-click it →
*Metals* → `aluminum`.

That's it. The component now emits a physics material with density,
static friction, dynamic friction, and restitution. Mass for any link
that uses this material auto-computes from `volume × density`.

> **Tip:** if you have gripper jaws, add a second Material Library and
> right-click → *Gripper Pads* → `tpu_rubber`. Real grippers need high
> static friction (μs ≈ 1.2) with a proper static/dynamic split.

---

## Step 2 — Links

For each link, drop a **Link Enhanced** component:

- **Name**: e.g. `base_link`, `shoulder_link`
- **Mesh**: connect the Rhino mesh
- **Material**: connect the Material from Step 1

Optional but useful:

- **Use Link Frame**: if true, rebases the mesh into the link's local
  frame. Leave false for the simplest workflow.
- **Static**: check this on the chassis if you want it pinned to the
  world (e.g. a fixed test rig, or when skipping wheel articulation).
- **Collision Fidelity**: leave at `auto` for most links. Set
  `convex_decomposition` on gripper jaws. Set `bounding_box` on a bulky
  chassis where contact precision doesn't matter.

The **Info** output shows the calculated mass, center of mass, and full
inertia tensor (Ixx, Iyy, Izz, Ixy, Ixz, Iyz) — all computed from your
geometry automatically.

---

## Step 3 — Joints

For each joint:

1. Drop a **Joint Type Library** component. Right-click → pick a preset
   (`revolute` for a typical hobby-arm joint, `shoulder` for a
   high-torque base joint, `finger` for a gripper). The preset carries
   sensible default limits, stiffness, damping, and torque.
2. Drop a **Joint Drive Library** component. Right-click → *Hiwonder
   (JetAuto)* → `hiwonder_lx225` (or whatever motor you actually have).
   The preset carries stiffness, damping, peak torque, max velocity,
   and joint friction matched to that servo class.
3. Drop a **Joint** component:
   - **Name**: e.g. `shoulder_joint`
   - **Parent Link** + **Child Link**: the two Link Enhanced outputs
   - **Joint Type**: from step 1
   - **Joint Frame**: a Rhino **plane** at the joint location with its
     **Z-axis pointing along the motion axis**
   - **Motor**: from step 2

The Joint component re-emits the child link with the parent reference
attached. **Use the joint's `Child Link` output, not the original Link
output, for everything downstream.**

---

## Step 4 — Assemble and validate

Drop an **Assembly** component:

- **Links**: list of all links (use the joints' Child Link outputs so
  parent references are set; the root link comes straight from its Link
  component or the joint's Parent Link output)
- **Joints**: list of all joints
- **Name**: your robot name

Drop a **Robot Validator** component and connect the Assembly output to
its **Assembly** input. This runs the kinematic-graph checks (cycles,
disconnected links, missing parents, duplicate names) and a
sim-fidelity audit. Read the **Report** output.

Common messages and what to do:

| Message | What it means | Fix |
|---|---|---|
| `no inertia tensor authored` | Mesh wasn't closed; a bounding-box fallback will be used | Run `_ShowEdges` in Rhino, close the mesh |
| `mass X kg is below the ... floor` | Unit mismatch — mesh probably in mm | Scale mesh by 0.001 or fix document units |
| `dynamic friction > static friction` | Non-physical material | Adjust your Material overrides |
| `stiffness ... is in the legacy range` | No motor attached, an unrealistically stiff default is in play | Attach a Joint Drive Library preset |
| `revolute range exceeds +/-360` | You probably wanted continuous rotation | Switch joint type to `continuous` |
| `Rhino document is in mm, not meters` | The whole pipeline assumes meters | Change document units and rescale |

Fix all errors (they block export). Warnings are advisory.

---

## Step 5 — Export

Drop an **Assembly Export** component:

- **Assembly**: connect the validated assembly
- **Export Path**: e.g. `C:\robots\my_arm.usda` (a folder path also
  works — the file is named after the robot)
- **Robot Name**: e.g. `my_arm`
- **Export**: connect a Boolean toggle, set it to True

The component writes a self-contained export folder (USD file, JSON
package, and mesh assets) and runs the bundled converter automatically.
The **Info** output shows the output paths and a summary.

---

## Step 6 — Load in Isaac Sim

Open Isaac Sim and drag the `.usda` into the stage. The robot loads
with:

- correct masses and inertia tensors (visible in the Property panel)
- correct PD gains and torque limits per joint
- the static/dynamic friction split for contacts
- any named frames you authored, addressable from Isaac Lab

If you set Drive Safety Profiles or Named Frames, they are authored as
custom USD attributes that Isaac Lab can read directly from the stage.

---

## Where to go next

- **Add a gripper**: `gripper_slide` joint type + `tpu_rubber` material
  on the jaws + `convex_decomposition` collision fidelity. The gripper
  will actually grasp. See [materials.md](materials.md).
- **Add a Transmission**: drop a Transmission component between your
  motor and the Joint. Right-click → `belt_4to1` or `harmonic_50to1`.
  Output torque, speed, stiffness, and damping all rescale correctly.
- **Add sensors**: drop a **Sensor Type Library** + **Sensor**
  component. Right-click the library → `realsense_d435` or whatever you
  have. Connect a Rhino plane at the mount point. See
  [sensors.md](sensors.md).
- **Add a TCP**: drop a **Named Frame** component. Name it `tcp`, pick
  the `tcp` role from the right-click menu, parent it to your gripper,
  and place the plane at the actual tool tip.
- **Sim-to-real**: drop a **Drive Safety Profile** with your servo's
  acceleration/jerk caps. Train your policy inside the same envelope
  the hardware has.
- **SimReady conformance**: declare the Robot-Body-Neutral or
  Robot-Body-Runnable profile so the export is validated against
  NVIDIA's SimReady Foundation spec. See [simready.md](simready.md).
- **Every component in detail**: [components.md](components.md).
- **All preset values**: [presets.md](presets.md).
- **Something not working?** [troubleshooting.md](troubleshooting.md).
