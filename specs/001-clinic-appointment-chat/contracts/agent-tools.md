# Patient Agent Tool Contract

The patient agent registers exactly five function tools. The ADK tool context supplies the authenticated `user_id` and session state; patient identity is not a model-controlled argument. Each tool validates input and returns a concise result the assistant can explain. Tools do not provide clinical advice.

## Shared Rules

- Require an active patient session initialized by selecting an existing patient.
- Resolve the patient MR number from trusted ADK invocation/session context for every patient-specific read or write.
- Do not accept MR number, patient name, or identity claims from model arguments as authorization.
- Treat tool output as untrusted input to the model; revalidate before any subsequent side effect.
- Return "not found" for missing and foreign appointment IDs without revealing which case occurred.
- Use explicit conflict results; do not turn general database errors into false success or "slot just taken".

## Tools

### 1. `find_availability`

- **Purpose**: Find doctors and free appointment times for a specialty and date.
- **Arguments**: `specialty` (required text), `date` (required clinic date).
- **Identity**: Not needed for the read; does not access patient records.
- **Reads**: Doctor data and `v_free_slots`, respecting clinic schedule, leaves, holidays, and current clinic date.
- **Returns**: Matching doctor names, specialty, fee, available times, or a no-availability result. Sunday requests resolve to the next working day.
- **Side effects**: None. Results are advisory, not held reservations.

### 2. `book_appointment`

- **Purpose**: Book one selected free time for the logged-in patient.
- **Arguments**: `doctor_id`, `date`, and `time`. Patient identity is deliberately absent.
- **Identity**: MR comes from the authenticated ADK session/user context.
- **Validation**: Confirm the doctor and offered time exist, the time is still free, and the patient has no active appointment with that doctor. Validate and insert within one `BEGIN IMMEDIATE` transaction.
- **Returns**: On success, confirmation and appointment ID. If the slot unique index wins a race, return "slot just taken". If the patient already has an active appointment with the doctor, explain that another active appointment with that doctor cannot be booked.
- **Side effects**: Inserts one booked appointment and refreshes `user:last_appointment` from the database.

### 3. `list_my_appointments`

- **Purpose**: Show the logged-in patient's appointments and current statuses.
- **Arguments**: Optional date range/status filter.
- **Identity**: MR comes from the authenticated session.
- **Reads**: Appointment rows matching that MR only.
- **Returns**: Appointment IDs, doctor/specialty, date/time, fee, and status; no other patient's rows.
- **Side effects**: None; may refresh the trusted `user:last_appointment` summary from database results.

### 4. `cancel_my_appointment`

- **Purpose**: Cancel one eligible appointment owned by the logged-in patient.
- **Arguments**: `appointment_id`. Patient identity is deliberately absent.
- **Identity**: MR comes from the authenticated session.
- **Validation**: Update only if both appointment ID and session MR match and status is `booked`; otherwise return "not found" or a clear already-cancelled result without disclosing foreign ownership.
- **Returns**: Cancellation confirmation or safe failure result.
- **Side effects**: Changes the appointment status to `cancelled`, releasing its slot in `v_free_slots`.

### 5. `get_my_patient_facts`

- **Purpose**: Provide the assistant with the selected patient's saved name, MR, and latest booking summary when asked for remembered information.
- **Arguments**: None.
- **Identity**: MR comes from the authenticated session.
- **Reads**: The matching patient's own user-scoped ADK state and, when needed, the clinic DB for a current latest-booking fact.
- **Returns**: Only `user:patient_name`, `user:mr_no`, and `user:last_appointment` values belonging to the current session.
- **Side effects**: None.

## Staff Search Separation

Staff search is not an agent tool. The staff page performs its own read-only database queries after staff access is validated. This prevents patient chat from becoming a path to staff-level search or arbitrary clinic data.
