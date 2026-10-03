# Feature Specification: Clinic Appointment Chat

**Feature Branch**: `001-clinic-appointment-chat`

**Created**: 2026-10-03

**Status**: Draft

**Input**: User description: "A chat assistant for a Peshawar clinic where many patients book, view and cancel appointments at the same time, are remembered across conversations, and nothing is lost on restart. Staff can search the clinic data. Buildable in 2 hours."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Book by chat (Priority: P1)

A logged-in patient asks for a specialty and date, reviews available doctors, fees, and free appointment times, then books a chosen time and receives its appointment ID.

**Why this priority**: Booking is the clinic's primary patient task and supplies the core value of the assistant.

**Independent Test**: Log in as a patient, ask for a specialty and working date, choose an offered time, and verify a confirmation and appointment ID are returned.

**Acceptance Scenarios**:

1. **Given** a logged-in patient and a specialty with doctors, **When** the patient asks for that specialty and a working date, **Then** the assistant shows matching doctors, their fees, and their free times on that date.
2. **Given** an offered free time, **When** the patient selects it, **Then** exactly one appointment is created for that patient and the assistant confirms it with an appointment ID.
3. **Given** a Sunday request, **When** the patient asks for availability, **Then** the assistant offers the next working day with available appointments.

---

### User Story 2 - Prevent double booking (Priority: P1)

Patients can book at the same time without claiming the same appointment time, and each patient's conversation and records stay private from other patients.

**Why this priority**: Duplicate bookings and cross-patient data exposure undermine safety and trust in the clinic.

**Independent Test**: Use two patient sessions to submit a booking for one free time simultaneously, then verify one booking succeeds and the other receives a "slot just taken" response; verify neither patient can access the other's conversation or appointment.

**Acceptance Scenarios**:

1. **Given** one free time and two simultaneous booking requests from different patients, **When** both requests complete, **Then** exactly one succeeds and the other is told "slot just taken".
2. **Given** two logged-in patients, **When** either tries to open the other's conversation or appointment, **Then** no other patient's data is shown and an appointment ID belonging to another patient is reported as "not found".
3. **Given** a patient with an active appointment with a doctor, **When** that patient tries to book a second active appointment with the same doctor, **Then** the booking is refused.

---

### User Story 3 - Remember conversations and patient facts (Priority: P2)

Patients can reopen their conversation history after a restart and start a new conversation while the assistant carries forward that patient's basic facts and last booking.

**Why this priority**: Continuity prevents patients from repeating information and lets them refer to prior bookings naturally.

**Independent Test**: Book an appointment, restart the application, reopen the old conversation to inspect its history, then start a new conversation and ask what was booked last time.

**Acceptance Scenarios**:

1. **Given** a saved conversation, **When** its owner reopens it after an application restart, **Then** its previous messages are shown in order.
2. **Given** a patient with a prior booking, **When** the patient starts a new conversation and asks what they booked last time, **Then** the assistant reports that patient's latest booking without asking for an MR number.
3. **Given** a patient viewing the memory display, **When** the display is opened, **Then** remembered items are grouped by app, patient, conversation, and current turn lifetime, and patient-specific items belong only to that patient.

---

### User Story 4 - View, cancel, and search clinic records (Priority: P3)

Patients can review and cancel their own upcoming appointments. Authorized staff can search clinic data and see assistant-created bookings and cancellations.

**Why this priority**: Patients need control of their appointments, and staff need a current view of clinic activity.

**Independent Test**: View and cancel a patient's upcoming appointment, verify its time becomes available, then search as staff and verify the appointment's updated status appears.

**Acceptance Scenarios**:

1. **Given** a logged-in patient with appointments, **When** the patient asks to view appointments, **Then** only that patient's appointments and their current statuses are shown.
2. **Given** a patient's upcoming appointment, **When** the patient cancels it, **Then** the appointment is marked cancelled and its time becomes available again.
3. **Given** an appointment owned by another patient, **When** a patient attempts to cancel it, **Then** the system reports "not found" and leaves it unchanged.
4. **Given** an authorized staff user on the search page, **When** they filter by specialty, date, patient, or status, **Then** matching doctors, free times, patients, or appointments are shown, including bookings and cancellations made by the assistant.
5. **Given** a staff user on the search page, **When** they inspect or filter results, **Then** the search page does not create, change, or cancel clinic records.

---

### Edge Cases

- A Sunday availability request offers the next working day according to the clinic schedule.
- If another request books a time after it was displayed, the patient receives "slot just taken" and can request fresh availability.
- An unknown, malformed, or other patient's appointment ID is reported as "not found" without revealing whether another patient's record exists.
- A patient cannot have more than one active appointment with the same doctor.
- When no doctor, date, or free time matches, the assistant states that no availability was found and offers a useful next step.
- Emergency symptoms prompt the patient to call 1122; the assistant does not proceed with appointment booking for that emergency request.
- Requests for diagnosis, medicine, or treatment advice are declined; the assistant may suggest contacting a relevant specialty without giving clinical advice.
- On a restart, persisted conversations, patient facts, and appointment status remain available.
- A persistence or validation failure does not produce a false booking confirmation or expose another patient's information.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Users MUST select a patient from the clinic dataset to establish a simulated logged-in patient session. All patient identity used for actions MUST come from that session, never from assistant output or a patient ID supplied in chat.
- **FR-002**: The system MUST keep conversations separate by patient and conversation ID, and let a patient start a conversation or reopen that patient's existing conversation.
- **FR-003**: The system MUST find doctors by specialty and show each matching doctor's fee and available times for a requested date.
- **FR-004**: The system MUST create and cancel appointments only for the patient identified by the logged-in session, and MUST confirm successful bookings with an appointment ID.
- **FR-005**: The system MUST ensure that concurrent requests for the same doctor and appointment time result in at most one active booking; losing requests MUST receive the "slot just taken" response.
- **FR-006**: The system MUST refuse a patient's attempt to hold more than one active appointment with the same doctor.
- **FR-007**: The system MUST carry each patient's name, MR number, and latest booking across that patient's conversations without requiring the patient to provide an MR number in chat.
- **FR-008**: The system MUST preserve conversations, patient facts, and bookings across application restarts, including conversation history and current appointment status.
- **FR-009**: The interface MUST show remembered information grouped by app, patient, conversation, and turn lifetime, and MUST not expose one patient's memory to another patient.
- **FR-010**: The system MUST provide an authorized staff search page for doctors, free times, patients, and appointments, with filters for specialty, date, patient, and status. Search MUST be read-only and reflect assistant-created changes.
- **FR-011**: The assistant MUST NOT provide diagnoses, medicine recommendations, or treatment advice. For emergencies it MUST direct the user to call 1122 and MUST NOT continue booking for the emergency request.
- **FR-012**: The system MUST validate requests and assistant outputs before actions, enforce identity and safety rules outside the assistant, and fail safely when validation or persistence fails.
- **FR-013**: The system MUST offer the next clinic working day when availability is requested for Sunday.
- **FR-014**: A patient MUST be able to view that patient's appointments and current statuses; appointments belonging to other patients MUST NOT be returned.

### Key Entities *(include if feature involves data)*

- **Doctor**: A clinic provider with a specialty, fee, and working schedule that defines appointment availability.
- **Patient**: A person in the clinic dataset with an MR number and a simulated authenticated session identity.
- **Appointment**: A scheduled time linking one doctor and one patient, with a date, time, and current status.
- **Conversation**: A patient's distinct chat history, addressable by conversation ID and retained across restarts.
- **Memory item**: A remembered fact or message associated with app, patient, conversation, or turn lifetime.
- **Staff user**: An authorized clinic user who can search clinic records without changing them through the search page.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A patient can find a specialty, choose a displayed free time, and receive an appointment ID in one chat session.
- **SC-002**: In a simultaneous two-patient attempt for one time, exactly one active appointment exists afterward and the other patient receives "slot just taken".
- **SC-003**: After an application restart, all previously saved conversation messages and appointment records used in the test remain available with their prior order and status.
- **SC-004**: In cross-patient access checks, zero conversations, memories, or appointments belonging to another patient are disclosed or changed.
- **SC-005**: A cancelled appointment's time appears as free in the next availability search, and its cancelled status appears in staff search.
- **SC-006**: All tested emergency prompts direct users to call 1122, and all tested requests for diagnosis or medicine advice are declined without clinical advice.
- **SC-007**: A staff search filtered by each supported field returns only records matching all applied filters and leaves clinic records unchanged.
- **SC-008**: The defined P1 booking and collision flows can be demonstrated end to end within the stated two-hour build constraint.

## Assumptions

- A fictional, pre-built clinic dataset is available in `clinic.db`, including doctors, specialties, fees, schedules, and patients.
- Login is simulated by selecting a patient from that dataset; the selected patient establishes the session identity.
- Staff use a distinct authorized staff session to access the search page; patients cannot use staff search.
- The clinic schedule defines working days and appointment durations; Sunday is closed. The next working day is the earliest following date on which the clinic schedule offers appointments.
- "Active appointment" means an appointment that is not cancelled and has not already occurred.
- A patient may have at most one active appointment with a given doctor, as specified by the edge case.
- Memory lifetime grouping is informational: app-lifetime values apply to all users, patient-lifetime facts belong to one patient, conversation-lifetime items belong to one conversation, and turn-lifetime items apply only to the current exchange.
- The first release is a single-server demonstration designed to fit a two-hour build. Appointment holding, rescheduling, conversation summarization, and deployment are out of scope.
