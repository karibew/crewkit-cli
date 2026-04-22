# CLI State Machine Architecture

**Status**: Revised after 3 rounds of structured debate (Round 1: 5-agent architecture, Round 2: 6-agent design review, Round 3: 5-agent scientific debate with academic grounding)
**Date**: 2026-02-07 (updated from 2026-02-06)
**Decision**: Coordinator-first with phased decision gate for statig FSM — see Appendix B for rationale

---

## 1. Executive Summary

The crewkit CLI manages complex, concurrent lifecycle state spread across 7+ locations with no formal state machine. Three rounds of structured debate — 5-agent architecture selection, 6-agent design review (12 gaps found), and 5-agent scientific debate grounded in academic literature (Harel, Agha, Lamport, Sustrik, Erlang OTP) — produced the following consensus.

**The revised consensus is coordinator-first with a decision gate**: extract coordinator phases from `run_with_auth()` with typed inputs/outputs, establish actor-style ownership boundaries for concurrent services, then evaluate whether to introduce `statig` FSM based on measured complexity triggers. The hybrid statig FSM + actors remains the likely long-term architecture but is deferred until state count exceeds ~15 concurrent states or ACP (Phase 9) reaches the 90-day roadmap.

**Key findings from the final round**:
- The coordinator extraction delivers ~80% of the value at ~50% of the cost (minimax regret analysis)
- Gap 6 (/clear new) is a verified bug: only 3/8 components reset — fix immediately
- Gaps 4, 5 are phantom: the features already exist in code (shutdown_ops.rs, tui/mod.rs)
- Only ~140-220 LOC of gap work is architecture-dependent (10-15% of total)
- Zero lifecycle tests exist — the transition matrix testing discipline (from reducer debate) should be adopted regardless of architecture
- Sustrik proved coroutines ≡ state machines for hierarchical lifecycles — the coordinator and FSM are formally equivalent encodings
- Erlang OTP's `gen_statem` offers two callback modes (`state_functions` ≈ FSM, `handle_event_function` ≈ reducer) with identical formal properties — the choice is organizational, not semantic
- The AI-authoring constraint (AI models as primary code consumers) has no academic precedent and is the genuinely novel factor

**The migration strategy is three-phased with decision gates** (see Section 10 and Appendix B).

---

## 2. Architecture Decision Record

### Problem

The CLI session lifecycle is managed across 7+ locations with implicit state:

| Location | State managed | Lines |
|----------|--------------|-------|
| `code.rs` `run_with_auth()` | Session lifecycle sequencing | ~1400 |
| `App` struct | 35+ fields: ViewMode, FocusPanel, subagent data, telemetry refs | ~200 |
| `SessionTelemetryData` | Token counts, model metrics, cost, timing | ~100 |
| `EventBuffer` | Pending events, session/thread IDs, flush state | ~150 |
| `HookService` | Capture mode, convention cache, challenge tracker | ~100 |
| `SubagentTracker` | Active subagents (HashMap), completed channel | ~80 |
| `ShutdownContext` | Phase, progress, exit code | ~50 |

Cannot answer "what is the system doing right now?" without reading all 7 locations. Cannot answer "how did we get here?" at all.

### Constraints

- Must support concurrent subagents (0-N, dynamic) with independent lifecycles
- Must handle degraded mode (API unreachable) without session loss
- Session malleability: `/clear new`, `--continue`, `--fork`
- Gate checks (safety, conventions) must block/modify events without crashing
- TUI renders at 20fps (50ms budget) — state machine dispatch must not exceed 1ms
- Future requirements: ACP protocol, SDK integration, Haiku gate checks
- Incremental migration — each phase must provide standalone value
- No breaking changes to existing CLI behavior during migration

### Options Considered

| Option | Verdict | Key strength | Fatal weakness |
|--------|---------|-------------|----------------|
| Pure statig FSM | Runner-up | Compile-time state hierarchy, self-documenting | Cannot model N concurrent subagents |
| EFSM + event sourcing | Rejected | Replay, fork, temporal queries | Third event system, schema evolution tax, 3-4 week migration |
| Tokio actor model | Component adopted | Natural concurrency, failure isolation, incremental migration | No compile-time transition validation |
| Hybrid (statig + actors) | **Adopted (refined)** | Right pattern per concern, superstate dedup | All-or-nothing FSM migration (mitigated by staged approach) |
| Compile-time typestate | Rejected | Zero runtime cost | Option unwrap lie, 1% coverage, testing bypass defeats purpose |

### Decision

**Hybrid architecture with staged migration:**

1. **Phase A**: Establish actor-style exclusive ownership for EventBuffer, SubagentTracker, and SessionTelemetry (incremental, wraps existing code)
2. **Phase B**: Introduce `statig` FSM for session lifecycle coordination (replaces `run_with_auth()` monolith)
3. **Phase C**: Wire gate checks, malleability events, and degraded mode through the FSM

### Consequences

**Positive:**
- Single source of truth for session phase (FSM state)
- Concurrent services own their state exclusively (no `Arc<Mutex<T>>`)
- Self-documenting state graph via `#[state(superstate = "...")]` annotations
- Superstate mechanism eliminates code duplication for cross-cutting concerns
- Transition audit trail via `on_transition` -> `debug_logger`

**Negative:**
- Two mental models (FSM for lifecycle, actor-style for concurrency)
- `statig` is a proc-macro dependency (~2K lines, actively maintained, forkable)
- ~10-14 days total migration effort across 3 phases

---

## 3. State Diagram

```
                    ┌─────────────────────────────────────────┐
                    │           CrewkitSession FSM             │
                    │      initial = "State::cold_start()"     │
                    └───────────────┬─────────────────────────┘
                                    │
          ┌─────────────────────────┼──────────────────────────┐
          │                         │                          │
┌─────────▼────────┐    ┌──────────▼──────────┐    ┌──────────▼──────────┐
│  «superstate»    │    │   «superstate»      │    │  «superstate»       │
│    Startup       │    │     Active           │    │   ShuttingDown      │
│                  │    │                      │    │                     │
│  handles:        │    │  handles:            │    │  handles:           │
│  - NetworkError  │    │  - NetworkError      │    │  - ForceQuit -> Done│
│    -> Offline    │    │    -> Degraded       │    │                     │
│                  │    │  - SubagentStart/    │    │                     │
│                  │    │    Stop (delegate    │    │                     │
│                  │    │    to tracker)       │    │                     │
│                  │    │  - GateCheck         │    │                     │
│                  │    │    (Handled)         │    │                     │
└──┬───────┬───┬───┘    └──┬────────────┬─────┘    └──┬────────┬────┬───┘
   │       │   │           │            │             │        │    │
   ▼       ▼   ▼           ▼            ▼             ▼        ▼    ▼
┌──────┐┌─────┐┌──────┐┌──────┐   ┌────────┐  ┌─────────┐┌──────┐┌────┐
│Cold  ││Auth ││Resolv││Runnin│   │Degraded│  │WaitFor  ││Flush ││End │
│Start ││     ││ing   ││g     │   │        │  │Subagents││Events││Sess│
│      ││     ││      ││      │   │        │  │         ││+Sum  ││ion │
└──┬───┘└──┬──┘└──┬───┘└──┬───┘   └───┬────┘  └────┬────┘└──┬───┘└─┬──┘
   │       │      │       │           │             │        │      │
   │AuthOk │ProjOk│SyncOk │ChildExit │Reconnected  │AllDone │Flush │End
   └───►   └──►   └───►   └──────────┼─────────►   └───►    └──►   └──►Done
                          ◄───────────┘
```

### State Transitions

| From | Event | To | Guard | Side Effect |
|------|-------|----|-------|-------------|
| ColdStart | AuthSuccess(token) | Authenticating | — | Store token, set API client |
| ColdStart | AuthFailed | Offline | — | Log, continue without auth |
| Authenticating | ProjectResolved | Resolving | — | Store org_id, project_id |
| Resolving | SyncComplete | Syncing | — | Store resource snapshot |
| Syncing | ApiSessionCreated | Running | — | Spawn Claude process, start TUI |
| Running | ChildExited(code) | WaitingForSubagents | — | Signal subagent tracker to drain |
| Running | ClearNew | Running (self) | — | Reset telemetry, new thread_id |
| Running | NetworkError | Degraded | — | Mark API unavailable |
| Degraded | Reconnected | Running | — | Flush buffered events |
| Degraded | ChildExited(code) | WaitingForSubagents | — | Signal drain |
| WaitingForSubagents | AllSubagentsDone | FlushingEvents | subagents.is_empty() | Begin event flush |
| FlushingEvents | EventsFlushed | GeneratingSummary | — | Begin Haiku summary |
| GeneratingSummary | SummaryDone | EndingSession | — | Begin API session end |
| EndingSession | SessionEnded | Done | — | Exit process |
| Any | ForceQuit | Done | — | Skip remaining shutdown |

### Superstate Behavior

| Superstate | Events handled | Behavior |
|-----------|---------------|----------|
| Startup | NetworkError | Transition to Offline |
| Active | NetworkError | Transition to Degraded |
| Active | SubagentStart/Stop | Delegate to SubagentTracker |
| Active | GateCheck | Handled (no transition) |
| Active | ClearNew/Resume | Self-transition on Running |
| ShuttingDown | ForceQuit | Transition to Done |

---

## 4. Event Taxonomy

### Input Events (trigger FSM transitions)

```rust
pub enum SessionEvent {
    // ── Startup ──
    AuthSuccess(String),           // token
    AuthFailed(String),            // reason
    ProjectResolved {
        org_id: String,
        project_id: Option<String>,
    },
    ProjectNotFound,
    SyncComplete { resource_count: usize },
    SyncFailed(String),
    ApiSessionCreated {
        session_id: String,
        crewkit_session_id: String,
    },

    // ── Runtime ──
    ChildExited(i32),              // exit code
    NetworkError(String),
    Reconnected,
    ClearNew,                      // /clear new
    Resume(String),                // --continue session_id
    SubagentStarted { id: String, agent_type: String },
    SubagentStopped { id: String, transcript_path: String },
    AllSubagentsDone,

    // ── Gate checks (no transition) ──
    GateCheck {
        hook_type: String,         // PreToolUse, UserPromptSubmit
        decision: GateDecision,
    },

    // ── View modes (Running substates, future) ──
    ToggleHelp,
    EnterScrollback,
    ExitOverlay,
    ToggleFeedback,

    // ── Shutdown progress ──
    EventsFlushed,
    SummaryDone(Option<String>),
    SessionEnded,
    ForceQuit,
}

pub enum GateDecision {
    Allow,
    Block(String),
    Feedback(String),
}
```

### Event Sources

| Source | Events | Delivery mechanism |
|--------|--------|-------------------|
| Auth system | AuthSuccess, AuthFailed | Direct call during startup |
| API client | ProjectResolved, ApiSessionCreated, NetworkError | Direct call / error handler |
| Resource sync | SyncComplete, SyncFailed | Direct call during startup |
| Claude process | ChildExited | `tokio::process::Child::wait()` |
| Hook service (axum) | SubagentStarted/Stopped, GateCheck, ClearNew | mpsc channel from HTTP handler |
| TUI input | ToggleHelp, ForceQuit, etc. | crossterm event poll |
| SubagentTracker | AllSubagentsDone | mpsc channel from tracker |
| Shutdown ops | EventsFlushed, SummaryDone, SessionEnded | oneshot channels |
| Reconnect timer | Reconnected | tokio interval -> mpsc |

---

## 5. Extended Variables — `SessionContext`

The FSM's shared storage holds accumulated data accessible from all states:

```rust
pub struct SessionContext {
    // ── Auth ──
    pub api_client: Arc<CrewkitApiClient>,
    pub auth_manager: Arc<AuthManager>,
    pub auth_token: Option<String>,
    pub api_available: bool,

    // ── Organization / Project ──
    pub org_id: Option<String>,
    pub project_id: Option<String>,
    pub project_slug: Option<String>,

    // ── Session ──
    pub session_id: Option<String>,
    pub crewkit_session_id: Option<String>,
    pub resource_snapshot: Vec<ResourceSnapshotEntry>,
    pub resource_count: usize,

    // ── Runtime services (owned references) ──
    pub subagent_tracker: SubagentTrackerHandle,  // channel-based access
    pub event_buffer: EventBufferHandle,           // channel-based access
    pub telemetry: SharedSessionTelemetry,         // Arc<RwLock> for TUI reads
    pub hook_service: Option<HookServiceHandle>,   // channel-based access

    // ── Shutdown ──
    pub exit_code: Option<i32>,
    pub summary: Option<String>,

    // ── Channels (inbound to FSM) ──
    pub event_rx: std::sync::mpsc::Receiver<SessionEvent>,
}
```

---

## 6. Concurrency Model

```
                         ┌─────────────────────────────────┐
                         │        TUI Thread (sync)         │
                         │                                  │
                         │  loop {                          │
                         │    poll crossterm events (10ms)  │
                         │    poll event_rx (non-blocking)  │
                         │    for each event:               │
                         │      fsm.handle(&event)          │
                         │    render frame                  │
                         │  }                               │
                         └──────┬──────────────────────────┘
                                │ reads
                                ▼
                   ┌────────────────────────────┐
                   │  SharedSessionTelemetry     │
                   │  Arc<RwLock<TelemetryData>> │
                   │  (read by TUI, written by   │
                   │   telemetry service)         │
                   └────────────────────────────┘

  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │ EventBuffer  │   │ Subagent     │   │ Telemetry    │
  │ Service      │   │ Tracker      │   │ Server       │
  │              │   │              │   │              │
  │ owns: events │   │ owns: agents │   │ owns: OTEL   │
  │ channel-in   │   │ channel-in   │   │ channel-in   │
  │ flush via    │   │ spawn/drain  │   │ axum server  │
  │ tokio task   │   │ via tokio    │   │              │
  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘
         │                  │                   │
         └──────────────────┼───────────────────┘
                            │
                   mpsc channels to FSM
                   (SessionEvent variants)
```

### Key principles

1. **Exclusive ownership**: Each service owns its mutable state. No `Arc<Mutex<T>>` for SubagentTracker or EventBuffer.
2. **Channel interfaces**: Services expose `Handle` types with `mpsc::Sender<Msg>` for commands.
3. **FSM is sync**: The TUI thread owns the FSM and dispatches events synchronously. Async operations send results back via `std::sync::mpsc`.
4. **TUI reads telemetry via Arc<RwLock>**: The one remaining shared read path. Write-rare (telemetry updates), read-often (TUI frames). Negligible contention.

---

## 7. Gate Condition API

Gate checks (safety, conventions) are events that the FSM handles without transitioning. They're resolved at the `Active` superstate level:

```rust
#[superstate]
fn active(&mut self, event: &SessionEvent) -> Outcome<State> {
    match event {
        SessionEvent::GateCheck { hook_type, decision } => {
            match decision {
                GateDecision::Allow => { /* no-op */ }
                GateDecision::Block(reason) => {
                    // Hook service already sent block response to Claude Code
                    debug_logger::log("GATE", &format!("Blocked {}: {}", hook_type, reason));
                }
                GateDecision::Feedback(msg) => {
                    // Convention guidance injected via hook response
                    debug_logger::log("GATE", &format!("Feedback on {}: {}", hook_type, msg));
                }
            }
            Handled
        }
        // ... other active events
        _ => Super
    }
}
```

### Future gate conditions

| Gate | Trigger | FSM integration |
|------|---------|-----------------|
| Haiku safety check | UserPromptSubmit hook | GateCheck event with Block/Allow |
| Convention enforcement | PreToolUse hook | GateCheck event with Feedback |
| Token budget | TelemetryUpdate threshold | Guard on Running state |
| ACP authorization | Future protocol event | New event variant + state |

---

## 8. Transition Audit Trail

Use the existing `debug_logger` infrastructure — no new JSONL event log.

```rust
// In the statig FSM
fn on_transition(&mut self, source: &State, target: &State) {
    debug_logger::log(
        "FSM",
        &format!("{:?} -> {:?}", source, target),
    );
}
```

Audit entries appear in `.crewkit/debug-latest.log` alongside existing categories:

```
[2026-02-06 14:23:01.234] [FSM] ColdStart -> Authenticating
[2026-02-06 14:23:01.567] [FSM] Authenticating -> Resolving
[2026-02-06 14:23:02.890] [FSM] Resolving -> Syncing
[2026-02-06 14:23:03.123] [FSM] Syncing -> Running
...
[2026-02-06 15:45:12.456] [FSM] Running -> WaitingForSubagents
[2026-02-06 15:45:13.789] [FSM] WaitingForSubagents -> FlushingEvents
```

### Why not a separate event log

The debate's strongest consensus point: a JSONL event log creates a third event system alongside `EventBuffer` (API delivery) and OTEL telemetry. The overhead (schema evolution, projection maintenance, I/O path) is not justified when `debug_logger` already captures the same transitions in human-readable form. If deterministic replay is needed in the future, it can be added as an opt-in feature.

---

## 9. Integration Strategy

### How FSM connects to existing systems

| System | Current integration | FSM integration |
|--------|-------------------|-----------------|
| `code.rs` `run_with_auth()` | 1400-line monolith | Replaced by FSM state entry actions |
| TUI event loop (`tui/mod.rs`) | 700-line `run_app()` | Polls `event_rx` + crossterm, dispatches to FSM |
| Hook service (axum) | `handle_hook_event()` match | Sends SessionEvent via mpsc to FSM |
| EventBuffer | `Arc<Mutex<EventBuffer>>` | `EventBufferHandle` with channel interface |
| SubagentTracker | `Arc<Mutex<SubagentTracker>>` | `SubagentTrackerHandle` with channel interface |
| SessionTelemetry | `Arc<RwLock<TelemetryData>>` | Unchanged (read path for TUI) |
| ShutdownPhase | Enum in `App` | Replaced by FSM states under ShuttingDown superstate |
| ViewMode | Enum in `App` | Can be FSM substates under Running (Phase C) |
| debug_logger | Direct calls | `on_transition` callback |

### Channel architecture

```rust
// Created during startup, before FSM
let (event_tx, event_rx) = std::sync::mpsc::channel::<SessionEvent>();

// Hook service gets a clone of event_tx
let hook_event_tx = event_tx.clone();

// SubagentTracker gets a clone
let subagent_event_tx = event_tx.clone();

// TUI loop polls event_rx
loop {
    // Poll crossterm (10ms timeout)
    if crossterm::event::poll(Duration::from_millis(10))? {
        let key_event = crossterm::event::read()?;
        // Convert to SessionEvent if applicable
    }

    // Poll FSM events (non-blocking)
    while let Ok(event) = event_rx.try_recv() {
        fsm.handle(&event);
    }

    // Render frame using FSM state + telemetry
    terminal.draw(|f| {
        let state = fsm.state();
        render_ui(f, state, &telemetry);
    })?;
}
```

---

## 10. Migration Plan (Revised 2026-02-07)

### Phase 1: Gap-First + Coordinator Extraction (5-7 days)

**Goal**: Fix verified bugs, extract coordinator phases, deliver immediate value.

**Changes**:
- Fix Gap 6 (/clear new bug): expand `check_session_reset()` to reset all 8 components (thread_id, message_count, challenge_tracker, active_plan, event_buffer session_id, summary state, summary timer, start time)
- Extract `ProjectSetupService` from init.rs for code sharing with code.rs (Gap 1 prerequisite)
- Add Offline recovery path (Gap 7): Reconnected -> ColdStart transition
- Extract coordinator phases from `run_with_auth()`: `auth_phase() -> AuthResult`, `resolve_phase() -> ResolveResult`, `sync_phase() -> SyncResult`, `run_phase() -> ExitContext`, `shutdown_phase() -> ExitCode`
- Each phase is a standalone async function with typed inputs/outputs

**Standalone value**: Bug fix, code sharing for onboarding, testable phase boundaries, monolith eliminated.

**Risk**: Low. Extract-and-name refactoring with verified behavior preservation.

### Phase 2: Actor Boundaries + Testing (5-7 days)

**Goal**: Eliminate `Arc<Mutex<T>>`, establish channel-based service handles, add lifecycle tests.

**Changes**:
- Create `EventBufferHandle` (wraps `mpsc::Sender<EventBufferMsg>`)
- Create `SubagentTrackerHandle` (wraps `mpsc::Sender<SubagentMsg>`) with internal hierarchical tracking (parent_id), flat FSM-facing API (Gap 8)
- Each service runs as a `tokio::spawn` task owning its state
- Add parent_id to SubagentStarted event
- **Write transition matrix tests**: enumerate all (phase, event) pairs and verify no unhandled transitions
- Add phase boundary integration tests as living documentation
- Add signal handling taxonomy (Gap 9): SIGTERM, SIGINT, SIGHUP -> ForceQuit

**Standalone value**: Eliminates lock contention, explicit service boundaries, lifecycle test coverage from zero to comprehensive.

**Risk**: Low. Wraps existing code, doesn't change behavior.

### Decision Gate (After Phase 2)

Measure before proceeding:
- Is the coordinator causing velocity problems? (qualitative)
- How many event types in the run loop? (Trigger: >15)
- How many state-interaction bugs found since Phase 1? (Trigger: >=2)
- Is ACP (Phase 9) on the 90-day roadmap? (Trigger: yes)

**If triggers hit** → Proceed to Phase 3 (statig FSM introduction).
**If not** → Ship PLAN Phases 4B/4C features, revisit at next quarter.

### Phase 3: FSM Introduction (7-10 days, CONDITIONAL)

**Goal**: Replace coordinator run-phase with statig FSM for lifecycle transitions.

**Changes** (same as original Phase B, with review gap adjustments):
- Define `SessionContext`, `SessionEvent`, FSM states with statig derive
- Add Onboarding superstate with 7 states (Gap 1 completion)
- Add `WaitingForSubagents` state in shutdown sequence
- Replace `ShutdownPhase` enum with FSM states under ShuttingDown superstate
- Wire `event_rx` polling into TUI loop
- Add `on_transition` -> `debug_logger`
- Add "GENERATED BY" comment block above `#[state_machine]` (Gap 10)
- Add `state_count_matches_documentation` compile-time test
- Document anti-patterns: what FSM does NOT model (Gap 12)

**Standalone value**: Self-documenting state graph, superstate inheritance for cross-cutting concerns, compile-time transition enforcement.

**Risk**: Medium. Mitigated by feature flag (`--fsm` or cargo feature) and by having Phase 1+2 coordinator as fallback.

### Phase 4: Runtime Integration (2-3 days, CONDITIONAL on Phase 3)

**Goal**: Wire gate checks, malleability, and degraded mode through FSM.

**Changes**: Same as original Phase C.

### Total: Phase 1+2 = 10-14 days (unconditional) | Phase 3+4 = +9-13 days (conditional)

---

## 11. Rust Code Sketches

### Core FSM (statig)

```rust
use statig::prelude::*;

#[state_machine(
    initial = "State::cold_start()",
    state(derive(Debug, Clone, PartialEq)),
    superstate(derive(Debug)),
    on_transition = "Self::on_transition",
)]
impl SessionContext {
    // ── Startup superstate ──────────────────────────────────────
    #[superstate]
    fn startup(&self, event: &SessionEvent) -> Outcome<State> {
        match event {
            SessionEvent::NetworkError(_) => Transition(State::offline()),
            SessionEvent::ForceQuit => Transition(State::done()),
            _ => Super,
        }
    }

    #[state(superstate = "startup")]
    fn cold_start(&mut self, event: &SessionEvent) -> Outcome<State> {
        match event {
            SessionEvent::AuthSuccess(token) => {
                self.auth_token = Some(token.clone());
                Transition(State::resolving())
            }
            SessionEvent::AuthFailed(_) => Transition(State::offline()),
            _ => Super,
        }
    }

    #[state(superstate = "startup")]
    fn resolving(&mut self, event: &SessionEvent) -> Outcome<State> {
        match event {
            SessionEvent::ProjectResolved { org_id, project_id } => {
                self.org_id = Some(org_id.clone());
                self.project_id = project_id.clone();
                Transition(State::syncing())
            }
            _ => Super,
        }
    }

    #[state(superstate = "startup")]
    fn syncing(&mut self, event: &SessionEvent) -> Outcome<State> {
        match event {
            SessionEvent::SyncComplete { resource_count } => {
                self.resource_count = *resource_count;
                Transition(State::running())
            }
            _ => Super,
        }
    }

    // ── Active superstate ───────────────────────────────────────
    #[superstate]
    fn active(&mut self, event: &SessionEvent) -> Outcome<State> {
        match event {
            SessionEvent::NetworkError(_) => {
                self.api_available = false;
                Transition(State::degraded())
            }
            SessionEvent::SubagentStarted { id, agent_type } => {
                self.subagent_tracker.on_start(id.clone(), agent_type.clone());
                Handled
            }
            SessionEvent::SubagentStopped { id, transcript_path } => {
                self.subagent_tracker.on_stop(id.clone(), transcript_path.clone());
                Handled
            }
            SessionEvent::GateCheck { decision, .. } => {
                // Gate checks don't cause transitions
                Handled
            }
            SessionEvent::ForceQuit => Transition(State::done()),
            _ => Super,
        }
    }

    #[state(superstate = "active", entry_action = "enter_running")]
    fn running(&mut self, event: &SessionEvent) -> Outcome<State> {
        match event {
            SessionEvent::ChildExited(code) => {
                self.exit_code = Some(*code);
                Transition(State::waiting_for_subagents())
            }
            SessionEvent::ClearNew => {
                self.telemetry.reset_for_new_session();
                self.event_buffer.generate_thread_id();
                Transition(State::running()) // self-transition
            }
            SessionEvent::Resume(session_id) => {
                self.event_buffer.restore_thread(session_id.clone());
                Handled
            }
            _ => Super,
        }
    }

    #[state(superstate = "active")]
    fn degraded(&mut self, event: &SessionEvent) -> Outcome<State> {
        match event {
            SessionEvent::Reconnected => {
                self.api_available = true;
                self.event_buffer.flush();
                Transition(State::running())
            }
            SessionEvent::ChildExited(code) => {
                self.exit_code = Some(*code);
                Transition(State::waiting_for_subagents())
            }
            _ => Super,
        }
    }

    // ── Shutdown superstate ─────────────────────────────────────
    #[superstate]
    fn shutting_down(&self, event: &SessionEvent) -> Outcome<State> {
        match event {
            SessionEvent::ForceQuit => Transition(State::done()),
            _ => Super,
        }
    }

    #[state(superstate = "shutting_down")]
    fn waiting_for_subagents(&mut self, event: &SessionEvent) -> Outcome<State> {
        match event {
            SessionEvent::AllSubagentsDone => {
                Transition(State::flushing_events())
            }
            SessionEvent::SubagentStopped { id, transcript_path } => {
                self.subagent_tracker.on_stop(id.clone(), transcript_path.clone());
                // Check if all done
                if self.subagent_tracker.is_empty() {
                    Transition(State::flushing_events())
                } else {
                    Handled
                }
            }
            _ => Handled, // Absorb other events during drain
        }
    }

    #[state(superstate = "shutting_down")]
    fn flushing_events(&self, event: &SessionEvent) -> Outcome<State> {
        match event {
            SessionEvent::EventsFlushed => Transition(State::generating_summary()),
            _ => Super,
        }
    }

    #[state(superstate = "shutting_down")]
    fn generating_summary(&mut self, event: &SessionEvent) -> Outcome<State> {
        match event {
            SessionEvent::SummaryDone(summary) => {
                self.summary = summary.clone();
                Transition(State::ending_session())
            }
            _ => Super,
        }
    }

    #[state(superstate = "shutting_down")]
    fn ending_session(&self, event: &SessionEvent) -> Outcome<State> {
        match event {
            SessionEvent::SessionEnded => Transition(State::done()),
            _ => Super,
        }
    }

    #[state]
    fn offline(&self, event: &SessionEvent) -> Outcome<State> {
        match event {
            SessionEvent::ChildExited(_) => Transition(State::done()),
            SessionEvent::ForceQuit => Transition(State::done()),
            _ => Handled,
        }
    }

    #[state]
    fn done(&self, _event: &SessionEvent) -> Outcome<State> {
        Handled // Terminal
    }

    // ── Actions ─────────────────────────────────────────────────
    fn enter_running(&mut self) {
        debug_logger::log("FSM", "entering Running state");
    }

    fn on_transition(&mut self, source: &State, target: &State) {
        debug_logger::log("FSM", &format!("{:?} -> {:?}", source, target));
    }
}
```

### Service Handle Pattern (Phase A)

```rust
// Channel-based handle replaces Arc<Mutex<EventBuffer>>
pub struct EventBufferHandle {
    tx: mpsc::Sender<EventBufferMsg>,
}

enum EventBufferMsg {
    AddEvent(HookEvent),
    SetSessionId(String),
    GenerateThreadId,
    RestoreThread(String),
    Flush,
    Finalize(oneshot::Sender<()>),
}

impl EventBufferHandle {
    pub fn add_event(&self, event: HookEvent) {
        let _ = self.tx.try_send(EventBufferMsg::AddEvent(event));
    }

    pub fn generate_thread_id(&self) {
        let _ = self.tx.try_send(EventBufferMsg::GenerateThreadId);
    }

    pub async fn finalize(&self) {
        let (done_tx, done_rx) = oneshot::channel();
        let _ = self.tx.send(EventBufferMsg::Finalize(done_tx)).await;
        let _ = done_rx.await;
    }
}

// Service task owns EventBuffer exclusively
pub fn spawn_event_buffer(mut buffer: EventBuffer) -> EventBufferHandle {
    let (tx, mut rx) = mpsc::channel(128);

    tokio::spawn(async move {
        while let Some(msg) = rx.recv().await {
            match msg {
                EventBufferMsg::AddEvent(event) => {
                    buffer.add_event(event);
                    if buffer.should_flush() {
                        let _ = buffer.flush().await;
                    }
                }
                EventBufferMsg::SetSessionId(id) => buffer.set_session_id(id),
                EventBufferMsg::GenerateThreadId => { buffer.generate_conversation_thread_id(); }
                EventBufferMsg::RestoreThread(id) => { buffer.restore_thread_for_session(&id); }
                EventBufferMsg::Flush => { let _ = buffer.flush().await; }
                EventBufferMsg::Finalize(done) => {
                    let _ = buffer.finalize().await;
                    let _ = done.send(());
                    break;
                }
            }
        }
    });

    EventBufferHandle { tx }
}
```

### SubagentTracker Handle

```rust
pub struct SubagentTrackerHandle {
    tx: mpsc::Sender<SubagentMsg>,
    completed_rx: mpsc::UnboundedReceiver<CompletedSubagent>,
}

enum SubagentMsg {
    Start { id: String, agent_type: String },
    Stop { id: String, transcript_path: String },
    IsEmpty(oneshot::Sender<bool>),
    ActiveCount(oneshot::Sender<usize>),
    Summary(oneshot::Sender<Vec<SubagentSummary>>),
    Drain, // Begin orderly shutdown
}

impl SubagentTrackerHandle {
    pub fn on_start(&self, id: String, agent_type: String) {
        let _ = self.tx.try_send(SubagentMsg::Start { id, agent_type });
    }

    pub fn on_stop(&self, id: String, transcript_path: String) {
        let _ = self.tx.try_send(SubagentMsg::Stop { id, transcript_path });
    }

    pub async fn is_empty(&self) -> bool {
        let (tx, rx) = oneshot::channel();
        let _ = self.tx.send(SubagentMsg::IsEmpty(tx)).await;
        rx.await.unwrap_or(true)
    }
}
```

---

## 12. Debate Summary

### Participation

5 agents debated across 3 rounds: position papers, cross-examination, and rebuttals with consensus vote.

### Vote Results

| Architecture | 1st-place | 2nd-place | Total |
|-------------|-----------|-----------|-------|
| hybrid-architect | 2 | 1 | 3 |
| fsm-purist | 2 | 1 | 3 |
| actor-advocate | 1 | 1 | 2 |
| typestate-minimalist | 0 | 1 | 1 |
| event-sourcer | 0 | 1 | 1 |

### Key Arguments That Moved the Debate

1. **Subagent shutdown race** (event-sourcer -> fsm-purist): The FSM's `ChildExited` transition to `FlushingEvents` with no guard on active subagents would lose in-flight transcript data. Led to the `WaitingForSubagents` state in the consensus design.

2. **Third event system** (fsm-purist, hybrid-architect -> event-sourcer): Creating JSONL event store alongside existing EventBuffer and OTEL was rejected as unjustified overhead. Led to dropping the event log from the hybrid.

3. **Option::unwrap lie** (event-sourcer, hybrid-architect -> typestate-minimalist): The typestate's compile-time safety was undermined by `Option<T>` fields with runtime unwraps. Fatal flaw that eliminated typestate as standalone approach.

4. **Silent state desync** (typestate-minimalist -> actor-advocate): Multi-actor `/clear new` reset creates race condition between session_id, thread_id, and telemetry updates. Led to barrier coordination in the actor rebuttal.

5. **All-or-nothing FSM migration** (actor-advocate, typestate-minimalist -> hybrid-architect): The FSM cannot be introduced incrementally because states are interdependent. Led to the staged migration strategy: actors first (Phase A), then FSM (Phase B).

### Concessions

| Agent | Conceded | Impact |
|-------|----------|--------|
| fsm-purist | Subagent gap is real, migration underestimated | Voted for hybrid over own proposal |
| event-sourcer | Third event system, sync flush, schema tax | Dropped event store for startup, kept concept for runtime |
| actor-advocate | State desync on reset, channel backpressure | Added barrier coordination, try_send + overflow |
| hybrid-architect | EventLog unjustified, migration too risky, deadlock | Dropped EventLog, adopted staged migration |
| typestate-minimalist | unwrap flaw is fatal, testing bypass defeats purpose | Conceded statig covers same ground better |

### Dissenting Opinion

**actor-advocate** voted against the statig-based approaches, preferring typestate for startup + actors for runtime. The argument: zero external dependencies, no proc-macro supply-chain risk, actors are already the existing pattern. This is a valid minority position — if statig becomes unmaintained, the actor-advocate's approach is the fallback.

---

## Appendix A: Crate Evaluation

| Crate | Async | Hierarchy | Concurrent | Maintained | Verdict |
|-------|-------|-----------|------------|------------|---------|
| **statig** | Yes | Yes (superstates) | No (use actors) | Active (748+ stars) | **Selected** |
| rust-fsm | No | No | No | Active | Rejected (no async, no hierarchy) |
| finny | No | Yes | Yes (regions) | Dead (2022) | Rejected (unmaintained) |
| Custom enum | Yes | Manual | Manual | N/A | Baseline (current approach) |
| Typestate | Yes | Manual | No (static) | N/A | Rejected (ergonomic/testing flaws) |

### statig Highlights

- `#[state_machine]` derive macro on impl block
- `#[state(superstate = "...")]` for hierarchy
- `#[superstate]` handlers with `Super` delegation
- Entry/exit actions via `#[action]`
- `on_transition` callback for logging
- `Outcome::Transition(State::...)` / `Outcome::Handled` / `Outcome::Super`
- Async support via `async` feature flag
- ~2K lines, zero runtime deps, `no_std` compatible
- Forkable if maintainer abandons (low proc-macro complexity)

### Performance Baseline

| Operation | Cost | Impact on 50ms TUI frame |
|-----------|------|--------------------------|
| statig dispatch (3-level depth) | ~10-30ns | 0.00006% |
| mpsc channel send | ~50-200ns | 0.0004% |
| debug_logger write | ~1-5us | 0.01% |
| Subagent event processing (peak: 15/min) | ~450ns/min | 0% |
| Total FSM overhead per frame | <1us | <0.002% |

---

## Appendix B: Final Round Debate Synthesis (2026-02-07)

### B.1 Debate Format

5 agents, each defending a falsifiable hypothesis, debated over 3 rounds of cross-examination. Each agent performed deep codebase research (reading actual source files, counting LOC, searching git history) before debating. Grounded in academic literature review: Harel (1987), Agha (1986), Lamport TLA+, Sustrik structured concurrency, Erlang OTP gen_statem, Akka typed behaviors.

### B.2 Hypotheses and Verdicts

| # | Hypothesis | Agent | Verdict |
|---|---|---|---|
| H1 | Proc-macro opacity is a phantom cost | fsm-advocate | CONFIRMED: 434 proc-macros already in CLI, statig adds 0.2% |
| H2 | State count exceeds coordinator threshold in 6mo | growth-analyst | MODIFIED: Growth is bimodal. Threshold reached at ACP (Phase 9), not immediately |
| H3 | Coordinator-first dominates via minimax regret | coordinator-strategist | CONFIRMED: Never catastrophically wrong; FSM IS catastrophically wrong if complexity stays low |
| H4 | Testability is decisive; reducer wins | reducer-champion | CONFIRMED thesis, REJECTED prescription: transition matrix testing adopted, full reducer architecture rejected |
| H5 | 12 gaps invalidate timelines | risk-analyst | PARTIALLY PROVEN: 2 phantom gaps, only 10-15% architecture-dependent, but timelines are 20-40% under |

### B.3 Key Evidence That Moved Positions

1. **434 proc-macro usages** (fsm-advocate): Empirically disproved proc-macro opacity as a blocking concern
2. **5 different NetworkError recovery strategies** (coordinator-strategist): Proved uniform superstate handlers would be incorrect for startup phases
3. **Gap 6 verified bug** (risk-analyst): /clear new resets 3/8 components — any centralized dispatcher fixes this
4. **Zero lifecycle tests** (reducer-champion): The 3,570-line code.rs and 8,350-line TUI module have no state transition tests
5. **Bimodal growth** (growth-analyst): 80% event-type growth (coordinator-friendly), 20% execution mode growth (needs hierarchy)
6. **gen_statem dual modes** (reducer-champion): Erlang's `state_functions` ≈ FSM and `handle_event_function` ≈ reducer have identical formal properties

### B.4 Cross-Debate Concessions

| From | To | Concession |
|---|---|---|
| fsm-advocate | coordinator-strategist | Phase A coordinator extraction is simpler and faster |
| fsm-advocate | reducer-champion | Zero lifecycle tests is a real problem |
| growth-analyst | coordinator-strategist | Coordinator viable for 3-4 months; core lifecycle stable at ~11-13 states |
| growth-analyst | reducer-champion | Acknowledged reducer test matrix, but scale concern at 28-33 states |
| coordinator-strategist | fsm-advocate | FSM makes state graph more visible |
| coordinator-strategist | reducer-champion | Zero lifecycle tests exist, coordinator must actually write them |
| reducer-champion | all | Sequential startup has more ceremony than async workflows |
| reducer-champion | fsm-advocate | is_active_phase() guard less type-safe than superstates |
| risk-analyst | reducer-champion | Architecture matters for Gaps 6 and 7 |
| risk-analyst | all | Phase A has standalone value regardless |

### B.5 Academic Grounding Summary

| Principle | Source | Impact on Decision |
|---|---|---|
| Hierarchical FSMs reduce state explosion | Harel (1987) | Superstates justified — but only at scale (>15 states) |
| Coroutines ≡ state machines for hierarchical lifecycles | Sustrik (2018) | Coordinator and FSM are formally equivalent encodings |
| FSM-within-actor is the industrial standard | Erlang OTP | The hybrid IS the established pattern; coordinator is a valid stepping stone |
| Two callback modes have identical formal properties | Erlang gen_statem | FSM vs reducer is organizational, not semantic |
| Verification is architecture-independent | Lamport TLA+ | Transition matrix tests work with any implementation |
| Actors lack transition structure | Agha (1986) | Every mature actor system adds FSM-like patterns eventually |

### B.6 The Novel Factor: AI-Authoring

No academic treatment addresses AI models as primary code authors/consumers. The debate produced assertions but no measurements:
- fsm-advocate: AI models handle 434 proc macros in this codebase
- coordinator-strategist: AI models understand tokio::select! from vast training data
- reducer-champion: Pure match functions are maximally transparent

This remains the unresolved question. The coordinator-first strategy hedges against it: if AI-authoring proves to prefer explicit patterns, the coordinator's flat structure serves. If AI models handle proc macros fine (as the 434-count suggests), the FSM migration proceeds without penalty.

### B.7 Dissenting Opinions

**Reducer (1/5)**: reducer-champion argues the `handle_event_function` pattern (Erlang-validated) with effects-as-data provides the strongest testing properties. Rejected as full architecture due to no Rust ecosystem precedent, sequential ceremony, and scaling concerns at 28-33 states. The testing DISCIPLINE was universally adopted.

**Immediate FSM (2/5)**: fsm-advocate and growth-analyst argue the FSM should proceed without a decision gate. Superstates are irreducible, state count is already 18-20 with gaps, and the proc-macro cost is empirically negligible. Valid position — the decision gate trades structural enforcement time for reduced execution risk.

### B.8 Scratchpad Files (Full Debate Records)

- `scratchpad/arch-actor-model.md` — Actor model position paper + 4 cross-debates
- `scratchpad/arch-async-workflow.md` — Async workflow position + convergence with actor
- `scratchpad/arch-fsm-enhanced.md` — Enhanced FSM position + post-debate concessions
- `scratchpad/arch-reducer-elm.md` — Reducer/Elm position + rebuttals
- `scratchpad/review-synthesis.md` — 6-agent design review (12 gaps, conditional approve)
- `scratchpad/final-round-synthesis.md` — This appendix in expanded form
