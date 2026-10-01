
## PHASE 4 — Actuator & Stability Tuning (addendum)

**Issues found and fixed, in order:**
1. Joints drooped under gravity — added `<actuator>` block (position actuators, one per joint)
2. Wrist joints jerked/oscillated violently — root cause was default Euler integrator unstable with stiff gains; fixed via `<option integrator="implicitfast" timestep="0.002"/>`. CONFIRMED by direct A/B test: Euler oscillates, implicitfast stable, all else equal.
3. wrist_2 settled at steady-state error (~-1 rad) under gravity despite actuator — root cause: proportional-only position control always has nonzero steady-state error against a constant disturbance (gravity torque); fixed via `gravcomp="1"` on all 6 moving bodies, canceling gravity's effect so actuators only fight the commanded target
4. Control sliders artificially capped below true joint range — `autolimits="true"` on `<compiler>` only auto-enforces JOINT limits, does not auto-set actuator `ctrlrange`; added explicit `ctrlrange` matching each joint's range to every `<position>` actuator
5. Elbow joint was capped at ±180° (ROS-Industrial software default for collision avoidance) — verified against official UR10 datasheet: ALL 6 joints are ±360° hardware capability. Unlocked elbow to full ±360°.
6. Per user preference: shoulder_pan and shoulder_lift intentionally restricted to ±180° (mirrors a real, configurable PolyScope "Joint Limits" safety profile — not arbitrary, matches actual UR safety-config options). All other joints at full ±360°. NOTE: update these to match real lab PolyScope config once robot is physically configured — cheap one-line change.

**Final actuator gains (tuned, not default):**
- shoulder_pan, shoulder_lift: kp=2000, kv=200
- elbow: kp=1500, kv=150
- wrist_1/2/3: kp=150, kv=30 (reduced from initial kp=500 due to oscillation on low-inertia links)

**Key concept — position actuator steady-state error:** a proportional(-derivative) position actuator never perfectly cancels a constant disturbance like gravity; it settles at the point where restoring force equals disturbance force. For robot arms, use `gravcomp="1"` rather than only increasing gains (which reintroduces instability).
