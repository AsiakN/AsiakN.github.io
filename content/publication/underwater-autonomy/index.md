---
title: "Embodiment, Drift, and Long-Horizon Control in GPS-Denied Underwater Vehicles"
authors:
  - admin
date: "2025-09-01T00:00:00Z"

# Schedule page publish date (NOT publication's date).
publishDate: "2026-07-23T00:00:00Z"

# Publication type (CSL standard). `manuscript` = unpublished / in progress.
publication_types: ["manuscript"]

abstract: >-
  An underwater vehicle is an embodied agent stripped of the assumptions most
  autonomy research is built on. There is no global position fix, so state is
  dead-reckoned and error accumulates monotonically with the length of the task.
  The plant is buoyant, coupled, and poorly modelled, so the map from commanded
  intent to realised motion is itself uncertain. And the vehicle's morphology —
  where its actuators sit — bounds what any controller or policy on top of it can
  achieve. This work studies long-horizon execution under unbounded state
  uncertainty, control allocation as a property of embodiment, and the
  sensor-geometry errors that corrupt proprioception before estimation begins.
  The vehicle stack is proprietary; the control-allocation analysis tool is open
  source.

# Summary. An optional shortened abstract.
summary: Long-horizon execution under unbounded state drift, control allocation as embodiment, and proprioceptive error in GPS-denied underwater vehicles — ROS 2 / PX4 autonomy on a fielded ROV.

tags:
  - Embodied Agents
  - Long-Horizon Execution
  - State Estimation
  - Control Allocation
  - Robot Learning
featured: false

url_code: 'https://github.com/AsiakN/actuator_viz'
url_pdf: ''

image:
  caption: 'Tethered pool trial — depth-hold and traverse'
  focal_point: "Center"
  preview_only: false

projects: []
---

{{% callout note %}}
Active, ongoing work. The vehicle autonomy stack is proprietary and not publicly
available; this page states the problems and positions rather than linking to
source. The control-allocation analysis tool discussed in Thread 2 is open
source and linked above.
{{% /callout %}}

## Motivation

Most autonomy research assumes a bounded state estimate. A global fix arrives
periodically — GPS, a known landmark, a loop closure — and error, however large
it grows between fixes, is eventually collapsed. Nearly every guarantee about
long-horizon behaviour rests on that assumption somewhere.

Underwater, it is simply absent. Radio does not propagate, so there is no global
fix at any point in the mission. Position is obtained by integrating velocity
from a Doppler velocity log fused with inertial measurement, which means
**position error accumulates monotonically with mission duration and is never
corrected**. The horizon over which an agent can be trusted to execute is
therefore bounded not by its planner or its policy, but by the integral of its
estimator's error.

I find this the most honest available setting for long-horizon execution. In
simulation, and in most ground-robot work, drift is a nuisance that a
localisation module hides. Here it is the governing constraint, and every design
decision above it — control, mode structure, task specification — has to be made
in explicit acknowledgement that the agent's belief about where it is degrades
without bound.

The platform is a fielded underwater vehicle: an onboard compute module running
ROS 2, a flight-control unit executing the inner loops, and a serial link
between them. I work on the estimation, control, and mode architecture that sits
between operator intent and thruster output.

## Research questions

**RQ1 — What can be executed reliably when state error is unbounded and
uncorrected?** If the belief degrades monotonically, task horizon is a resource
to be budgeted. What does a planner or policy need to represent in order to
spend it well?

**RQ2 — How much of an agent's capability is fixed by its morphology rather
than its controller?** Actuator geometry determines which directions of motion
are achievable, how strongly they couple, and what survives a failure. This is
decided before any learning or control design begins, and it is rarely treated
as a variable.

**RQ3 — Where does proprioceptive error actually originate?** In practice, a
substantial share of estimator error is not sensor noise but unmodelled
geometry — frame conventions, mounting offsets, timing. These are corrections
that no amount of filtering recovers.

## Thread 1: Long-horizon execution under unbounded drift

The autonomy stack is organised as a hierarchy of modes of increasing
authority — direct actuation, attitude stabilisation, depth regulation, position
holding, waypoint following, and return-to-launch — each layered on a strictly
larger portion of the state estimate. This hierarchy is essentially a statement
about **how much drift each level of autonomy can tolerate**: the modes that
depend only on locally observable quantities such as depth and attitude remain
trustworthy indefinitely, while the modes that depend on integrated position
degrade with time in the water.

{{< video src="rov-pool-trial.mp4" controls="yes" >}}

*Tethered pool trial. The vehicle holds depth just below the surface while
traversing — the regime in which the locally observable part of the state
estimate (depth, attitude) is sufficient and integrated position is not yet
load-bearing. The tether is present for power and telemetry during bring-up;
the estimation and control problem it exists to test is the untethered one.*

Several findings have come out of building this:

- **Autonomy must degrade along the estimate, not along the mission.** When
  position quality falls, the correct behaviour is to fall back to a mode that
  does not depend on position, rather than to continue executing a plan with a
  belief that no longer supports it. Making that fallback explicit in the mode
  hierarchy is more valuable than making the estimator marginally better.
- **Stale state is more dangerous than noisy state.** A position estimate that
  arrives late but is treated as current produces confident motion toward a
  target that no longer exists — the failure looks like a controller instability
  but originates in timestamp handling. Freshness must be a first-class
  precondition on any position-dependent behaviour, not an assumption.
- **Discontinuities in belief are discontinuities in action.** When the
  estimator revises its own history, a position-holding controller sees an
  instantaneous error and responds violently to a change that reflects nothing
  physical. Any long-horizon controller needs an explicit policy for what to do
  when its own state estimate jumps.
- **The communication channel is part of the control loop.** The link between
  the compute module and the flight controller has finite bandwidth, and
  publishing estimates faster than it can carry them degrades everything sharing
  the link. Perception and estimation rates are a scheduling problem, not a
  quality dial — the same conclusion I reached from the opposite direction in
  the multi-camera work.
- **Feedforward encodes what the model already knows.** A vehicle that is not
  neutrally buoyant has a constant, known disturbance acting on it; supplying it
  directly rather than making the feedback loop rediscover it every cycle is a
  small change that materially improves regulation. The general principle —
  give the learned or tuned component only the residual the analytic model
  cannot explain — is the one I would carry into any policy-learning setting
  here.

## Thread 2: Control allocation as embodiment

Between a commanded motion in six degrees of freedom and the individual
actuators that must produce it sits the **effectiveness matrix**, determined
entirely by where the actuators are mounted and which way they point. Its
structure decides what the vehicle can do — and it is fixed at design time, in
CAD, usually by intuition or by copying an existing layout.

I treat this as an embodiment question rather than a control-engineering detail.
The properties that matter are:

- **Rank** — whether all six degrees of freedom are independently commandable at
  all, or whether the body has silently forfeited one.
- **Conditioning** — whether authority is distributed evenly, or whether one
  axis is so much weaker than the others that any controller above it inherits a
  badly scaled problem.
- **Coupling** — whether commanding one motion necessarily induces another, which
  determines how much cross-axis structure a policy has to learn before it can
  do anything useful.
- **Failure tolerance** — whether the remaining actuators still span the space
  after one is lost, which is the difference between graceful degradation and a
  vehicle that must surface.

To make these answerable before hardware exists, I wrote an open-source tool
that computes the effectiveness matrix from a declarative geometry
specification and reports controllability, conditioning, weak axes, redundancy,
and coupling — with a non-zero exit status when the layout is not fully
controllable, so a design regression can fail continuous integration like any
other defect.

The research position underneath it: **morphology is a design variable that
should be optimised jointly with control, and it is currently optimised by
neither.** A learned policy on a badly conditioned body is compensating for a
decision that could have been made correctly for free, months earlier. The
natural continuation is to search actuator layouts against the demands of the
task rather than validating a layout someone has already committed to.

## Thread 3: Proprioception before estimation

A recurring result from field work is that the errors that matter most are
geometric and conventional rather than stochastic. Three that I have written
about publicly, because they are general and badly documented:

- **Frame conventions and transformation chains.** Every measurement is
  meaningless until attributed to a frame, and the transformations between
  sensor, body, and world frames are where a large share of integration errors
  actually live. See
  [Coordinate Frames and Transformations in Robotics](/post/coordinate-frames/).
- **Orientation representation.** The choice between Euler angles, rotation
  matrices, and quaternions is not aesthetic — it determines whether the
  estimator has singularities and what it costs to run on constrained onboard
  compute. See
  [Why Localization in Robotics Needs Quaternions](/post/robot-quaternions/).
- **The lever-arm effect.** An inertial sensor mounted away from the centre of
  rotation measures centripetal and tangential accelerations that the vehicle's
  centre of mass never experiences, and reports them as real motion. The
  estimator is not wrong; it is being told about a different point on the body.
  See [Why Your Robot's IMU Is Lying to You](/post/lever-arm-effect/).

The reason I group these as a research thread rather than as engineering
folklore: they are all cases where **the agent's model of its own body is
wrong**, and no amount of filtering, fusion, or learning on top recovers what a
correct kinematic model would have given for free. For any embodied learning
system, self-model error is a systematic bias in the training signal itself, and
it deserves to be treated as such rather than absorbed into noise.

## Toward robot learning in this setting

The stack is entirely hand-designed — analytic estimation, tuned gains, an
allocation matrix derived from geometry. That is the right starting point, and
it makes the boundary where learning should enter unusually legible.

**Where the analytic model runs out.** Hydrodynamic drag, added mass, and
thruster interaction are the terms that are genuinely hard to model and that
tuning currently absorbs. Learning the residual on top of an analytic model,
rather than a policy from scratch, is the formulation that fits this
domain — and it inherits the feedforward principle from Thread 1.

**Sim-to-real where the sim is weakest.** Underwater dynamics are exactly the
regime in which simulators are least trustworthy, and the fielded vehicle
produces logged trajectories continuously. This is a domain-adaptation problem
with an unusually honest reality check attached.

**Learning to spend the horizon.** If drift bounds how long an agent can be
trusted, then a policy for a long mission has to reason about its own
uncertainty budget — when to commit to a long traverse, when to re-anchor
against a known feature, when to fall back. That is a long-horizon planning
problem in which the state uncertainty is the primary cost, and it is the
direction I am most interested in pursuing.

## Relation to the multi-camera work

The [other theme](/publication/current-research/) studies an agent that
perceives and reasons over long horizons but does not act. This one studies an
agent that acts continuously but perceives almost nothing beyond its own motion.
The constraint they share is that a hard onboard budget — compute, bandwidth,
power — decides what the agent is permitted to know, and the interesting
question in both is how autonomy should degrade when that budget cannot supply
what the task assumes.

## Open problems

- Characterising how far a task can be executed as a function of accumulated
  estimator error, rather than treating drift as a fault to be suppressed.
- Deciding when to spend mission time re-anchoring against a known reference
  instead of progressing — an explicit exploration/exploitation trade on the
  uncertainty budget.
- Jointly optimising actuator layout and controller, rather than fixing the
  first and tuning the second.
- Detecting self-model error — mounting offsets, timing skew, frame
  mistakes — automatically from logged motion, rather than through field
  diagnosis.

## Availability

The vehicle autonomy stack is proprietary and not public. The control-allocation
analysis tool is open source and linked above. Design notes and findings are
published here as they become shareable.
