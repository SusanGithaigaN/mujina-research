# Mujina Firmware: Contribution Gaps & Roadmap Analysis


## Executive Summary

Mujina is a Rust-based open-source Bitcoin mining firmware with solid architectural foundations:
a clean hardware abstraction layer, a working BM13xx ASIC driver, a functioning Stratum V1
client, and a REST API. However, several subsystems are either explicitly stubbed out
(`unimplemented!()`), listed as roadmap items in the docs, or entirely absent by comparison
to what production mining firmware requires.

**The three highest-impact contribution areas for a long-term contributor are:**

1. **Stratum V2** — the largest protocol gap in all open-source mining firmware. Encryption,
   binary framing, and miner-side block template construction (Job Declaration Protocol) are
   all missing. This directly enables decentralization and censorship resistance.

2. **Antminer S19 board support** — the most widely deployed mining hardware in the world
   is explicitly on the project roadmap but unimplemented. A working board driver here would
   expand mujina's real-world reach dramatically.

3. **Persistent configuration + multi-pool failover** — `Config::load()` is `unimplemented!()`.
   The project currently only runs via environment variables, cannot failover to a backup pool,
   and has no runtime reconfiguration. These are blockers for any production deployment.

---

## 1. Protocol Layer

### 1.1 Stratum V2 (Not Started)

**Status:** No implementation exists. Only Stratum V1 is supported.

**What Stratum V2 provides over V1:**
- Noise Protocol encryption (V1 sends credentials and work in plaintext)
- Binary framing instead of JSON (significantly lower bandwidth)
- **Job Declaration Protocol (JDP):** miners construct their own block templates, choosing
  their own transactions and coinbase — the key decentralization feature
- **Template Distribution Protocol (TDP):** direct connection from miner to `bitcoind`

**Relevant implementation reference:**
- Official spec: <https://stratumprotocol.org>
- Braiins Rust library: `stratum-mining/stratum` on GitHub (Apache-2.0 licensed, potentially
  reusable as a dependency)

**Suggested scope for a contributor:**
1. Start with the **Mining Device Protocol** (the miner-to-proxy layer, replacing V1's
   `mining.subscribe` / `mining.notify`)
2. Add a `job_source/stratum_v2.rs` implementing the existing `JobSource` interface
3. Build toward the Job Declaration Protocol once the transport works

---

### 1.2 Solo Mining via `getblocktemplate` (Planned, Not Built)

**Status:** Explicitly listed in `job_source/mod.rs` module doc as a planned source type.
No `job_source/solo.rs` file exists.

**What it requires:**
- Bitcoin Core JSON-RPC client (`getblocktemplate` call)
- Coinbase transaction construction (extra nonce injection, output script)
- Long-polling support for new block notifications
- Registration as a `JobSource` in the scheduler

**Suggested scope:** Add a `job_source/solo.rs` module that implements the `SourceEvent` /
`SourceCommand` channel protocol, then wire it into the daemon's startup path behind an
environment variable / config flag.

---

### 1.3 Multi-Pool Failover (Modeled, Not Implemented)

**Status:** `config.rs` defines `Vec<PoolConfig>` with a `priority: u32` field per pool.
The scheduler only ever connects to a single source. Priorities are ignored.

**What's needed:**
- Priority-ordered source startup
- Health tracking per source (last job timestamp, accepted/rejected share ratio)
- Automatic switch to the next-priority source on disconnect or liveness timeout
- Recovery and switch-back when the primary source comes back

The scheduler already has `SourceId`-keyed `StreamMap` entries — the plumbing for multiple
concurrent sources is mostly there.

---

## 2. Configuration & Deployment

### 2.1 Persistent Configuration (Stubbed Out)

**Status:** `config.rs` has a complete struct model (`Config`, `PoolConfig`, `HardwareConfig`,
`ApiConfig`) but both `Config::load()` and `Config::load_from()` return `unimplemented!()`.
All runtime config today is environment variables only.

**What's needed:**
- TOML deserialization via `toml` or `config-rs`
- XDG-compliant file search: `/etc/mujina/mujina.toml` → `~/.config/mujina/mujina.toml`
- Environment variable override layer (env vars win over file config)
- File watching for hot-reload (the struct already has a `hot-reload via file watching` note
  in its module doc)
- Wire the loaded config into daemon startup

**Why this is a blocker:** Without it, users cannot change pool URLs without restarting the
daemon, and there is no way to persist fan limits, temperature thresholds, or API settings.

---

### 2.2 Systemd Integration (Partially Modeled)

**Status:** `DaemonConfig` has a `systemd: bool` field. Not wired up.

**What's needed:**
- `sd_notify(READY=1)` on successful startup (via `sd-notify` crate)
- Watchdog keepalive (`WATCHDOG=1`) on the main loop tick
- Proper `SIGTERM` handling tied to the existing `CancellationToken`

---

### 2.3 API Authentication + TLS (Roadmapped, Not Started)

**Status:** The API docs explicitly state: *"There's no authentication, so binding to a
public address would be unsafe."* The `ApiConfig` struct already has `tls: bool`,
`cert_path`, and `key_path` fields.

**What's needed:**
- Bearer token or HTTP Basic auth middleware on the Axum router
- TLS via `rustls` or `axum-server` with `rustls`
- A way to provision / rotate the credential (API key in config file)

---

## 3. Telemetry & Monitoring

### 3.1 WebSocket Streaming Telemetry (Explicitly "On Deck")

**Status:** The architecture doc says: *"A WebSocket channel for streaming telemetry is on
deck, so UIs can reflect live changes without polling."* Not implemented.

**What's needed:**
- `GET /api/v0/stream` WebSocket endpoint on the Axum server
- Fan out from the existing `watch::Receiver<MinerTelemetry>` channel to connected WebSocket
  clients
- Define a simple event envelope (`{ "type": "telemetry", "data": {...} }`)

**Why it matters:** The TUI (see §5.1) and any web dashboard need this to avoid 1-second
polling loops.

---

### 3.2 Prometheus / OpenMetrics Metrics Endpoint (Not Started)

**Status:** No `/metrics` endpoint exists. The telemetry data is already structured in
`MinerTelemetry`, `BoardTelemetry`, `ThreadTelemetry`, etc.

**What's needed:**
- Add `GET /metrics` endpoint (or a separate bind port)
- Use the `prometheus` or `metrics` + `metrics-exporter-prometheus` crates
- Expose: per-thread hashrate, temperature per sensor, fan RPM, shares accepted/rejected,
  pool connection status, uptime

**Why it matters:** Standard for any mining farm using Grafana. A single endpoint makes
mujina observable without custom tooling.

---

### 3.3 Historical Data / Ring Buffer Storage (Not Started)

**Status:** All telemetry is point-in-time. There is no trend data (hashrate over the past
hour, temperature history, share rate over time).

**What's needed:**
- A lightweight in-process RRD-style ring buffer (e.g., fixed-size `VecDeque` per metric
  with a configurable retention window)
- API endpoints to return time-series data: `GET /api/v0/history/hashrate?window=1h`
- Optionally: InfluxDB line protocol export for external storage

---

## 4. Thermal & Power Management

### 4.1 Closed-Loop Fan Control (PID Controller) (Not Started)

**Status:** The EMC2101 driver supports reading temperatures and setting fan duty cycle
manually. There is no feedback loop. `HardwareConfig` already models `temp_limit`,
`fan_min_rpm`, `fan_max_rpm`.

**What's needed:**
- A PID controller struct (or use the `pid` crate) with temperature as the process variable
  and fan duty cycle as the control output
- Configurable setpoint (target chip temperature), Kp/Ki/Kd gains
- Integration with the board peripheral loop (runs on the existing board task cadence)
- Hysteresis to prevent oscillation at steady state

---

### 4.2 Thermal Throttling (Not Started)

**Status:** No automatic frequency or voltage reduction when temperature approaches limits.

**What's needed:**
- A per-board thermal alarm state (uses the existing `DebouncedAlarm` type in `types/`)
- When temperature exceeds a configurable threshold: reduce ASIC frequency via the BM13xx
  frequency registers
- When temperature drops below a recovery threshold: restore frequency
- API telemetry field: `thermal_throttled: bool` per board

---

### 4.3 Dynamic Voltage/Frequency Scaling & J/TH Optimization (Not Started)

**Status:** Chips are initialized at a fixed frequency. There is no mechanism to sweep
frequency against power consumption and find the efficiency sweet spot.

**What's needed:**
- API command: `PATCH /api/v0/boards/{name}` with `target_frequency_mhz`
- Automated sweep mode: step frequency up/down while measuring hashrate and power draw,
  record efficiency (J/TH) at each point, select the optimal operating point
- Power cap enforcement (stop increasing frequency once `power_limit_w` is reached)

---

## 5. User Interface

### 5.1 Terminal UI (TUI) (Explicitly Placeholder)

**Status:** `mujina-miner/src/bin/tui.rs` is a `println!("coming soon!")` with a TODO list.
The binary exists in `Cargo.toml` but does nothing.

**The TODO in the file lists exactly what's needed:**
- Dashboard with hashrate graphs
- Temperature and power monitoring
- Pool status and shares
- Board overview
- Keyboard navigation

**Suggested implementation path:**
- Use `ratatui` (already popular in the Rust ecosystem)
- Poll or subscribe to the REST API (or WebSocket once §3.1 lands)
- Layout: top bar (uptime, aggregate hashrate), left panel (boards/threads), right panel
  (pool/source), bottom (temperature, fan, power), sparkline for 5-minute hashrate trend

---

### 5.2 CLI Enhancements (Partially Implemented)

**Status:** `mujina-cli` supports `status` and raw `api <endpoint>`. Fan control, pause/resume,
pool switching, and board-level commands exist in the API but have no CLI verbs.

**What's needed:**
- `mujina-cli fan set <name> <percent>` / `mujina-cli fan auto <name>`
- `mujina-cli pause` / `mujina-cli resume`
- `mujina-cli pool add <url>` / `mujina-cli pool list` / `mujina-cli pool switch <name>`
- `mujina-cli board <name> freq <mhz>` / `mujina-cli board <name> voltage <mv>`

---

## 6. Hardware Abstraction & Board Support

### 6.1 Antminer S19 Board Driver (Explicitly Roadmapped)

**Status:** Listed in `README.md` under "Near-term targets." Not started.

**What's needed:**
- Reverse engineering or documentation of the S19 control board serial protocol
  (the S19 uses an ESP32-based control board communicating over UART with the hashboards)
- A `board/antminer_s19.rs` implementing the `Board` trait
- Transport: likely USB serial or direct UART
- Peripherals: temperature sensors, fan PWM control

**Why it matters:** The S19 series is the most widely deployed mining hardware. A working
driver would make mujina practical for thousands of operators overnight.

---

### 6.2 Additional BM13xx Chip Variants

**Status:** BM1370 (Bitaxe Gamma) and BM1362 (EmberOne00) are implemented. Other members
of the family used in commercial hardware are not.

| Chip   | Used in          | Status      |
|--------|------------------|-------------|
| BM1370 | Bitaxe Gamma     | Implemented |
| BM1362 | EmberOne00       | Implemented |
| BM1368 | Antminer S21     | Not started |
| BM1397 | Antminer S17/T17 | Not started |
| BM1387 | Antminer S9      | Not started |

The BM13xx driver is parameterized — adding a new chip variant is primarily
a matter of documenting its register map differences and initialization sequence.

---

### 6.3 Libreboard Control Board (Roadmapped)

**Status:** Listed in `README.md`. 256 Foundation's forthcoming open-source control board.
Likely to land as a hardware target once the board is finalized.

---

## 7. Testing & Benchmarking

### 7.1 Integration Test Coverage for Stratum V1 Edge Cases

**Status:** `stratum_v1/STRATUM_QUIRKS.md` documents known protocol quirks (word-swap
encoding, extranonce edge cases). Test coverage exists for the happy path against captured
real-pool data, but adversarial edge cases (malformed JSON, missing fields, pool sending
duplicate job IDs) are not systematically tested.

**What's needed:**
- A mock pool server (`MockConnector` exists) that can inject protocol errors
- Property-based tests (via `proptest`) for the message parsing layer
- Fuzz targets for `messages.rs` JSON deserialization

---

### 7.2 Hardware-in-the-Loop (HIL) Test Framework

**Status:** Integration tests for the Bitaxe Gamma exist in `tests/bitaxe_gamma.rs` but
require physical hardware and are gated behind a feature flag / environment variable.

**What's needed:**
- A documented process for running HIL tests (currently not in `CONTRIBUTING.md`)
- A CI step that runs against real hardware (even if only on the maintainer's infra)
- A mock ASIC backend for testing the BM13xx driver without hardware

---

### 7.3 Hashrate Benchmarking Tool

**Status:** No tool exists to measure actual vs. expected hashrate, find the optimal
frequency for a given chip, or characterize chip-to-chip variance across a chain.

**What's needed:**
- A `mujina-bench` binary (or a `mujina-cli bench` subcommand) that runs a chip through
  a frequency sweep, records shares per second at each point, and emits a report

---

## Prioritized Contribution List

Ordered by impact / feasibility for a contributor who already knows the ASIC layer:

| # | Area | Difficulty | Impact | Files to Start From |
|---|------|-----------|--------|---------------------|
| 1 | Persistent config (TOML + env override) | Low | High | `src/config.rs` |
| 2 | PID fan control + thermal throttling | Low-Med | High | `src/peripheral/emc2101.rs`, `src/types/debounced_alarm.rs` |
| 3 | TUI dashboard (ratatui) | Medium | High | `src/bin/tui.rs` |
| 4 | WebSocket telemetry streaming | Low-Med | Medium | `src/api/v0.rs`, `src/api/server.rs` |
| 5 | Prometheus metrics endpoint | Low | Medium | `src/api/v0.rs` |
| 6 | Multi-pool failover | Medium | High | `src/scheduler.rs`, `src/config.rs` |
| 7 | CLI enhancements | Low | Medium | `src/bin/cli.rs` |
| 8 | API authentication + TLS | Medium | High | `src/api/server.rs` |
| 9 | Solo mining (getblocktemplate) | Medium-High | High | `src/job_source/` (new file) |
| 10 | Systemd integration | Low | Medium | `src/daemon.rs` |
| 11 | BM1368 chip variant (S21) | Medium | High | `src/asic/bm13xx/` |
| 12 | Antminer S19 board driver | High | Very High | `src/board/` (new file) |
| 13 | Stratum V2 client | Very High | Very High | `src/job_source/` (new module) |

---

## Suggested Next Steps

### Week 1-2: Orient and pick a track

1. Run the full test suite (`just checks`) and the CPU miner end-to-end to verify your
   local environment.
2. Read `docs/architecture.md` and `CODING_GUIDELINES.md` in full.
3. Pick **one** of the top-5 items above based on interest. The persistent config and
   fan control cluster are the most self-contained starting points.

### Short-term (1-3 months): Ship something deployable

- **Persistent config** is the foundation everything else builds on. Wire up `Config::load()`
  with TOML + env var override. Open a PR for this alone before touching anything else.
- **Thermal management (PID fan + throttling)** is immediately useful to anyone running
  real hardware. It's well-scoped and the peripheral driver already exists.

### Medium-term (3-9 months): Protocol and observability

- Once config lands, **multi-pool failover** is straightforward — the scheduler already
  handles multiple `SourceId` entries.
- Add **Prometheus metrics** and **WebSocket telemetry** together — they share the same
  telemetry data path.
- The **TUI** becomes practical once WebSocket streaming exists.

### Long-term (9+ months): Protocol frontier

- **Solo mining** requires understanding `getblocktemplate` and coinbase construction —
  start by reading BIP 22/23 and testing against a regtest `bitcoind`.
- **Stratum V2** is the biggest contribution possible. Study the spec at
  `stratumprotocol.org`, evaluate the `stratum-mining/stratum` Rust library as a
  dependency, and plan the `job_source/stratum_v2.rs` interface before writing code.
- **Antminer S19** requires hardware access and protocol documentation. Connect with the
  256 Foundation community on their forum and Telegram to coordinate.
