# Mujina Long-Term Contribution Roadmap

## Executive Summary

**The strongest recommended specialty is mining reliability, protocol correctness, and diagnostics.** Mujina needs to ensure that a miner which appears to be running is actually producing useful work, can explain failures, and can recover appropriately.

The highest-impact contribution areas are:

1. **Source health and recovery:** expose upstream faults, track accepted and rejected shares, and detect silent mining failures.
2. **Scheduler correctness:** prevent panics, overlapping work assignments, and wasted hashing as workers and jobs change.
3. **Failure-oriented testing:** build reusable simulated-pool scenarios that turn operational bugs into regression tests.
4. **Persistent configuration and runtime control:** extend the existing configuration/API proposal rather than introduce a competing design.
5. **Thermal control and efficiency:** develop safe control and tuning capabilities with real hardware validation.


This roadmap is an independent contribution assessment, not an approved maintainer roadmap. Existing PRs and issues should be checked before starting implementation.

## Assessment Basis and Project Maturity

The preceding review examined local code at `019d1b0916457bb7ecd7cb7e93fee63c0329f9a2`, which matched upstream main at review time, together with public issues, PRs, and project documentation. This document exports those findings; upstream status can change.

The project README describes:

- **Working:** Bitaxe Gamma and the CPU backend.
- **Landing:** EmberOne00 support.
- **Near-term targets:** installable Antminer S19 images, Libreboard, and broader commercial hardware support.

Source: [Mujina project status](https://github.com/256foundation/mujina#current-status).

The distinction between missing, partial, and in-progress functionality matters. Mujina already has Stratum reconnect logic, temperature monitoring, emergency shutdown, a scheduler, and an HTTP API. The opportunities below extend these foundations. They do not imply those systems are entirely absent.

## What Complete Bitcoin Mining Firmware Includes

| Area | Responsibilities |
|---|---|
| Mining protocols | Authenticate to pools, receive work, handle difficulty changes, submit shares, recover connections. |
| Work scheduling | Construct valid work, divide search space across chips, avoid duplication, retire obsolete jobs. |
| Hardware control | Discover ASICs, initialize chains, manage communication, clocks, voltage, and power sequencing. |
| Thermal protection | Read sensors, control fans, detect faults, throttle or shut down safely. |
| Efficiency | Find stable operating settings and meet power or hashrate targets. |
| Operations | Persist settings, expose diagnostics, support remote control and recovery. |
| Firmware delivery | Provide installable images, startup integration, upgrades, rollback, and recovery. |

Mujina's architecture separates boards, the mining core, and the API. That separation offers several ways to contribute without initially owning mining hardware. See the [architecture overview](https://github.com/256foundation/mujina/blob/main/docs/architecture.md).

## 1. Stratum Protocol and Stratum V2 Support

### Current state and gaps

Stratum V1 already has transient reconnect handling with jittered exponential backoff. The larger gap is how failures are classified, reported, and recovered from.

[Issue #113](https://github.com/256foundation/mujina/issues/113) describes cases where invalid upstream parameters, authorization failures, or malformed jobs can leave the miner running without useful work. Its proposed design distinguishes transient errors, deterministic faults, and per-message errors.

Native Stratum V2 is already being implemented in [PR #65](https://github.com/256foundation/mujina/pull/65). A separate implementation would duplicate active work.

### Contribution projects

- Validate upstream parameters during negotiation and job conversion.
- Implement agreed fault classification, slow retesting, and recovery behavior.
- Test disconnects during share submission and transitions between jobs.
- Help validate SV2 interoperability, share accounting, and protocol error handling.
- Develop pool failover after configuration and source lifecycle support are established.

### Longer-term Bitcoin infrastructure work

Explore integration with miner-selected block templates. SV2 mining support alone does not provide independent transaction selection: Job Declaration and Template Distribution are separate parts of the architecture.

Reference: [Stratum V2 protocol overview](https://stratumprotocol.org/specification/03-protocol-overview/).

**Hardware needs:** most early protocol work can use simulated pools and the CPU backend.

## 2. Work Scheduling and Mining Correctness

### Confirmed problems

- [Issue #114](https://github.com/256foundation/mujina/issues/114) documents a scheduler panic when the extranonce2 space is too small to divide among eligible threads.
- The reviewed scheduler contains a TODO noting that a newly eligible thread can receive a range overlapping existing assignments until the next job redistributes work.

Code reference: [scheduler at the reviewed revision](https://github.com/256foundation/mujina/blob/019d1b0916457bb7ecd7cb7e93fee63c0329f9a2/mujina-miner/src/scheduler.rs).

### Contribution projects

1. Handle insufficient search space gracefully, with explicit reporting when some workers cannot receive distinct work.
2. Preserve disjoint assignments when workers join, leave, pause, or resume.
3. Exercise job invalidation and late-arriving shares.
4. Investigate richer allocation across extranonce, permitted version bits, and time updates as needed.

### Success criteria

- Valid small search spaces cannot crash the scheduler.
- Worker lifecycle changes do not introduce unintended duplicate work.
- Tests demonstrate correct handling of obsolete work and late results.

**Learning value:** block headers, coinbase construction, merkle roots, targets, version rolling, and the conversion from pool jobs to ASIC work.

**Hardware needs:** little initially; real hardware validation becomes useful as behavior changes reach ASIC workers.

## 3. Telemetry, Monitoring, and Recovery

### Main gap

A live process and responding API do not prove that the miner is producing valid, accepted work. Operators need visibility into source, scheduler, and worker progress.

[Issue #113](https://github.com/256foundation/mujina/issues/113) proposes structured conditions, fault visibility, and accepted/rejected share reporting. This is the best candidate for a long-term contribution track.

### Contribution projects

- Expose source status and structured conditions with stable machine-readable reasons.
- Track accepted and rejected shares, including rejection reasons where available.
- Distinguish connection health from useful mining progress.
- Provide fault clearing and retesting consistent with the configuration/API design.
- Add watchdogs for ineffective mining and integrate them with recovery policies.
- Build monitoring integrations after the underlying telemetry is reliable and its contract is agreed.

### Design considerations

Share arrivals are probabilistic. A fixed rule such as “no accepted share in one minute means failure” can falsely flag a healthy low-hashrate miner or a source using high difficulty. Watchdogs should account for expected share intervals and changes in assigned work.

**Hardware needs:** most early work can be validated with CPU mining and simulated upstream behavior.

## 4. Configuration, Runtime Control, and Management Security

### Current state

Current main primarily configures the daemon through environment variables. The configuration module contains unimplemented loading functions; its descriptive comments do not establish working persistence or hot reload.

Existing work includes [issue #83](https://github.com/256foundation/mujina/issues/83) and [PR #88](https://github.com/256foundation/mujina/pull/88). The latter starts with a read-only, in-memory configuration tree based on the configuration/API proposal.

### Contribution projects

- Validate and apply runtime changes while mining.
- Save settings atomically and restore them after restart.
- Distinguish requested configuration from successfully applied state.
- Report and recover from partial reconfiguration failures.
- Manage multiple sources and priorities.
- Develop API access control and credential-handling conventions in coordination with maintainers.

The API documentation currently describes no authentication or encryption. This is a documented management gap, not a claim that a particular authentication design has been approved. See [API documentation](https://github.com/256foundation/mujina/blob/main/docs/api.md).

**Hardware needs:** low for configuration persistence and API work; some runtime hardware settings require board testing.

## 5. Thermal and Power Management

### Existing protection

The Bitaxe monitor already reads temperature and shuts down on thermal emergencies or repeated unusable readings. Thermal protection is therefore partial functionality to extend, not an entirely missing feature.

Closed-loop fan control is tracked by [issue #9](https://github.com/256foundation/mujina/issues/9), with an existing [PI-controller PR #33](https://github.com/256foundation/mujina/pull/33).

### Contribution projects

- Help test and complete the existing fan-control approach.
- Detect fan stalls and define appropriate responses.
- Model sensor faults and distinguish unavailable readings from safe temperatures.
- Add controlled throttling before emergency shutdown where supported.
- Define explicit fault states and recovery conditions across supported boards.

### Validation

Use simulated sensor failures to establish software behavior, followed by real board testing to establish control stability and effective protection. Simulation alone cannot validate physical thermal behavior.

**Hardware needs:** real hardware is essential for completing and validating this track.

## 6. Efficiency Tuning and Performance Profiles

### Opportunity

The review did not find a general autotuning subsystem in current main. Related work exists in the open BZM2 series, including [PR #70](https://github.com/256foundation/mujina/pull/70), which covers board support, calibration, and runtime tuning.

As a capability reference, mature firmware supports power and hashrate targets. See [Braiins configuration documentation](https://academy.braiins.com/braiins-os/configuration).

### Suggested implementation sequence

1. Establish trustworthy power, temperature, hardware-error, and hashrate measurements.
2. Support bounded manual operating-point changes.
3. Characterize stability repeatably across supported settings.
4. Implement automated tuning within hardware-specific limits.
5. Persist validated profiles and adapt to changing operating conditions.

### Measurement principles

- Optimize stable useful hashing per unit of energy, not clock speed alone.
- State whether a power measurement covers the ASIC core, board input, or whole miner at the wall.
- Evaluate rejected work, errors, thermal behavior, and sustained stability alongside raw hashrate.

**Hardware needs:** significant, including supported equipment, reliable measurements, and sustained test time.

## 7. Hardware Abstraction and Platform Support

### Existing architecture

Mujina already separates board composition, ASIC workers, peripherals, management protocols, and transport. Its hardware-interface traits enable peripheral testing with mocks. A new abstraction layer should only be introduced for a concrete need.

### Contribution opportunities

- Improve worker and board lifecycle handling as hardware connects or disconnects.
- Extend consistent diagnostics across supported boards.
- Add transport or board support with real-device validation.
- Contribute review and tests to active hardware work rather than duplicate it.

Relevant active work at review time includes:

- [PR #76: native Windows support](https://github.com/256foundation/mujina/pull/76).
- [Issue #58: Windows USB hotplug](https://github.com/256foundation/mujina/issues/58).
- [PR #68: board command and telemetry foundations](https://github.com/256foundation/mujina/pull/68).
- [PR #69: BZM2 ASIC support](https://github.com/256foundation/mujina/pull/69).
- [PR #70: BZM2 board support](https://github.com/256foundation/mujina/pull/70).
- [PR #71: BZM2 diagnostics and documentation](https://github.com/256foundation/mujina/pull/71).

**Hardware needs:** platform and device access are usually required to finish this work credibly.

## 8. Testing and Benchmarking

### Recommended project: reusable simulated-pool scenarios

Extend existing tests with a reusable harness that can intentionally:

- Reject authorization.
- Return invalid negotiation parameters.
- Send malformed or unusable jobs.
- Change difficulty and replace jobs.
- Disconnect during share submission.
- Remain connected while ceasing to provide useful work.

Each fixed failure should become a deterministic regression scenario. Coordinate with existing Stratum tests and the SV2 PR's integration harness before designing new infrastructure.

### Additional projects

- Scheduler tests for small search spaces and worker lifecycle changes.
- Replayable protocol vectors and captures, continuing the approach used in PR #103.
- Fault injection for sensors and hardware communication.
- Sustained hardware tests measuring recovery, accepted work, and stability.
- Reproducible efficiency benchmarks with documented measurement boundaries.

Benchmarking and test harness work are recommendations from this assessment, not claims of approved upstream projects or a complete absence of existing tests.

## 9. Installation, Updates, and Recovery

Installable S19 images are a stated project target. Moving toward operator-ready firmware creates opportunities in:

- Packaging and control-board image integration.
- Startup supervision and meaningful service health reporting.
- Upgrade and rollback behavior.
- Recovery testing after interrupted updates or configuration failures.

Some of this work may belong in companion OS or image repositories rather than the miner daemon. Agree on repository ownership before implementation.

## Prioritized Long-Term Project List

Priorities reflect recommended contributor value and sequencing, not official maintainer commitments.

| Priority | Project | First useful deliverable | Dependencies or overlap | Hardware |
|---|---|---|---|---|
| 1 | Source fault reporting and recovery | One agreed fault path visible through the API and covered by tests | Issue #113; configuration/API evolution; PR #99 | Usually unnecessary initially |
| 2 | Scheduler search-space correctness | Graceful handling of insufficient extranonce2 space | Issue #114 | Usually unnecessary initially |
| 3 | Simulated-pool failure harness | Reusable scenarios for a confirmed failure and its recovery | Existing V1 tests; SV2 harness | No |
| 4 | Share accounting and mining watchdogs | Reliable share outcomes and progress signals | Issue #113; source events; scheduler telemetry | Helpful later |
| 5 | Persistent configuration and runtime control | One validated, persistent setting with clear apply semantics | Issue #83; PR #88; configuration/API proposal | Depends on setting |
| 6 | SV2 interoperability | Reproducible interoperability or recovery regression test | PR #65 | Optional initially |
| 7 | Thermal control and fault recovery | Validated fan-control or fault-handling improvement | Issue #9; PR #33 | Required |
| 8 | Power and efficiency tuning | Repeatable measurements and bounded operating-point control | Hardware controls; telemetry; BZM2 work | Required |
| 9 | Hardware and transport expansion | Supported device or platform behavior validated on real equipment | Active board and transport PRs | Required |
| 10 | Secure deployment and image recovery | Agreed management-security or installation/recovery slice | API design; target image repositories | Depends on scope |

## Suggested Next Steps

### 1. Choose one entry point

Start with a bounded part of **issue #113**, or investigate **issue #114** as a focused correctness task. Check current ownership and overlapping work first.

### 2. Understand the relevant code path

For the recommended reliability track, read:

- `mujina-miner/src/daemon.rs`: startup, source creation, and task lifecycle.
- `mujina-miner/src/job_source/stratum_v1.rs`: upstream lifecycle and retry behavior.
- `mujina-miner/src/stratum_v1/`: protocol client and connection handling.
- `mujina-miner/src/scheduler.rs`: work assignment and share routing.
- `mujina-miner/src/api_client/types.rs`: externally visible state.
- `docs/architecture.md` and `docs/api.md`: component boundaries and API conventions.

### 3. Agree on a small, reviewable scope

Mujina asks substantial new feature proposals to begin in an Ideas discussion. Existing issues are intended to represent actionable work. Explain the failure, proposed behavior, validation approach, and interaction with active PRs.

Reference: [contribution workflow](https://github.com/256foundation/mujina/blob/main/CONTRIBUTING.md).

### 4. Reproduce before expanding

Use the CPU backend or a simulated pool to demonstrate the selected failure. Add a meaningful regression test, implement the bounded fix, and follow repository checks and atomic-commit requirements.

### 5. Build toward subsystem ownership

| Stage | Focus | Outcome |
|---|---|---|
| First month | Source/scheduler lifecycles and one agreed bug | Focused fix with a reproducible regression test. |
| Months 2–3 | Source health and share outcomes | Operators can see why mining stopped or work is rejected. |
| Months 3–6 | Failure simulation and recovery | Repeatable coverage for malformed work, disconnections, and stalled sources. |
| Longer term | End-to-end mining reliability | Watchdogs, failover, diagnostics, and consistent recovery across protocols and hardware. |

These are adaptable learning and contribution milestones, not delivery estimates.

The proposed long-term ownership area is straightforward: **ensure Mujina can tell whether it is doing useful mining work, explain why it is not, and recover appropriately.**
