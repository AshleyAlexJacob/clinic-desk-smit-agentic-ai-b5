# User Interface Contract

## Patient Chat Page

1. **Patient selection**: A patient picker selects an existing dataset record and establishes the server-side patient session. The selected MR is passed to ADK as `user_id`; the model cannot change it.
2. **Conversation control**: Show only the selected patient's conversations. Let the patient create a conversation or reopen one of that patient's saved IDs. Changing patient selection clears the visible transcript and reloads sessions for the newly selected MR.
3. **Chat**: Display user and assistant messages in order. The assistant can search availability, book, list, and cancel via the five tools. Show appointment IDs after successful bookings.
4. **Safety states**: Emergency requests visibly direct the patient to call 1122 and stop booking in that request. Diagnosis, medicine, and treatment advice requests receive a brief refusal with an optional specialty suggestion.
5. **Memory panel**: Show state grouped by app, patient, conversation, and current turn. Clearly identify temporary free-slot data as turn-local and advisory. Do not show hidden reasoning.
6. **Visual theme**: Use a light theme with white page surfaces (`#FFFFFF`), pale aqua secondary surfaces (`#E7F7F5`), a darker aqua accent for focus/actions (`#00696E`), and dark readable text (`#173435`). Success, warning, and error messages use distinct accessible status colors.

## Staff Search Page

- Require a separate staff access gate configured outside source code; patient selection does not grant staff access.
- Provide read-only filters for specialty, date, patient (MR/name), and appointment status.
- Show matching doctors, free times, patients, and appointments, including assistant changes.
- Show only necessary patient fields (MR number and name for identification); do not expose CNIC, allergy, or chronic-condition columns by default.
- Provide no create, edit, cancel, or arbitrary SQL controls. Search results and filters must not write to the clinic database.

## User-Facing Outcomes

- Free time selected: show a confirmation and appointment ID.
- Slot taken by a simultaneous request: show exactly "slot just taken" and allow a fresh availability search.
- Foreign/malformed appointment ID: show "not found" without revealing whether another patient's appointment exists.
- Successful cancellation: confirm cancellation and make the time available in the next search.
- Data/session service failure: show a safe retry message; never claim an uncommitted booking succeeded.
