# SimReady Conformance

Armature can validate and declare exports against NVIDIA's
[SimReady Foundation](https://nvidia.github.io/simready-foundation/2026.04.1/)
**Robot-Body** profiles (spec v2026.04.1, profiles pinned at 1.0.0).
A SimReady-conformant USD is structured the way NVIDIA's tooling (Isaac
Sim, Isaac Lab) expects a robot asset to look, so it loads predictably
in pipelines that consume SimReady robots.

## The two profiles

| Profile | What it means |
|---|---|
| **Robot-Body-Neutral** | The robot-body *structure* is conformant: correct units and hierarchy, mass on every dynamic body, triangulated meshes, valid joint graph, complete mimic definitions, and the conformance declaration in the USD. No runnable-physics requirements. Recommended baseline. |
| **Robot-Body-Runnable** | Everything Neutral has, **plus** runnable-physics requirements: every non-fixed joint carries a drive or mimic (passive joints automatically receive an inert drag drive at export), a max velocity on every joint, a declared **Robot Type**, root pinning that matches that type, no unfiltered rest collisions, and `IsaacRobotAPI` authored on the robot root. MUST-level checklist failures **block export**. |

If you don't select a profile, nothing changes — no checklist runs and
no declaration is authored.

> Note on wording: exports *validate against* the SimReady profiles and
> *declare* conformance. There is no such thing as third-party
> "SimReady certification".

## How to enable it

Three wiring points:

1. **SimReady Profile Library** component — right-click, pick
   *Robot-Body-Neutral* or *Robot-Body-Runnable*. Connect its output to
   **both**:
   - the **Robot Validator**'s *SimReady Profile* input, and
   - the **Assembly Export**'s *SimReady Profile* input.

   (You can also type the token directly: `neutral`, `runnable`,
   `Robot-Body-Neutral`, `Robot-Body-Runnable` — case-insensitive.
   Empty or `none` disables it; unknown values warn and are treated as
   none.)

2. **Robot Type Library** component — right-click, pick the robot
   category, connect to the **Assembly**'s *Robot Type* input. The
   eight allowed tokens (case-sensitive):
   `End Effector`, `Manipulator`, `Humanoid`, `Wheeled`, `Holonomic`,
   `Quadruped`, `Mobile Manipulators`, `Aerial`.
   Robot Type is **required** for Runnable, optional for Neutral.

3. Export as usual. The Validator gives you the pre-flight checklist;
   the export runs a post-write audit on the actual USD.

## The Validator checklist

With a profile selected, the Robot Validator's **SimReady** output (and
a `=== SimReady (<profile>) ===` section in the Report) lists one line
per requirement:

```
[PASS] RC.008 RobotType 'Mobile Manipulators' is in allowed list
[FAIL] BA.001 expected 1 root link, found 2
[POST] VG.021 all faceVertexCounts == 3 (writer triangulates)
[N/A]  BA.002 enable Check Collisions to evaluate
[ADVISORY] DJ.006 joint 'lift' stiffness 12000 outside guidance [100,10000]
```

What the tags mean:

| Tag | Meaning |
|---|---|
| `[PASS]` | Requirement verified at assembly level. |
| `[FAIL]` | Requirement violated. Under **Runnable**, MUST-level failures are added to the validation errors and block export. |
| `[POST]` | Cannot be predicted in Grasshopper — it is guaranteed and verified by the exporter's post-write audit (units, triangulation, collision purpose, schema application, etc.). Listed so the checklist is complete. |
| `[N/A]` | Not applicable in the current configuration (e.g. the rest-collision check needs *Check Collisions* enabled; Robot Type is optional under Neutral). |
| `[ADVISORY]` | In spec range but outside the recommended guidance band — review, not a failure. |

A summary line closes the section, e.g.
`SimReady Robot-Body-Runnable: 14 pass, 0 fail, 6 post-export`.

### Requirements checked at assembly level

| ID | Requirement | Profile |
|---|---|---|
| UN.001 / UN.002 | Stage in meters and kilograms (writer guarantees — POST) | both |
| HI.004 | Robot under a single default prim (POST) | both |
| RB.001 / RB.007 | Every non-static link has mass > 0 | both |
| RB.MB.001 | At least 2 non-static links | both |
| RB.010 | Collision meshes use `guide` purpose (POST) | both |
| VG.021 | All mesh faces triangulated (POST) | both |
| JT.001–003 | Joints reference existing links, single targets | both |
| DJ.011 | Joint graph is acyclic | both |
| DJ.004 | Every non-fixed joint has a drive or is a mimic target (passive joints get an inert drag drive at export) | Runnable |
| DJ.005 | Every non-fixed joint authors a max velocity > 0 (omitted values default to 100 deg/s revolute / 1 m/s prismatic, reported as POST) | Runnable |
| DJ.006 | Drive stiffness in [~1, 1e5] and damping in [0.01, 1e4]; advisory outside guidance bands [100, 10000] / [1, 100] | both |
| DJ.007 | Mimic completeness: reference exists, gearing ≠ 0, no mimic chains, both joints limited | Runnable |
| BA.001 | Exactly one root link | both |
| BA.002 | No unfiltered rest collisions between non-adjacent links (requires *Check Collisions* on) | Runnable |
| RC.003 | Assembly name is lowercase + underscores (advisory) | Runnable |
| RC.008 | Robot Type present and in the allowed list | Runnable (FAIL if missing); Neutral (N/A if unset) |
| RC.009 | Root pinning matches the Robot Type (see below) | Runnable |
| DJ.001 / DJ.002 | Drive maxForce > 0, JointStateAPI consistent with drive targets (POST) | both |
| RC.007 | `IsaacRobotAPI` applied, SimReady metadata declared (POST) | both |

### The root-pinning rule (RC.009)

The robot's root link must match its declared type:

- **Stationary types** (`Manipulator`, `End Effector`): the root link
  must be **pinned** — set **Static = True** on the root link (Link
  Enhanced).
- **Mobile types** (`Wheeled`, `Holonomic`, `Quadruped`,
  `Mobile Manipulators`, `Aerial`, `Humanoid`): the root link must
  **not** be static.

A mismatch is a Runnable export blocker.

## The export-time audit

When a profile is selected, the exporter re-opens the written USD and
audits it: stage units, triangulation, collision-mesh purpose, joint
state/drive consistency, mimic attribute completeness, drive zeroing on
mimic targets, `IsaacRobotAPI` and its joint/link relationships, root
pinning versus robot type, tree reachability, and the presence of the
conformance declaration. Any MUST-level failure fails the export with
the requirement ID in the error message; on success the converter log
(shown in Assembly Export's Info output) ends with a line like:

```
SimReady Robot-Body-Runnable v1.0.0: PASS (27 checks)
```

## What gets authored in the USD

- **Conformance declaration** in the root layer's `customLayerData`:

  ```
  SimReady_Metadata = {
      validation = {
          profile = "Robot-Body-Runnable",
          profile_version = "1.0.0",
      }
  }
  ```

- **`IsaacRobotAPI`** applied on the robot's default prim, with
  `isaac:namespace` (sanitized robot name), `isaac:robotType` (your
  Robot Type token, verbatim), and the `isaac:physics:robotJoints` /
  `isaac:physics:robotLinks` relationships in kinematic order, root
  first.
- **`PhysicsJointStateAPI`** on driven joints, with state positions
  consistent with the drive targets.
- Collision meshes authored with `purpose = guide`; all meshes
  triangulated; mimic joints carry complete PhysX mimic attributes
  (natural frequency and damping ratio included); mimic-target drives
  are force-zeroed where the spec requires it.

## Roadmap

**Robot-Body-Isaac** (multi-layer payload composition) is planned but
not yet supported.
