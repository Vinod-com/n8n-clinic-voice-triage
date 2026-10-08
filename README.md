# 🏥 AI Voice-Agent Clinic Booking & Operations Engine

An automated, multi-doctor clinic appointment management system built with **n8n**, **Supabase (PostgreSQL)**, and **Google Calendar**.

Designed to process voice call ingest payloads (e.g., Vapi, Retell, Twilio, or custom webhooks), extract conversational intent, enforce clinical scheduling guardrails, and execute atomic bookings, reschedules, and cancellations across isolated clinician calendars without slot cross-contamination.

---

## 📌 Table of Contents
- Architecture Overview
- Key Features & Guardrails
- Workflow Logic & Branching
- Database Schema (Supabase)
- Technical Architecture & Hard-Won Edge Cases
- Setup & Deployment Guide
- Testing & Verification Payloads
- Technical Specifications & Schemas

---

## 🏗 Architecture Overview

* **Inbound Voice Payload**: Ingested via webhook (telephony transcript and caller phone number).
* **Format & Normalize**: Strips country codes and standardizes phone format to the last 10 digits.
* **Information Extractor**: LLM extracts patient intent, caller name, doctor name, and appointment timestamps into structured JSON.
* **Lookup Doctor**: Resolves clinician database record to obtain native `doctor_id` and isolated Google Calendar ID.
* **Conditional Branching**:
  * **New Booking**: Enforces clinic operating hours and future-date guardrails, verifies calendar availability, creates Google Calendar event, and records confirmed booking in Supabase.
  * **Reschedule Branch**: Applies composite filters (`doctor_id` and `appointment_start >= now()`) to find the active upcoming appointment, updates Google Calendar slot times, and marks appointment rescheduled.
  * **Cancellation Branch**: Directs deletion request strictly to target practitioner's calendar and updates database status to cancelled.

---

## ⚡ Key Features & Guardrails

1. **Multi-Doctor Calendar Isolation**:
   - Manages independent Google Calendars per practitioner (`@group.calendar.google.com`).
   - Ensures lookups, conflict detection, updates, and cancellations only target the requested doctor’s schedule.
2. **Deterministic Intent Extraction**:
   - Extracts caller phone, patient name, doctor, intent (`book`, `reschedule`, `cancel`), and requested slot times from unstructured conversational transcripts.
3. **Clinical Scheduling Guardrails**:
   - **Temporal Bounds**: Rejects past time slots relative to execution run time (`is_valid_slot = false`).
   - **Operating Hours**: Enforces working hours (e.g., 9:00 AM – 6:00 PM).
   - **Clinic Closures**: Blocks Sunday bookings and redirects to staff triage.
4. **Resilient Temporal Lookups**:
   - Uses dynamic timestamps (`$now.toISO()`) to ignore expired morning slots when querying active records for same-day afternoon modifications.
5. **Two-Way State Synchronization**:
   - Synchronizes Google Calendar event states with Supabase database rows atomically, maintaining `calendar_event_id` references for clean lifecycle tracking.

---

## 🔄 Workflow Logic & Branching

### 1. Ingest & Extraction
* Normalizes the caller's phone number to a consistent format (last 10 digits).
* Evaluates intent, clinician, and timestamp using LLM node.
* Queries Supabase `doctors` table by name to resolve `doctor_id` and specific Google Calendar ID.

### 2. New Booking Branch
* Validates requested time slot against operating hours and past-date filters.
* Verifies clinician calendar availability.
* Creates Google Calendar event with patient details.
* Inserts row into `appointments` with status `confirmed`.

### 3. Reschedule Branch
* Filters active appointments for the patient specifically associated with the requested `doctor_id` where `appointment_start >= $now.toISO()`.
* Bypasses non-target appointments even if the patient has earlier bookings with other clinicians on the same day.
* Updates Google Calendar event start/end times and updates the Supabase record to `rescheduled`.

### 4. Cancellation Branch
* Queries active target appointment using native `doctor_id` and temporal filters.
* Executes Google Calendar deletion via REST API (`DELETE /calendars/{calendar_id}/events/{event_id}`).
* Sets appointment status to `cancelled` in Supabase.

---

## 🗄 Database Schema (Supabase)

### `doctors` Table
| Column | Type | Description |
|---|---|---|
| `id` | `int8` (PK) | Doctor identifier |
| `name` | `text` | Clinician name (e.g., Dr. Smith, Dr. Mark) |
| `specialty` | `text` | Medical specialty |
| `calendar_id` | `text` | Full Google Calendar ID (`...abc@group.calendar.google.com`) |

### `patients` Table
| Column | Type | Description |
|---|---|---|
| `id` | `int8` (PK) | Patient identifier |
| `patient_name` | `text` | Full name of patient |
| `patient_phone`| `text` | Contact number |
| `telegram_chat_id` | `text` | Optional notification ID |

### `appointments` Table
| Column | Type | Description |
|---|---|---|
| `id` | `int8` (PK) | Appointment record ID |
| `patient_id` | `int8` (FK) | Reference to `patients.id` |
| `doctor_id` | `int8` (FK) | Reference to `doctors.id` |
| `patient_phone`| `text` | Denormalized phone for rapid indexing |
| `appointment_start` | `timestamptz` | Slot start time |
| `appointment_end` | `timestamptz` | Slot end time |
| `calendar_event_id`| `text` | Target Google Calendar Event UID |
| `status` | `text` | `confirmed`, `rescheduled`, `cancelled` |

---

## 💡 Technical Architecture & Hard-Won Edge Cases

### 1. Supabase PostgREST Embedded Join Traps
* **Problem**: Attempting to filter foreign table values with syntax like `&doctors.name=ilike.*...*` causes PostgREST errors unless explicit `!inner` join syntax is used.
* **Resolution**: Always resolve the clinician's internal `doctor_id` first via a dedicated `Lookup Doctor` step, then query the native column directly: `&doctor_id=eq.{{ $json.id }}`.

### 2. Multi-Doctor Same-Day Appointment Contention
* **Problem**: When a patient has multiple appointments on the same day across different doctors (e.g., Dr. Smith at 3 PM, Dr. Mark at 4 PM), sorting by `appointment_start.asc` without doctor isolation causes Dr. Smith's appointment to be modified accidentally when the user intended to reschedule Dr. Mark.
* **Resolution**: Strict composite filtering combining:
  1. `patient_phone=like.*{{ phone_last_10 }}*`
  2. `doctor_id=eq.{{ target_doctor_id }}`
  3. `appointment_start=gte.{{ $now.toISO() }}`
  4. `status=in.(confirmed,rescheduled)`

### 3. Google Calendar API Parameter Mode
* **Problem**: Modern n8n Google Calendar nodes fail with parameter errors or 404s when expressions are pasted into dropdown fields or when Calendar ID and Event ID are swapped.
* **Resolution**:
  - Always toggle Calendar and Event fields to **By ID** mode with the **Expression** tab active.
  - Ensure Calendar ID preserves the full `@group.calendar.google.com` suffix.
  - When using HTTP Request nodes for direct Google API calls, encode the calendar address via `encodeURIComponent()`.

---

## 🚀 Setup & Deployment Guide

1. **Supabase Setup**:
   - Create tables `doctors`, `patients`, and `appointments` using the schema above.
   - Insert doctor records with their dedicated secondary Google Calendar IDs.
2. **Google Calendar Permissions**:
   - Share secondary doctor calendars with the service account or authenticated Google account with edit/manage rights.
3. **n8n Workflow Import**:
   - In n8n, click **Workflows** > **Import from File**.
   - Select the workflow JSON file from this repository.
   - Attach your Supabase API credentials and Google Calendar OAuth2 credentials.
4. **Webhook Configuration**:
   - Point your telephony / voice agent webhook URL to the n8n Trigger endpoint.

---

## 🧪 Testing & Verification Payloads

### Test 1: Book Appointment
`{"message": {"transcript": "Hi, this is Rahul Rampuria. Can you please book an appointment for me with Dr. Smith today at 3:00 PM?", "customer": {"number": "+919830127501"}}}`

### Test 2: Reschedule Doctor
`{"message": {"transcript": "Hi, this is Rahul Rampuria. I need to reschedule my consultation with Dr. Mark today to 5:00 PM.", "customer": {"number": "+919830127501"}}}`

### Test 3: Cancel Doctor
`{"message": {"transcript": "Hi, this is Rahul Rampuria. I need to cancel my appointment with Dr. Mark today.", "customer": {"number": "+919830127501"}}}`

---

## 📄 Technical Specifications & Schemas

For detailed LLM JSON Input Schema (Draft-07), system prompts, clinical triage classifications, and mock voice fixtures, refer to:
- [System Prompt, JSON Schemas & Config](docs/system-prompt-and-schema.md)
