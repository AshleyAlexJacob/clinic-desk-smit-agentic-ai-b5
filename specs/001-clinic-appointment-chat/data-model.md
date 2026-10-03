# Data Model: Clinic Appointment Chat

## Persistence Boundaries

| Store | Contents | Lifetime | Authority |
|---|---|---|---|
| `runtime/sessions.db` | ADK sessions, event history, scoped state, and conversation summary | Across restarts | Conversation history and conversational memory |
| `runtime/clinic.db` | Mutable appointments plus the copied clinic reference data | Across restarts | Patient, doctor, schedule, slot, and appointment facts |
| Streamlit session | Current selected patient, conversation ID, and display controls | Current browser view | UI selection only; never the source for patient authorization |
| `temp:` ADK state | Current invocation scratch data, including `free_slots` | Current invocation only | Advisory only; never the source of booking availability |

The runtime clinic file is copied from `data/clinic.db` only when it does not exist. Restarting the app must not recopy the seed over saved bookings or cancellations. At startup, update `app:clinic_today` and synchronize the runtime clinic `meta.today` value used by its views using the current Peshawar date. The repository currently has the seed at its root as `clinic.db`; implementation must establish the requested canonical `data/clinic.db` path first.

## Entities

### Doctor

- **Identity**: `doctor_id`
- **Fields**: name, specialty, consultation fee, follow-up fee, slot duration, qualifications, languages, room, rating.
- **Relationships**: Has schedule rows and leave rows; may have many appointments.
- **Validation**: Availability is calculated from schedule/slot records and clinic holidays/leaves. User-facing specialty search shows fee and available times.

### Doctor Schedule, Leave, and Holiday

- **Schedule identity**: schedule row ID, doctor ID, weekday, start time, end time.
- **Leave identity**: leave row ID, doctor ID, start date, end date, optional reason.
- **Holiday identity**: date.
- **Rules**: Sunday requests offer the next date with free slots, respecting the supplied clinic schedule, leaves, and holidays.

### Patient

- **Identity**: `mr_no`
- **Fields**: full name and the seed dataset's demographic/contact fields.
- **Session ownership**: the patient picker establishes the selected MR as ADK `user_id`; patient tool calls derive MR from trusted session context. Model-visible arguments cannot override it.
- **Privacy**: patient-specific tools scope reads/writes to the session MR. Patient-facing responses never return another patient's records. The staff page returns only fields needed for authorized search and does not expose CNIC, allergies, or chronic conditions by default.

### Appointment

- **Identity**: `appointment_id`
- **Fields**: `mr_no`, `doctor_id`, date, time, duration, visit type, status, fee, booking source, created time.
- **Relationships**: Belongs to one patient and one doctor.
- **States**: `booked` → `cancelled`, or `booked` → `completed` / `no_show` as maintained by the clinic dataset. Cancellation releases the time because free-slot lookup excludes only appointments with status `booked`.
- **Rules**: The active-slot unique index prevents two booked appointments from occupying one doctor/date/time. The clinic database write transaction checks that the same patient has no other active booking with the same doctor before inserting. The seed contains pre-existing repeat patient/doctor appointments; preserve these rows and reject additional bookings for such pairs.
- **Ownership**: Appointment lookup/cancellation includes both `appointment_id` and session `mr_no`; a missing or foreign appointment returns "not found".
- **Failure mapping**: An active-slot uniqueness conflict returns "slot just taken". Other constraint failures are not presented as slot conflicts. Busy/locked storage conditions receive a bounded retry or actionable retry message.

### Conversation Session

- **Identity**: ADK `app_name`, `user_id` (selected MR number), and `session_id` (conversation ID).
- **Fields**: ordered events, creation/activity information, and conversation-scoped state.
- **Rules**: Patients can create or reopen their own session IDs. Session lookup is always scoped by the selected MR. Full history remains persisted even when the model request is shortened.
- **History context**: `before_model_callback` presents a stored conversation summary plus at most the latest 20 content items. When older content needs summarization, the orchestration layer refreshes the summary before the callback and stores it at conversation scope; if summarization fails, it must not silently discard the older history.

### Memory / Scratchpad State

| State key | Lifetime | Meaning | Trust rule |
|---|---|---|---|
| `app:clinic_today` | Application | Clinic's current date reference | Recomputed on app startup; never accepted from model output |
| `user:mr_no` | Patient/user | MR number for the ADK user | Set by patient picker/session setup only |
| `user:patient_name` | Patient/user | Patient name for continuity | Loaded from clinic data for the selected MR |
| `user:last_appointment` | Patient/user | Most recent booking summary/ID | Derived from the clinic DB and refreshed after booking/listing |
| `last_viewed_doctor` | Conversation/session | Last doctor presented in this conversation | Display context only; revalidated against DB before booking |
| `conversation_summary` | Conversation/session | Short summary of older conversation messages | Derived from saved history; not a source of identity or appointment facts |
| `temp:free_slots` | Current invocation | Free-slot results used in the current assistant turn | Advisory scratchpad only; booking rechecks availability in a write transaction |

The memory panel labels values by app, patient, conversation, and turn lifetime. The scratchpad stores structured operational context only; it does not expose hidden model reasoning.

## Read and Write Paths

- **Availability**: Search `v_free_slots` by specialty and requested date; join doctor fee/name data as needed. Availability is not a reservation.
- **Patient appointment view**: Query appointments for the session MR and show current status.
- **Booking**: Validate inputs, begin an immediate write transaction, confirm there is no active appointment for the same MR/doctor, insert the appointment, commit, and return the generated ID. The existing unique slot index decides slot races.
- **Cancellation**: Update only the matching appointment ID owned by the session MR and currently eligible for cancellation; commit the status change so the free-slot view can offer the time again.
- **Staff search**: Use read-only queries with bounded filters for doctor, slot, patient, and appointment data. Search cannot mutate clinic records.
