# Implementation Plan: Clinic Appointment Chat

**Branch**: `001-clinic-appointment-chat` | **Date**: 2026-10-03 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-clinic-appointment-chat/spec.md`

## Summary

Build a small Python Streamlit clinic chat using a Google ADK `LlmAgent` backed by LiteLLM with OpenAI `gpt-4o`, five patient-facing tools, and an ADK session scratchpad. Persist conversation sessions in `runtime/sessions.db` and appointments in a one-time runtime copy of the clinic seed database. Keep identity checks, clinical refusal/emergency routing, and booking uniqueness in deterministic application/database logic outside the model. Provide a separate read-only staff search page with a white and aqua theme.

## Technical Context

**Language/Version**: Python 3.11+

**Primary Dependencies**: Google ADK (database extra), LiteLLM, Streamlit, aiosqlite, pytest

**Storage**: ADK sessions and events in `runtime/sessions.db`; mutable clinic records in `runtime/clinic.db`, initialized once from `data/clinic.db`.

**Testing**: pytest for tool, authorization, database, callback, and UI-flow checks; include a two-connection simultaneous booking check against the unique index and same-doctor transaction rule.

**Target Platform**: Local single-server Python application for a Peshawar clinic demonstration.

**Project Type**: Single-project Streamlit application with an ADK agent and local SQLite files.

**Performance Goals**: Search and booking should feel immediate for the provided clinic dataset. Simultaneous requests for one doctor/date/time must result in no more than one active appointment.

**Constraints**: Two-hour build scope; minimal, readable modules; one-line comments only for non-obvious behavior; no diagnosis or medicine advice; emergency requests direct to 1122; session-derived patient identity; no deployment, appointment holds, rescheduling, or conversation summarization beyond the requested bounded-history summary.

**Scale/Scope**: One server and the supplied fictional clinic dataset; correctness under simultaneous users is required, but no higher throughput target is specified.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

- **Simple, readable code**: PASS. Keep a small root-level app, agent, tool, and database module; no general framework layer.
- **Session-bound patient identity**: PASS. Patient picker establishes the selected MR in the server-controlled ADK session; tool arguments never contain patient identity. Every patient read/write scopes by that session identity.
- **Atomic appointment booking**: PASS. Existing partial unique index on active `(doctor_id, date, time)` is the slot authority; the same-doctor rule is checked and inserted inside one `BEGIN IMMEDIATE` transaction. Availability checks are advisory only.
- **Safe clinical boundaries**: PASS. Emergency and prohibited-advice handling is deterministic and specified in the agent instructions and application guardrails.
- **Explicit guardrails**: PASS. Validate model tool inputs/outputs; do not allow model output to choose a patient or directly write records. Staff search has a separate read-only access path.
- **Clinical safety constraints**: PASS. Search and chat do not expose another patient's private conversations or appointment data.
- **Development workflow**: PASS. The plan includes automated checks for authorization, concurrent booking, and safety behavior before release.

## Project Structure

### Documentation (this feature)

```text
specs/001-clinic-appointment-chat/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── agent-tools.md
│   └── ui.md
└── tasks.md
```

### Source Code (repository root)

```text
app.py                    # Streamlit patient chat and staff search pages
agent.py                  # ADK agent, runner, persistent sessions, callback
tools.py                  # Five patient-facing tools and safety checks
clinic_db.py              # Read-only queries and appointment writes
data/
└── clinic.db              # Canonical seed data copied once to runtime/
runtime/                   # Local mutable databases; ignored by version control
.streamlit/
└── config.toml             # White and aqua light theme
tests/
├── test_tools.py
├── test_booking_race.py
└── test_safety_and_identity.py
requirements.txt
```

**Structure Decision**: Use a single small Python project at repository root. Separate UI, agent wiring, tool policy, and clinic database access into four readable modules. Do not add a separate API service or a frontend build. The repository currently has `clinic.db` at root; implementation must establish `data/clinic.db` as the canonical seed path without changing its schema before the runtime copy is created.

## Phase 0: Research

Research decisions and primary-source links are recorded in [research.md](research.md). Key decisions:

- Use the LiteLLM adapter with model ID `openai/gpt-4o` and exactly five patient-facing function tools.
- Use `before_model_callback` to bound model context to the latest 20 content items plus a compact summary; retain complete persisted history for reopen.
- Use ADK app/user/session/invocation state scopes for the requested memory panel. `temp:free_slots` is short-lived and never authorizes a later booking.
- Keep the existing SQLite partial unique index authoritative for simultaneous slot claims; enforce the per-patient/same-doctor rule within a write transaction because the seed already contains multiple active records for some patient/doctor pairs. Distinguish slot conflicts from SQLite busy/storage errors.
- Use persisted ADK sessions and a separate persisted mutable clinic database. Initialize the mutable clinic copy only once.
- Keep the staff search page read-only and behind a separate staff access gate.

## Phase 1: Design and Contracts

- [Data model](data-model.md) defines the clinic entities, ADK conversation/state lifetimes, ownership rules, and appointment transitions.
- [Agent tool contracts](contracts/agent-tools.md) define the five functions, trusted identity source, validation, side effects, and conflict responses.
- [UI contract](contracts/ui.md) defines patient chat/session/memory interactions and the staff-only read-only search page.
- [Quickstart](quickstart.md) gives local setup and acceptance walkthroughs for persistence, identity isolation, safety, and simultaneous booking.

## Constitution Check (Post-Design)

- **Simplicity**: PASS. The design uses one process, four application modules, one seed database, and two runtime SQLite databases.
- **Identity and authorization**: PASS. MR comes from the selected patient's server-side session; tool contracts omit MR arguments. Staff is a separate role-gated page.
- **Booking integrity**: PASS. The existing unique partial index prevents duplicate active slots; the same-doctor check and insert share a write transaction. Existing seed records are preserved.
- **Clinical safety**: PASS. Deterministic guardrails run outside the model. Emergency requests route to 1122 and do not continue into booking.
- **Persistence and privacy**: PASS. Full transcripts and app/user/session state are stored in ADK's session database; patient queries are scoped and the staff page is read-only.
- **Review checks**: PASS. Planned checks cover session identity, two simultaneous inserts, cancellation releasing a slot, memory scope, and emergency/medical advice refusals.

## Complexity Tracking

No constitution violations or additional architectural layers are introduced.
