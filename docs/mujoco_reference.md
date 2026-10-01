
---

# MuJoCo Fundamentals — Tutorial

## What MuJoCo actually is

MuJoCo is a **physics engine** — a C library that, given a description of rigid bodies, joints, and contacts, computes how they move under forces (gravity, actuation, collisions) over time. It does not know about XML natively; XML (MJCF) is just the *authoring format* you write, which gets compiled into an internal binary representation the engine actually runs on.

Two separate objects you'll see in every script:

- **`mjModel`** — the STATIC description: masses, geometry, joint types, everything you wrote in the XML, compiled into efficient arrays. Created once via `MjModel.from_xml_path(...)`. Doesn't change while simulating.
- **`mjData`** — the DYNAMIC state: current joint positions/velocities, contact forces, everything that changes every timestep. Created via `mujoco.MjData(model)`, mutated by `mujoco.mj_step(model, data)`.

Analogy: `mjModel` is the blueprint of a car (fixed), `mjData` is the car's current speedometer/odometer/engine-temp reading (constantly changing as you drive). You load the blueprint once; you read/write the dashboard every instant.

## The simulation loop

Everything in MuJoCo ultimately boils down to this pattern:

```python
model = mujoco.MjModel.from_xml_path('scene.xml')  # load blueprint, once
data = mujoco.MjData(model)                          # create dashboard, once

for step in range(num_steps):
    data.ctrl[:] = my_action                         # write actuator commands
    mujoco.mj_step(model, data)                       # advance physics by one timestep
    frame = render(data)                               # read camera/state out
```

Your `scripts/record_episode.py` (Phase 9) will be exactly this loop, with camera rendering and HDF5 writing added inside it. Everything before that phase — all the XML — exists purely to define what `model` contains.

## MJCF tag family — what each one is for

| Tag | Purpose | Where you've used it |
|---|---|---|
| `<mujoco>` | Root element, one per file | every file |
| `<compiler>` | Load-time settings: angle units, auto-limits, mesh directory | `angle="radian"` in scene_base.xml |
| `<option>` | Physics engine settings: gravity vector, timestep, solver type | not yet used — default gravity (0,0,-9.81) is fine for us |
| `<worldbody>` | The root body, origin of your whole coordinate system | contains everything |
| `<body>` | A rigid object; has a `pos`/`quat` relative to its PARENT | robot_tower, working_table |
| `<geom>` | Gives a body actual shape — visual AND collision by default (can split via `contype`/`conaffinity` later) | box primitives so far, mesh geoms coming in Phase 4 |
| `<joint>` | Connects a body to its parent with a degree of freedom (hinge, slide, free, ball) | not yet used — comes with the UR10 import, 6 hinge joints |
| `<site>` | A labeled reference point with NO physics/collision — used for attaching sensors, cameras, or marking locations | will use for camera mount points, gripper TCP marker |
| `<camera>` | A viewpoint that can be rendered — can be fixed (child of worldbody) or attached to a moving body (child of a robot link, for eye-in-hand) | Phase 7 — D405 wrist cam, ZED third-person |
| `<light>` | Illumination source for rendering | 3 lights added in Phase 3 |
| `<asset>` | Container for reusable resources: `<mesh>`, `<material>`, `<texture>` definitions, referenced by name elsewhere | will hold UR10 mesh file declarations |
| `<actuator>` | Defines how you CONTROL a joint — motor, position servo, velocity servo | needed once robot joints exist, to actually drive them |
| `<sensor>` | Readable physics quantities — joint position/velocity, force, touch | optional, useful for logging eef_pose cleanly |
| `<equality>` | Constraints tying bodies/joints together without a real joint — e.g. gripper finger mimicking | will likely need this for gripper mechanism |
| `<default>` | Shared attribute templates applied to multiple elements, reduces repetition | not yet used — worth adopting once the file gets larger |

## Why each repo file contributes to "the MuJoCo environment"

- **`frame_convention.yaml`** — not read by MuJoCo at all. It's YOUR ground truth; you manually transcribe its numbers into XML `pos` attributes. MuJoCo has no concept of this file.
- **`scene_base.xml`** — THIS is what MuJoCo actually loads. Everything else is either a source MuJoCo reads (meshes) or documentation for humans (yaml, md files).
- **`assets/meshes/*.stl`** — referenced BY FILENAME from `<mesh file="...">` tags inside the XML's `<asset>` block. MuJoCo opens these at load time to get triangle geometry. Without a `<mesh>` declaration pointing at it, a sitting STL file does nothing — it has to be referenced.
- **`scripts/*.py`** — the only thing that makes the model DO something over time. A `.xml` file alone is a static description; nothing moves until a Python script loads it and calls `mj_step()` in a loop.

## Mental model summary

MJCF describes **what exists and where** (geometry, joints, connectivity). It does NOT describe **what happens over time** — that's entirely the job of your Python script calling `mj_step()` repeatedly, reading/writing `mjData` each iteration. A gorgeous, perfectly-dimensioned XML scene that no script ever loads and steps is just an unused blueprint — the actual "simulation" only exists while a script is running.

---

# MuJoCo Fundamentals — Tutorial

## What MuJoCo actually is

MuJoCo is a **physics engine** — a C library that, given a description of rigid bodies, joints, and contacts, computes how they move under forces (gravity, actuation, collisions) over time. It does not know about XML natively; XML (MJCF) is just the *authoring format* you write, which gets compiled into an internal binary representation the engine actually runs on.

Two separate objects you'll see in every script:

- **`mjModel`** — the STATIC description: masses, geometry, joint types, everything you wrote in the XML, compiled into efficient arrays. Created once via `MjModel.from_xml_path(...)`. Doesn't change while simulating.
- **`mjData`** — the DYNAMIC state: current joint positions/velocities, contact forces, everything that changes every timestep. Created via `mujoco.MjData(model)`, mutated by `mujoco.mj_step(model, data)`.

Analogy: `mjModel` is the blueprint of a car (fixed), `mjData` is the car's current speedometer/odometer/engine-temp reading (constantly changing as you drive). You load the blueprint once; you read/write the dashboard every instant.

## The simulation loop

Everything in MuJoCo ultimately boils down to this pattern:

```python
model = mujoco.MjModel.from_xml_path('scene.xml')  # load blueprint, once
data = mujoco.MjData(model)                          # create dashboard, once

for step in range(num_steps):
    data.ctrl[:] = my_action                         # write actuator commands
    mujoco.mj_step(model, data)                       # advance physics by one timestep
    frame = render(data)                               # read camera/state out
```

Your `scripts/record_episode.py` (Phase 9) will be exactly this loop, with camera rendering and HDF5 writing added inside it. Everything before that phase — all the XML — exists purely to define what `model` contains.

## MJCF tag family — what each one is for

| Tag | Purpose | Where you've used it |
|---|---|---|
| `<mujoco>` | Root element, one per file | every file |
| `<compiler>` | Load-time settings: angle units, auto-limits, mesh directory | `angle="radian"` in scene_base.xml |
| `<option>` | Physics engine settings: gravity vector, timestep, solver type | not yet used — default gravity (0,0,-9.81) is fine for us |
| `<worldbody>` | The root body, origin of your whole coordinate system | contains everything |
| `<body>` | A rigid object; has a `pos`/`quat` relative to its PARENT | robot_tower, working_table |
| `<geom>` | Gives a body actual shape — visual AND collision by default (can split via `contype`/`conaffinity` later) | box primitives so far, mesh geoms coming in Phase 4 |
| `<joint>` | Connects a body to its parent with a degree of freedom (hinge, slide, free, ball) | not yet used — comes with the UR10 import, 6 hinge joints |
| `<site>` | A labeled reference point with NO physics/collision — used for attaching sensors, cameras, or marking locations | will use for camera mount points, gripper TCP marker |
| `<camera>` | A viewpoint that can be rendered — can be fixed (child of worldbody) or attached to a moving body (child of a robot link, for eye-in-hand) | Phase 7 — D405 wrist cam, ZED third-person |
| `<light>` | Illumination source for rendering | 3 lights added in Phase 3 |
| `<asset>` | Container for reusable resources: `<mesh>`, `<material>`, `<texture>` definitions, referenced by name elsewhere | will hold UR10 mesh file declarations |
| `<actuator>` | Defines how you CONTROL a joint — motor, position servo, velocity servo | needed once robot joints exist, to actually drive them |
| `<sensor>` | Readable physics quantities — joint position/velocity, force, touch | optional, useful for logging eef_pose cleanly |
| `<equality>` | Constraints tying bodies/joints together without a real joint — e.g. gripper finger mimicking | will likely need this for gripper mechanism |
| `<default>` | Shared attribute templates applied to multiple elements, reduces repetition | not yet used — worth adopting once the file gets larger |

## Why each repo file contributes to "the MuJoCo environment"

- **`frame_convention.yaml`** — not read by MuJoCo at all. It's YOUR ground truth; you manually transcribe its numbers into XML `pos` attributes. MuJoCo has no concept of this file.
- **`scene_base.xml`** — THIS is what MuJoCo actually loads. Everything else is either a source MuJoCo reads (meshes) or documentation for humans (yaml, md files).
- **`assets/meshes/*.stl`** — referenced BY FILENAME from `<mesh file="...">` tags inside the XML's `<asset>` block. MuJoCo opens these at load time to get triangle geometry. Without a `<mesh>` declaration pointing at it, a sitting STL file does nothing — it has to be referenced.
- **`scripts/*.py`** — the only thing that makes the model DO something over time. A `.xml` file alone is a static description; nothing moves until a Python script loads it and calls `mj_step()` in a loop.

## Mental model summary

MJCF describes **what exists and where** (geometry, joints, connectivity). It does NOT describe **what happens over time** — that's entirely the job of your Python script calling `mj_step()` repeatedly, reading/writing `mjData` each iteration. A gorgeous, perfectly-dimensioned XML scene that no script ever loads and steps is just an unused blueprint — the actual "simulation" only exists while a script is running.
