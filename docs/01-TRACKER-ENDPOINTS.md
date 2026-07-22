# Endpoint Trackers — Capture Layer

**Intelligent Employee Productivity Tracking System (PTA)**

The capture layer is the source of every signal the system reasons about. It is a
set of **five independent Rust services** that run on each monitored Windows
machine, plus a single supervisor that starts and restarts them. Each service
owns one input domain, owns its own database tables, and can be built, deployed
and reasoned about on its own — a crash in one never stops the others.

```
pta_launcher  (supervisor: spawn, stream logs, restart on failure)
   ├─ process monitor      → process / ETW signals
   ├─ keyboard service     → keystroke dynamics
   ├─ mouse agent+backend  → movement / clicks / scroll / drag
   ├─ focus service        → active-window focus
   └─ activities syncer    → derived, enriched activity records
```

---

## 1. Design principles

- **One service per domain.** Independent agents that each own their tables.
  Simpler to build and deploy, and fault-isolated.
- **Capture dynamics, not content.** The keyboard service records keystroke
  *timing, counts, modifier and shortcut categories* — it is **not** a content
  keylogger. Behavioural features never need the actual characters typed.
- **Write straight to the source of truth.** Every service writes raw events to
  the central PostgreSQL database through a connection pooler. Nothing is
  aggregated or interpreted on the endpoint; that happens server-side and stays
  reproducible.
- **Auto-start.** Adding `pta_launcher` to OS auto-start brings the whole capture
  stack up automatically at logon.

---

## 2. The launcher — `pta_launcher`

A single supervisor process spawns each of the five services, streams their
console output into a combined log, and restarts any service that exits. It is
the only thing that has to be registered with Windows auto-start; everything else
comes up underneath it.

Responsibilities:

- Spawn each capture service as a child process.
- Capture and interleave their stdout/stderr for a single operator view.
- Detect a child exit and relaunch it (the capture services are long-lived).
- Bring the whole stack down cleanly on shutdown.

---

## 3. The five services

| Service | Captures | Capture mechanism | Primary tables |
|---|---|---|---|
| **Process monitor** | Running processes, lifecycle, resource use, security, dependencies | Windows **ETW** (Event Tracing for Windows) via `ferrisetw`, plus native APIs (`NtQuerySystemInformation`, `GetProcessIoCounters`, `WinVerifyTrust`); ~100 ms scan | `process_events` (+ `process_lifecycle`, `process_resource_history`, `process_performance_metrics`, `process_session_integration`, `process_security_permissions`, `process_application_classification`, `process_dependencies`) |
| **Keyboard service** | Keystroke timing, modifiers, shortcuts | Low-level keyboard hook (`SetWindowsHookEx` / `WH_KEYBOARD_LL`) | `key_events`, `key_modifier_events`, `key_shortcut_events`, `key_special_events` |
| **Mouse service** | Movement, clicks, scroll, drag, gestures | A capture agent (`rdev`) posts events over local HTTP to a small backend (Axum, `127.0.0.1`) that writes to the DB | `mouse_events` (+ `mouse_movement_events`, `mouse_click_events`, `mouse_scroll_events`, `mouse_drag_events`, `mouse_gesture_events`) |
| **Focus service** | Active window / application focus | Foreground-window polling via the active-window API (`active_win`) | `focus_events` |
| **Activities syncer** | Derived, enriched activity records | Does **not** capture hardware — see §4 | `activities` |

### 3.1 Process monitor

The heaviest and most detailed service. It listens to Windows ETW to see process
starts/stops as they happen, and enriches each with native calls for resources
(CPU, memory, I/O counters), security (signature verification via
`WinVerifyTrust`), session integration and dependency relationships. A ~100 ms
scan interval keeps process state close to real-time without a busy loop.

### 3.2 Keyboard service

A low-level keyboard hook produces one record per keystroke, but only the
*behavioural* attributes are retained: press/release timing, inter-key intervals,
modifier usage (Ctrl/Alt/Shift), and recognised shortcut/special-key categories.
These are exactly the signals the cheat-detection model needs to tell human
rhythm from machine rhythm.

### 3.3 Mouse service (agent + backend)

Split in two: a lightweight capture **agent** (`rdev`) that observes raw mouse
input, and a local **backend** (Axum HTTP on `127.0.0.1`) that receives posted
events and writes them to the database. Movement, clicks, scroll, drag and
gesture events are stored in separate tables so each can be aggregated at its own
resolution.

### 3.4 Focus service

Polls the foreground window and records focus transitions — which application and
window title held focus, and for how long. This is the backbone of the
application-context and app-classification features (which app was in focus while
the keyboard/mouse activity happened).

---

## 4. Activities are *derived*, not captured

The **activities service is different from the other four**: it reads no
hardware. It runs a **syncer** that reads `process_events`, enriches each event
by joining the latest related process / session / focus rows, classifies it
(e.g. `web_browsing`, `development`, `communication`, `system`), and writes one
`activities` row per source event.

The syncer is **device-scoped**: each machine's syncer only processes its *own*
machine's `process_events` (matched on host name). This guarantees:

- two machines never process the same events (no duplicate activities), and
- one machine going offline never stalls activity derivation for the others.

```
process_events (this host only) → syncer (enrich + classify) → activities
```

---

## 5. Where the raw events go

Every service writes into the **raw layer** of the central database — 20 event
tables, one family per capture domain. The raw layer is treated as append-only
and is the source of truth: all features and predictions downstream are derived
from it and can be rebuilt at any time.

All endpoint traffic passes through **PgBouncer** (transaction pooling). Each
machine runs five write-heavy services, so without pooling `machines × services`
would exhaust database connections; the pooler multiplexes many client
connections onto a small number of real ones. Services use **named prepared
statements** sized to the pooler so statement caching and transaction pooling
coexist correctly.

See [Backend API & Database](02-BACKEND-API-AND-DATABASE.md) for the storage,
identity and aggregation layers that consume these raw events.

---

## 6. Known limitations (honest)

- **Service crashes.** In the collection period the process and focus services
  crashed intermittently (roughly 17–44% of active hours on some machines),
  which is the largest single source of missing data. Hardening these is the
  first item of collection-side future work.
- **Windows only.** The capture mechanisms (ETW, low-level hooks, active-window
  API) are Windows-specific by design.
- **Upstream productivity heuristic is coarse.** The capture layer's own
  `productive`/`unproductive` flag marks any browser session productive
  regardless of content; the ML layer therefore does **not** trust that flag and
  recomputes productivity from app-classified behaviour instead.
