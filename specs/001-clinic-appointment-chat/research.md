# Research: Clinic Appointment Chat

## Decisions

### Agent and model

- **Decision**: Use a Python ADK `LlmAgent` with the LiteLLM model adapter configured as `LiteLlm(model="openai/gpt-4o")`. Register five patient-facing function tools: availability search, booking, appointment listing, cancellation, and patient facts/last booking.
- **Rationale**: This matches the requested model and tool count. Tool functions are the narrow boundary between model requests and application data; they derive patient identity from the authenticated ADK session context and do not accept an MR number from model arguments. Staff search remains a direct read-only UI query, outside the patient agent's tool list.
- **Alternatives considered**: Let the model query SQLite directly (rejected because it would bypass authorization and query constraints); expose separate tools for every search field (rejected to keep the requested five-tool limit).
- **Source**: [ADK LiteLLM models](https://adk.dev/agents/models/litellm/), [ADK function tools](https://adk.dev/tools-custom/function-tools/).

### Conversation history and summarization

- **Decision**: Use `before_model_callback` to prepare a bounded model request containing a concise stored summary of older conversation context plus at most the latest 20 content items. When history grows beyond the window, the orchestration layer refreshes the summary before running the agent; the callback itself does not recursively call the model. Persist the full conversation events and summary with the ADK session service.
- **Rationale**: The callback can inspect and modify the pending model request. Keeping the summary separate from the user-visible history preserves the full saved transcript while limiting prompt size.
- **Alternatives considered**: Delete old persisted messages (rejected because the spec requires reopening complete conversation history); summarize from inside the callback with another model request (rejected because it makes callback behavior recursive and harder to bound).
- **Source**: [ADK callback types](https://adk.dev/callbacks/types-of-callbacks/).

### Scratchpad and state lifetimes

- **Decision**: Use ADK session state as the structured scratchpad, with `app:clinic_today`, `user:mr_no`, `user:patient_name`, `user:last_appointment`, `last_viewed_doctor`, and `temp:free_slots`. The scratchpad contains bounded operational facts only, never hidden reasoning. `temp:free_slots` is advisory and turn-local; booking rechecks the exact doctor/date/time against the clinic database before inserting.
- **Rationale**: ADK supports app-, user-, session-, and invocation-scoped state. `temp:` values are discarded after an invocation and cannot safely carry a slot choice across separate chat turns. The database remains authoritative for availability and bookings.
- **Alternatives considered**: Trust a prior `temp:free_slots` entry on the next user turn (rejected because it is gone after the invocation and stale slots can be claimed); persist slots as authoritative state (rejected because concurrent requests can make them stale).
- **Source**: [ADK session state](https://adk.dev/sessions/state/).

### Persistence and session identity

- **Decision**: Use ADK `Runner` with `DatabaseSessionService(db_url="sqlite+aiosqlite:///runtime/sessions.db")`. Set `user_id` to the MR number selected by the patient picker, and use a separate session ID for each conversation. Initialize/update `app:clinic_today` on startup and synchronize the runtime clinic database's `meta.today` value used by its views. Keep the mutable clinic database in `runtime/clinic.db` and create that runtime copy only when it does not yet exist.
- **Rationale**: The database session service persists session events and state across process restarts. Patient selection is the application's simulated login step, so MR is assigned by the UI/session boundary rather than supplied by the model. Reusing the runtime clinic file preserves bookings across restarts.
- **Alternatives considered**: In-memory sessions (rejected because they do not survive restart); recopy the seed clinic file on each launch (rejected because it would erase bookings and cancellations).
- **Source**: [ADK sessions and session services](https://adk.dev/sessions/session/), [ADK session state](https://adk.dev/sessions/state/).

### Active-slot uniqueness and concurrent requests

- **Decision**: Keep the existing unique partial index `ux_active_slot` on `(doctor_id, date, time) WHERE status = 'booked'` as the authoritative no-double-booking rule. Availability reads may improve the user experience but cannot reserve a time. Booking inserts the appointment and translates a unique-slot conflict to "slot just taken". On `IntegrityError`, verify that the attempted slot now has an active booking before mapping the failure; propagate unrelated constraint failures. Handle SQLite busy/locked errors separately with a short bounded retry or clear retry message.
- **Additional rule**: A patient cannot create a second active appointment with the same doctor. The inspected seed contains existing patient/doctor pairs with multiple active records, so a new global unique index on `(mr_no, doctor_id)` would fail on initialization and alter existing data. Enforce this rule in the clinic database using one `BEGIN IMMEDIATE` transaction that checks for the patient's active appointment and performs the insert while holding SQLite's write reservation. The existing slot unique index remains the final slot-level invariant.
- **Rationale**: A unique partial index atomically rejects duplicate active slots even when different patients race. SQLite serializes writers, so a database transaction around the same-doctor check and insert also prevents two simultaneous requests from the same patient from passing that check. `aiosqlite` serializes work on one connection but does not make independent connections or processes globally atomic; SQLite's transaction/index provide the invariant.
- **Alternatives considered**: Check-then-insert without a transaction/constraint (rejected because simultaneous requests can both pass the check); add a same-doctor unique index to the seed (rejected because current seed records already violate it); map every database error to slot-taken (rejected because it hides storage failures).
- **Sources**: [SQLite partial indexes](https://www.sqlite.org/partialindex.html), [SQLite transactions](https://www.sqlite.org/lang_transaction.html), [SQLite result codes](https://www.sqlite.org/rescode.html), [Python sqlite3 exceptions](https://docs.python.org/3/library/sqlite3.html#sqlite3.IntegrityError), [aiosqlite documentation](https://aiosqlite.omnilib.dev/en/stable/).

### Clinic database and query policy

- **Decision**: Use the provided clinic database's `v_free_slots` and `v_upcoming` views for availability and patient appointment reads, and query doctors/patients/appointments through narrowly scoped read-only functions. Keep booking/cancellation writes in the clinic database. The staff search page uses read-only access and returns only fields needed for staff work.
- **Rationale**: The repository's seed database already defines doctor schedules, holidays, patient and appointment tables, the active-slot index, and the requested views. Reusing them avoids duplicate schedule logic. Patient-facing queries always include the session MR number; staff results omit sensitive patient fields such as CNIC, allergies, and chronic conditions unless a separately approved use case requires them.
- **Alternatives considered**: Rebuild availability rules in agent code (rejected because it duplicates the dataset's view logic); expose arbitrary SQL to the model (rejected because it bypasses authorization and read-only constraints).
- **Source**: Repository `clinic.db` schema, inspected during planning.

### Streamlit interface and appearance

- **Decision**: Build a Streamlit patient chat page with patient picker, conversation list/reopen controls, and a state/memory panel; provide a separate read-only staff database search page. Use a light theme with white backgrounds and aqua accents, dark readable text, and clear success/error statuses.
- **Rationale**: Streamlit supports chat message/input widgets, per-view session state, multipage apps, and configurable light-theme colors. Persistent conversation history is loaded from ADK storage rather than relying on Streamlit session state, which resets after reload.
- **Alternatives considered**: Store transcripts only in Streamlit session state (rejected because it resets on page reload and does not meet restart persistence); add custom frontend components (rejected to keep the two-hour build scope small).
- **Sources**: [Streamlit chat messages](https://docs.streamlit.io/develop/api-reference/chat/st.chat_message), [Streamlit session state](https://docs.streamlit.io/develop/api-reference/caching-and-state/st.session_state), [Streamlit theme colors](https://docs.streamlit.io/develop/concepts/configuration/theming-customize-colors-and-borders).

## Assumptions and Resolved Questions

- Use Python 3.11 or later, Google ADK's database extra, LiteLLM, Streamlit, and pytest. Pin compatible package versions during implementation; the initial two-hour build does not target deployment.
- The existing tracked database is at repository root as `clinic.db`, while the requested source path is `data/clinic.db`. The implementation must establish `data/clinic.db` as the canonical seed path (move/copy the existing file without changing its schema) before creating the first runtime copy.
- A staff page requires a separate staff-only access gate; for the local demo, use a configured access code rather than treating the patient picker as staff authorization. Do not hard-code the code in source.
- The app is a single-server local demonstration. SQLite serializes writes; the required guarantee is correctness under simultaneous requests, not a specified high-throughput capacity.
- A patient can have at most one active appointment per doctor, matching the feature specification's edge case. Cancelled and already occurred appointments do not count as active.
