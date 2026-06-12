# Material Workflow

How material-based mass calculation works, how the friction model
affects grasping, and how to pick or customize materials.

## Why material-based mass

Connect a **Material Library** preset to a **Link Enhanced** component
and the link's mass is computed automatically:

```
mass = mesh volume × material density
```

Benefits:

- **Realistic masses** from your actual geometry.
- **Design iteration** — change the material, the mass updates.
- **Consistent contact physics** — friction and restitution match the
  real material.
- The center of mass and the full inertia tensor are computed from the
  same geometry, so the dynamics are self-consistent.

If you connect a Material, the Link's Mass Override input is ignored.
With no material, the override is used; with neither, mass defaults to
1.0 kg (and the Validator will likely complain).

> **Units:** the pipeline assumes the Rhino document is in **meters**.
> Densities are kg/m³; a millimeter-unit document makes every link
> billions of times too heavy before correction and corrupts the
> numbers you read off the Info panels. Set *File → Properties → Units
> → Meters* and rescale.

---

## The friction model

Every material carries **two** friction coefficients:

- **Static friction (μs)** — resistance just before slipping starts.
- **Dynamic friction (μk)** — resistance while sliding.

Real materials have μk ≈ 0.7–0.9 × μs, and PhysX uses both. Authoring a
single value for both (the naive approach) produces "sticky-then-launch"
contact: the object holds, breaks loose violently, and shoots out of
the gripper. The shipped presets all carry a realistic split.

PhysX requires μk ≤ μs; the Material Library clamps your overrides to
enforce it, and the Validator flags violations.

**For grippers**, use the *Gripper Pads* presets:

| Preset | μs | μk | When |
|---|---:|---:|---|
| `tpu_rubber` | 1.20 | 0.95 | 3D-printed TPU jaw pads (Shore 95A) |
| `silicone_rubber` | 1.40 | 1.05 | Cast silicone pads — maximum grip |

Pair them with `Collision Fidelity = convex_decomposition` on the jaw
links so the collision shape actually has a gap to close around
objects.

---

## Picking materials by part

| Part | Material | Why |
|---|---|---|
| Chassis / frame | `aluminum` | Structural, light |
| Heavy base | `steel` | Mass keeps the robot planted |
| Arm links | `carbon_fiber` | Low inertia = faster, stabler arm |
| Wheels / tires | `rubber` | High friction for traction |
| Gripper jaws / pads | `tpu_rubber` or `silicone_rubber` | Grasp-rated friction |
| Printed brackets / shells | `abs_plastic` or `pla_plastic` | Matches the print |
| Transparent covers | `acrylic` or `polycarbonate` | Realistic density |
| Padding | `foam` | Very light, absorbs contact |

Full table with every value: [presets.md](presets.md#physics-materials).

### Typical desktop-robot masses (sanity check)

- Total robot: 3–8 kg
- Chassis: 2–3 kg
- Wheel: 0.1–0.2 kg
- Arm link: 0.15–0.3 kg
- Gripper: 0.05–0.1 kg total

Red flags: a single link over 10 kg (units or solid-vs-hollow problem),
a total under 0.5 kg (open meshes → near-zero volume), a desktop robot
over 20 kg.

---

## Hollow parts

Mass is volume × density, and most real parts are not solid. Options:

1. **Model the part hollow** (best — inertia is also correct).
2. **Reduce the density**: use the Custom Density override, e.g.
   aluminum at 900 kg/m³ approximates a honeycomb/walled part at one
   third of solid mass.
3. **Override the mass** directly: pass the link through **Link
   Dynamics Override** with the measured mass — useful when the real
   part contains motors and electronics the mesh doesn't show. Note
   that overriding only the mass keeps the mesh-derived inertia shape;
   override the diagonal inertia too if precision matters.

---

## Custom materials

Use the override inputs on Material Library:

```
right-click → aluminum
Custom Density   = 2500     (modified alloy)
Custom Static Friction  = 0.5
Custom Dynamic Friction = 0.4
```

The legacy single **Custom Friction** input sets both coefficients at
once — fine for quick tests, but prefer the split for anything that
grasps or rolls.

Density references: kg/m³ = g/cm³ × 1000 (aluminum 2.7 g/cm³ → 2700
kg/m³).

---

## One material per link

A link has exactly one physics material. If a part genuinely mixes
materials (rubber tire on an aluminum hub), split it into two links
joined by a `fixed` joint, each with its own material.

Visual appearance is separate: connect a **Visual Material** for
color/metallic/roughness without affecting physics.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Mass shows 1.0 kg | No material connected, no override | Connect a Material Library output (the object, not a text name) |
| Mass wildly too high/low | Document units not meters | Fix units, rescale geometry |
| Volume ≈ 0, "open mesh" warning | Mesh not closed | `_ShowEdges` in Rhino, cap/repair naked edges |
| Gripper drops or launches objects | μs == μk, or low-friction jaw material | Use a Gripper Pads preset; keep the static/dynamic split |
| "dynamic friction > static" error | Override mistake | Lower the dynamic value — PhysX requires μk ≤ μs |
| Restitution warning | Value outside 0–1 | Choose a value in range; also note Assembly Export caps robot restitution (default 0.10) |
