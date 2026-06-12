# Troubleshooting

Quick fixes for the most common problems, roughly in the order you'll
hit them. The **Robot Validator** catches most of these before export —
run it first and read its Report output.

## Installation / plugin loading

**Components don't appear under the Armature tab (Rhino 8).**
Rhino 8 runs plugins on the modern .NET runtime by default; if the
toolbar is missing or components fail to load, run the
`SetDotNetRuntime` command in Rhino, choose **NETFramework**, and
restart Rhino. (Rhino 7 is unaffected.)

**Components appear but error immediately.**
Re-install via `_PackageManager` and restart Rhino. The bundled USD
converter is part of the package — no separate install is needed.

## Units and mass

**Link shows "Mass: 1.000 kg" when it shouldn't.**
1. No material connected — connect a **Material Library** output (the
   material *object*, not a text panel with a name).
2. Document units not meters — the Validator warns about this
   explicitly.
3. Mesh not closed — run `_ShowEdges` in Rhino and repair naked edges;
   open meshes give approximate volume and no real inertia tensor.

**Masses or volumes are absurdly large/small.**
Almost always a unit mismatch. The pipeline assumes the Rhino document
is in **meters**. Go to *File → Properties → Units → Meters* and
rescale the geometry (or `Scale` by 0.001 from mm). Every value shown
on Info panels before export is in raw document units, so a mm document
poisons everything you read.

**"mass ... is below the 0.001 kg PhysX stability floor".**
Sub-gram bodies destabilize the solver. Usually the same unit mistake;
otherwise raise the mass via override or merge tiny parts into their
parent link.

## Validation errors

**"Duplicate link name" / "Duplicate joint name".**
Every link and joint name must be unique — joints, mimics, sensors, and
collision filters all resolve their targets *by name*. When attaching
the same sub-assembly multiple times with Assembly Attach, keep
**Namespace Child Instances** on.

**"Joint references non-existent parent/child link".**
The link wired into the joint isn't in the Assembly's Links input, or
its name changed. Remember to wire the **joint's Child Link output**
(not the original link) into the Assembly.

**"Expected exactly one root link, found N" / "disconnected from the
root kinematic chain".**
Exactly one link must have no parent joint. Stray links you forgot to
joint into the tree, or links wired from the wrong output, cause this.

**"Link ... is marked Static but is the child of a joint".**
A static link has no rigid body, so a drive on its parent joint has
nothing to push — the joint silently does nothing in sim. Static is for
the root chassis / fixed rigs only.

**"Visual material ... has inconsistent RGB values across links".**
The same visual-material name is being created with different colors in
different places. Use one Visual Material component per appearance and
wire it everywhere it's used.

## Export problems

**Export does nothing.**
The **Export** input must be set to True (use a Boolean toggle). With
Export = False the component only runs the dry-run validation — read
the Info output; it lists exactly what blocks the export.

**"Export blocked due to unresolved references".**
The Info output lists each problem (missing links, duplicate names,
invalid meshes, broken mimic references, zero-range revolute without a
drive). Fix them in order; they're all authoring-side.

**"USD conversion failed".**
The Info output contains the full converter log — read the last lines.
Common causes: the output folder isn't writable, the path contains
characters the filesystem rejects, or (with a SimReady profile
selected) the post-export audit found a MUST-level violation, named by
requirement ID. See [simready.md](simready.md).

**The export is a folder, not just a file.**
That's by design: the folder contains the `.usda`, a JSON description,
and the mesh assets it references. Keep the folder together when moving
or versioning exports.

## Behavior in Isaac Sim

**Robot explodes on load.**
The classics, in order of likelihood:
1. Overlapping collision geometry between unfiltered links — enable
   *Check Collisions* on the Validator and add **Collision Filter /
   Collision Matrix** coverage for every pair it reports at rest.
2. Drive stiffness too high for the physics timestep — anything in the
   legacy 1e5–1e6 range models an infinitely stiff actuator. Attach a
   **Joint Drive Library** preset; real hobby servos are ~30–220
   N·m/rad.
3. Non-physical material (μk > μs) or bad inertia (open meshes) — the
   Validator flags both.
4. Extreme mass ratios between adjacent links (keep below ~50:1; model
   parts hollow or override masses).
You can also raise **Solver Position Iterations** on Assembly Export.

**Robot loads but doesn't move.**
- Drive stiffness/torque too low for the link weight — pick a stronger
  motor preset or raise Max Force.
- The chassis was marked **Static** and the immobile link is its child.
- Joint limits are zero-range, or the joint type is `fixed`.

**Gripper passes through objects.**
The jaws' collision approximation is a convex hull, which bridges the
gap between the jaws — there is nothing to close around. Set
**Collision Fidelity = `convex_decomposition`** on the jaw links, and
use a gripper-pad material (`tpu_rubber` / `silicone_rubber`). See
[materials.md](materials.md).

**Gripper jaws lock up (mimic-coupled).**
Check the Validator's mimic warnings: a rest pose at/near a joint limit
or a target rest pose inconsistent with the mimic equation
(`q_target = −(gearing × q_ref + offset)`) causes solver lock. Also
remember same-direction coupling usually needs **gearing = −1**.

**Joint angle doesn't match the real encoder.**
Pass the joint through **Joint Calibration** and set the Calibration
Offset (and optionally a Home Position) so the simulated zero matches
the hardware zero.

**Joint rotates around the wrong axis / wrong direction.**
The joint plane's Z-axis is the motion axis. Flip the plane (`Flip` in
Rhino) for direction, re-place its origin if the link orbits an offset
point. See [plane-placement.md](plane-placement.md).

**Wheels spin freely instead of driving the robot.**
Mecanum/omni rollers shouldn't each be a driven joint. Use the
`continuous` joint type + `passive` drive on rollers and drive the
robot with a mobile-base controller in Isaac Lab — or mark the chassis
Static and treat it as a fixed test rig.

**Robot falls through the floor.**
Add a ground plane in Isaac Sim; also verify the links actually have
collision meshes (the Info output of Link Enhanced shows collision face
counts).

**Materials/colors don't show.**
Connect a **Visual Material** to the link, make sure Isaac Sim isn't in
wireframe mode, and add a light to the scene.

## Sensors

See the troubleshooting table in [sensors.md](sensors.md) — black
camera images, missing sensors, and lidar-with-no-returns are almost
always frame-orientation issues, and the Sensor component's viewport
preview shows you exactly where it points.
