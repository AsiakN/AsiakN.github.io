---
title: "Grounded Perception and Long-Horizon Reasoning in Fielded Multi-Camera Systems"
authors:
  - admin
date: "2025-01-01T00:00:00Z"

# Schedule page publish date (NOT publication's date).
publishDate: "2026-07-23T00:00:00Z"

# Publication type (CSL standard). `manuscript` = unpublished / in progress.
publication_types: ["manuscript"]

abstract: >-
  A fielded multi-camera monitoring system is a passively embodied agent: it
  perceives a physical environment continuously, under a hard compute and
  bandwidth budget, and must convert raw sensory input into events that support
  reasoning over hours rather than frames. This work studies three coupled
  problems in that setting — grounding an open-ended event taxonomy without
  per-deployment supervision, allocating a fixed perception budget between local
  inference and transmitted observation, and sustaining autonomy in embedded
  agents deployed without operator supervision. Each is a degenerate case of a
  problem in embodied agency, distinguished mainly by the absence of actuation.
  Implementation is proprietary and not publicly available.

# Summary. An optional shortened abstract.
summary: Grounding open-ended event taxonomies, allocating a fixed perception budget, and sustaining autonomy in embedded agents — studied in fielded multi-camera systems, framed toward embodied agents and long-horizon execution.

tags:
  - Embodied Agents
  - Long-Horizon Reasoning
  - Grounded Perception
  - Robot Learning
  - Edge Computing
featured: true

# No `url_code` on purpose — the implementation is proprietary, and Wowchemy
# only renders a button for keys that have a value.
url_pdf: ''

image:
  caption: 'Dense detection, sparse events, and the long reasoning horizon above them'
  focal_point: "Center"
  preview_only: false

projects: []
---

{{% callout note %}}
Active, ongoing work carried out professionally. The implementation is
proprietary and not publicly available, so this page states the problems,
positions, and open questions rather than linking to source. Updated as the
work progresses.
{{% /callout %}}

## Motivation

An embodied agent is usually characterised by the loop it closes: perceive,
reason, act. It is worth asking what remains when the last term is removed —
when a system perceives a physical environment continuously and reasons over
what it sees, but does not move.

The setting I work in is exactly that: fixed camera arrays deployed in
warehouses and fulfillment centres, observing a partially observable, non-
stationary physical environment populated by people and machines, under a
compute and bandwidth budget fixed by the hardware bolted to the wall. The
system must decide what in its sensory stream is worth representing, hold that
representation over hours, and produce reasoning a human operator can act on.

I take this to be a legitimate laboratory for embodied agency rather than a
detour from it. Every constraint that makes robot learning hard in the field is
present — partial observability, distribution shift across deployments, a hard
real-time compute ceiling, no labelled data for the environment you are actually
in — with actuation removed and therefore with the credit-assignment problem
sharply simplified. What is left is the perception and long-horizon reasoning
substrate that an actuating agent would need anyway.

## Research questions

**RQ1 — Can an open-ended event taxonomy be grounded without per-deployment
supervision?** Every site has a different notion of what constitutes a
significant event. Collecting labels per deployment does not scale. What is the
right division of labour between a fixed perceptual front-end and a
language-conditioned semantic layer, and what does that division cost in
reliability?

**RQ2 — How should a fixed perception budget be allocated between local
inference and transmitted observation?** An embodied system cannot perceive
everything. The choice of what to process locally, what to transmit, and what
to discard is a resource-allocation problem that determines the agent's
effective perceptual field.

**RQ3 — What does an agent need in order to remain autonomous once fielded?**
Deployed embedded agents lose connectivity, are installed by non-experts, and
must recover without intervention. Autonomy here is a systems property, not a
policy-learning one, and it bounds everything above it.

## Thread 1: Grounding an open-ended taxonomy

The perceptual front-end — continuous object detection over the video stream —
is not the bottleneck. Detection in a structured indoor environment is close to
saturated. The bottleneck is **semantic**: mapping a stream of detections onto
the set of situations a particular operator considers actionable, when that set
differs per deployment and is not known in advance.

The position I have taken is to separate the two levels explicitly. A fast
detector runs continuously and is deployment-invariant. A language model
performs the semantic classification, conditioned on a natural-language
specification of what matters at that site. The consequence is that the event
taxonomy lives in configuration rather than in weights, and extending it is a
specification change rather than a retraining cycle.

This is essentially the argument for language as a task-specification interface
that motivates much recent work in robot learning, evaluated in a setting where
the specifications are written by domain experts who are not ML practitioners
and where the cost of a misclassification is borne immediately. Three findings
so far:

- **Specifications behave like production configuration, not like prompts.**
  They require versioning, staged rollout, and provenance. Every derived event
  carries the specification revision that produced it, which makes evaluation
  retrospective — a regression can be attributed to a revision after the fact
  rather than requiring a controlled run.
- **Semantic drift is the dominant failure mode**, not perceptual error. The
  detector is stable; what degrades is the mapping from detections to meaning
  as the environment changes around a fixed specification.
- **Statelessness in the inference layer is worth its cost.** The perceptual
  and semantic stages hold no environment state; the world model is external
  and authoritative. This makes the reasoning layer replaceable and scalable,
  at the price of requiring explicit context delivery to it.

## Thread 2: The perception budget

An embodied agent's perceptual field is not given by its sensors — it is given
by what it can afford to process. This system makes that trade explicit, and
the design decisions transfer directly to any robot with an onboard compute
ceiling.

- **Transformation is more expensive than transport.** Re-encoding sensory data
  on a small board consumes the frame budget outright. Passing observations
  through in their native representation wherever downstream stages tolerate it
  is almost always correct.
- **Local persistence decouples degradation from loss.** With uplink treated as
  best-effort and local storage as the durability layer, a connectivity failure
  degrades the latency of reasoning rather than destroying the observation.
  This is the same argument for onboard buffering in intermittently connected
  robots.
- **Attention must be actively allocated.** Observations nobody is consuming
  should not occupy the budget, which converts perception into a lifecycle
  problem — acquiring, sharing, and releasing perceptual resources under
  competing demands.
- **Sensor heterogeneity is an unmodelled cost.** A large fraction of the
  engineering is establishing what a given sensor actually emits versus what it
  advertises. Embodied systems papers rarely price this in; fielded ones cannot
  avoid it.

## Thread 3: Autonomy as a systems property

The unit of deployment is not a program but a **complete device image**,
flashed onto hardware installed by someone with no technical training and no
console access. The agent must bring itself from cold boot to a fully
participating node with no operator input.

This decomposes into a bootstrapping sequence with a genuinely agentic
structure: acquire a communication channel through an out-of-band modality when
the primary one is unavailable; establish a unique identity by exchanging a
shared credential for an individual one, which is what makes a single image
safely replicable across a fleet; then remain deliberately inert until assigned
a role and a location by the coordinating layer.

That last property is the one I find most transferable. **A provisioned agent
with no assigned task should do nothing, and this must be legible as correct
behaviour rather than as failure.** The largest practical cost in the field was
not any of the hard technical problems but the absence of an unambiguous way to
distinguish "idle because unassigned" from "broken".

Beneath this sits conventional systems work with a bearing on safety
argumentation: least-privilege execution with privileged operations brokered
through explicit policy, event-driven rather than polled state reconciliation,
and strict pinning of the base platform image. On the last point — a kernel
revision in a later base image silently broke the out-of-band provisioning
path, so two builds of identical source behaved differently in the field
depending only on when the base was obtained. For any fielded learning system,
the platform is part of the experimental configuration and has to be recorded
as such.

## Toward embodied agents and long-horizon execution

The direction I want to take this is the obvious one: the system currently
reasons over hours of observation but acts on none of it. Three questions
follow.

**Temporal abstraction over event sequences.** The aggregate reasoning layer —
occupancy, movement structure, anomaly — already operates over sequences rather
than frames, and its useful output is a natural-language summary rather than a
scalar. This is a long-horizon representation problem in a setting where ground
truth arrives sparsely and late, which is precisely the regime that makes
long-horizon robot learning hard.

**From detection to intervention.** The moment the system is permitted to act —
even weakly, by directing attention or requesting a different observation — the
credit-assignment problem returns, and the specification-as-configuration
approach has to answer for outcomes rather than only classifications.

**Multi-agent perception.** A site is already a set of agents with overlapping,
partial views and a shared world model. Coordinating what each attends to,
given a shared budget, is a multi-agent resource-allocation problem I have so
far solved by policy rather than by learning.

## Open problems

- Evaluating a language-specified taxonomy without ground truth, where the
  specification and the evaluation criterion are the same artefact.
- Detecting semantic drift automatically, rather than through operator
  complaint.
- Deciding what belongs in weights versus in specification — the boundary is
  currently drawn by engineering convenience, not by principle.
- Attributing outcomes to specification revisions once the agent's behaviour
  affects what it subsequently observes.

## Relation to the underwater autonomy work

The [companion theme](/publication/underwater-autonomy/) inverts this setting:
an agent that acts continuously but perceives almost nothing beyond its own
motion, operating with no global position fix at all. Read together, the two
bracket the same question from opposite sides — how autonomy should degrade when
a hard onboard budget cannot supply what the task assumes.

## Availability

The implementation is proprietary and not public. Design notes and findings are
published here as they become shareable.
