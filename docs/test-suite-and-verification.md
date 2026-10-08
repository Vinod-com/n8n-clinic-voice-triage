# 🧪 Test Suite & Verification Matrix

This document catalogs the integration payloads, edge-case scenarios, and verification assertions used to validate the AI Voice-Agent workflow across **n8n**, **Supabase**, and **Google Calendar**.

---

## 1. Multi-Doctor Same-Day Contention Test

### Scenario
A patient (Rahul Rampuria) has an existing 3:00 PM appointment with **Dr. Smith** and an upcoming 4:00 PM appointment with **Dr. Mark**. The patient calls to reschedule Dr. Mark's appointment to 5:00 PM.

### Inbound Payload
```json
{
  "message": {
    "transcript": "Hi, this is Rahul Rampuria. I need to reschedule my consultation with Dr. Mark today to 5:00 PM.",
    "durationSeconds": 14,
    "endedReason": "customer-ended-call",
    "customer": {
      "number": "+919830127501"
    }
  }
}
Verification Assertions
Clinician Isolation: Query must filter by doctor_id=eq.2 (Dr. Mark) so Dr. Smith's 3:00 PM slot is completely ignored.

Temporal Safety: Uses appointment_start=gte.{{ $now.toISO() }} to ignore any earlier completed/expired morning bookings.

Google Calendar: Only Dr. Mark's secondary calendar (...ade4e4@group.calendar.google.com) receives the Event: Update PATCH request.

Supabase Status: Row status updates to rescheduled with updated timestamps; Dr. Smith's appointment remains confirmed.
2. Cancellation & Event Removal Test
Scenario
The patient calls to cancel their 5:00 PM appointment with Dr. Mark.

Inbound Payload
2. Cancellation & Event Removal Test
Scenario
The patient calls to cancel their 5:00 PM appointment with Dr. Mark.

Inbound Payload
{
  "message": {
    "transcript": "Hi, this is Rahul Rampuria. I need to cancel my appointment with Dr. Mark today.",
    "durationSeconds": 14,
    "endedReason": "customer-ended-call",
    "customer": {
      "number": "+919830127501"
    }
  }
}
Verification Assertions
GCal HTTP Method: Sends a direct DELETE request to Google Calendar REST endpoint:
Verification Assertions
GCal HTTP Method: Sends a direct DELETE request to Google Calendar REST endpoint:
[https://www.googleapis.com/calendar/v3/calendars/](https://www.googleapis.com/calendar/v3/calendars/){{ calendar_id }}/events/{{ event_id }}
Event Removal: Dr. Mark's event is purged from Google Calendar.

Database State: Supabase updates appointments.status to cancelled.

Collateral Check: Dr. Smith's separate appointment remains untouched on both Supabase and Dr. Smith's calendar.
3. PostgREST Query Construction Reference
When querying Supabase in n8n HTTP Request nodes without causing relational join errors:

Reschedule / Cancellation Query Template:
https://<PROJECT_REF>.supabase.co/rest/v1/appointments?select=id,patient_id,doctor_id,appointment_start,appointment_end,calendar_event_id,status,patient_phone,doctors(id,name,calendar_id),patients(id,patient_name,telegram_chat_id)&patient_phone=like.*{{ $('Format Ingest Payload').first()?.json?.phone_last_10 }}*&doctor_id=eq.{{ $json.id }}&status=in.(confirmed,rescheduled)&appointment_start=gte.{{ $now.toISO() }}&order=appointment_start.asc&limit=1
Critical Query Parameters:ParameterPurposepatient_phone=like.*{{ phone_last_10 }}*Resilient phone matching regardless of country code or formatting.doctor_id=eq.{{ $json.id }}Native foreign key filtering preventing multi-doctor cross-talk.appointment_start=gte.{{ $now.toISO() }}Filters out past appointments so expired slots are never targeted.status=in.(confirmed,rescheduled)Restricts query to active bookings only.
