# Quickstart: Clinic Appointment Chat

This guide describes the local demonstration once implementation tasks are complete. It validates the primary user outcomes without changing the clinic seed database.

## Prerequisites

- Python 3.11 or later.
- An OpenAI API key for the configured `openai/gpt-4o` LiteLLM model.
- A local staff access code configured outside source code.
- The clinic seed file at `data/clinic.db`. The current repository copy is at root as `clinic.db`; the first implementation task must put the canonical source at the requested path.

## Start the Application

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
export OPENAI_API_KEY="<your-openai-api-key>"
export STAFF_ACCESS_CODE="<local-demo-staff-code>"
streamlit run app.py
```

On first launch, the app creates `runtime/sessions.db` and copies `data/clinic.db` to `runtime/clinic.db`. On later launches it reuses both runtime databases so session history and clinic changes survive restarts. Runtime files are local state and must not replace the seed database in version control.

## Acceptance Walkthroughs

### Patient booking and no double booking

1. Select a patient in the patient picker and start a conversation.
2. Ask for a specialty and working date; verify doctor, fee, and free times appear.
3. Choose a time; verify the response includes an appointment ID.
4. Open two separate browser tabs with different patients and submit the same free doctor/date/time as close together as possible.
5. Verify one request creates an appointment and the other receives exactly "slot just taken". Verify the database contains one active appointment for that slot.
6. Attempt a second active booking with the same doctor for the original patient; verify it is refused.

### Identity isolation, history, and restart

1. With one patient selected, create two conversations and book an appointment.
2. Ask for the last booking in the second conversation; verify it uses only that patient's appointment data and does not ask for MR number.
3. Verify the memory panel labels app, patient, conversation, and turn values separately.
4. Try a conversation ID and appointment ID belonging to another patient; verify "not found" with no details disclosed.
5. Stop and restart the app, reselect the original patient, and reopen the first conversation. Verify messages remain ordered and the booking still exists.

### Cancellation, staff search, and clinical safety

1. List a patient's appointments, cancel one upcoming appointment, and search availability again. Verify its time is free.
2. Open the staff page using the configured staff access code. Search by specialty, date, patient, and status; verify assistant-created bookings and cancellations are visible and search is read-only.
3. Ask an emergency question; verify the assistant directs the patient to call 1122 and does not proceed to booking.
4. Ask for a diagnosis or medicine recommendation; verify the assistant declines and provides no clinical advice.
5. Ask for Sunday availability; verify the next scheduled working date is offered.

## Automated Validation

```bash
python -m pytest -q
```

The suite should include a two-connection slot race, same-patient/same-doctor transaction race, patient/session isolation, cancellation releasing a slot, restart persistence, memory scope display, staff read-only behavior, and clinical safety cases. A database failure must not produce a booking confirmation.
