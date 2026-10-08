# 🧪 Test Suite & Verification Matrix

This document catalogs the integration payloads, edge-case scenarios, and verification assertions used to validate the AI Voice-Agent workflow across **n8n**, **Supabase**, and **Google Calendar**.

---

## 1. Multi-Doctor Same-Day Contention Test

### Scenario
A patient (Rahul Rampuria) has an existing 3:00 PM appointment with **Dr. Smith** and an upcoming 4:00 PM appointment with **Dr. Mark**. The patient calls to reschedule Dr. Mark's appointment to 5:00 PM.


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
---
## 2. Cancellation & Event Removal Test

### Scenario
The patient calls to cancel their 5:00 PM appointment with Dr. Mark.

### Inbound Payload
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

