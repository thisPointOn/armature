# Arrayed Links

How to create many identical links and joints — Mecanum wheel rollers,
multi-finger grippers, repeated leg segments — **without copying
components**. Grasshopper's list handling plus Armature's list inputs
do all the work.

## The idea

**Link Enhanced** and **Joint** take *lists*. Feed 8 meshes into one
Link Enhanced and you get 8 link objects out. Feed 8 child links and 8
joint planes into one Joint component and you get 8 joints. Inputs with
a single value (a material, a joint type, a motor, one name) broadcast
across all items.

Bonus: give Link Enhanced **one** name and several meshes, and the
links are automatically named `name_0`, `name_1`, … — same for Joint
names. No expression gymnastics needed for unique names.

## Worked example: Mecanum wheel (1 hub + 8 rollers)

### 1. Hub

```
Material Library (aluminum) ──► Link Enhanced
[hub mesh]                  ──► Mesh
"hub"                       ──► Name          → hub link
```

### 2. Roller geometry

Model **one** roller, mesh it, then polar-array it in Grasshopper
(count 8, 45° steps, around the hub axis). Result: a list of 8 meshes.

### 3. Roller links — one component

```
Material Library (rubber)   ──► Material
[8 roller meshes]           ──► Mesh
"roller"                    ──► Name
                                  → 8 links: roller_0 … roller_7
```

Check the Info output with a Panel: 8 entries, each with its own mass
and inertia.

### 4. Joint planes

Create one plane at the first roller's axis (origin on the roller's
spin axis, **Z along the roller's cylinder axis**), then polar-array it
with the same count and angle as the geometry. Result: 8 planes, each
matched to its roller.

### 5. Roller joints — one component

```
"roller_joint"          ──► Name           (auto: roller_joint_0 …)
hub link                ──► Parent Link    (broadcasts to all 8)
[8 roller links]        ──► Child Link
Joint Type Library      ──► Joint Type     (preset: continuous)
[8 joint planes]        ──► Joint Frame
Joint Drive Library     ──► Motor          (preset: passive)
```

Grasshopper matches the lists one-to-one: hub + roller_0 + plane_0 →
joint 0, and so on.

Preset notes for rollers:

- Joint type **`continuous`** — unlimited rotation with light passive
  friction and child angular damping, so rollers don't spin forever.
- Motor **`passive`** — no drive, just damping. (Or leave Motor
  unconnected; `continuous` carries passive friction on its own.)

### 6. Collect and assemble

```
hub link + [8 roller Child Link outputs]  → Merge → Flatten → Assembly Links
[8 joints]                                → Flatten         → Assembly Joints
```

**Use the Joint's Child Link output** for the rollers — that's the copy
carrying the parent reference (see
[components.md](components.md#joint-joint)).

If neighboring rollers' collision meshes touch at rest, add a
**Collision Matrix** with all 8 rollers in Group A (Group B empty) to
filter every roller-roller pair in one component.

## Scaling up: four wheels on a chassis

Build the wheel once as its own **Assembly** (hub + rollers + joints),
then attach it to the chassis four times with **Assembly Attach**:

- Parent: the chassis assembly; Child: the wheel assembly (wired four
  times, or as a list with four joint frames).
- **Keep "Namespace Child Instances" on** — it prefixes each instance's
  link/joint names so four copies of `hub` don't collide. Duplicate
  names block export.

Totals for a 4 × (1 hub + 8 rollers) robot: 37 links, 36 joints — and
the Grasshopper definition contains roughly six Armature components.

## Verifying

| Check | How |
|---|---|
| 8 links created | Panel on Link Enhanced Info → 8 entries |
| Unique names | Names read `roller_0` … `roller_7` |
| 8 joints created | Panel on Joint Info → 8 entries |
| Rollers placed correctly | Select the Joint component — all 8 frames preview in the viewport |
| Whole wheel animates correctly | Robot Validator + Animation slider |

## Common pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Only one roller/joint produced | A list got flattened to one item, or counts differ | Panel-check list lengths before each input |
| All rollers at the same position | Arrayed the links but not the meshes | Polar-array the *geometry* before Link Enhanced |
| Joints attached at one point | Forgot to array the joint planes | Array the plane with the same count/angle as the meshes |
| "Duplicate link name" at export | Same explicit name list reused | Supply one name and let auto-suffixing number them |
| Export blocked: sibling rest collisions | Adjacent rollers overlap at rest | Collision Matrix over the roller group |
