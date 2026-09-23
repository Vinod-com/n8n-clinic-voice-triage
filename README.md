# Clinic Voice Triage & Automated Booking Workflow

An automated clinic voice triage, booking, and cancellation pipeline built in [n8n](https://n8n.io/). The system ingests post-call webhooks from [Vapi](https://vapi.ai/), queries and synchronizes appointment records in [Supabase](https://supabase.com/), manages events on [Google Calendar](https://calendar.google.com/), and sends real-time dispatch alerts to staff and patients via [Telegram](https://telegram.org/).

---

## Architecture & Flow Overview

1. **Ingestion (`Vapi Post-Call Ingest`):** Receives end-of-call webhooks containing call metadata, transcript strings, and caller details.
2. **Entity & Intent Extraction (`Information Extractor`):** Extracts caller name, phone number, and action intent (e.g., booking, cancellation, rescheduling).
3. **Database Lookups (`Supabase`):** Verifies whether matching active appointments exist for the caller.
4. **Conditional Routing (`If (Booking Found?)`):**
   - **True Path:** Proceeds with business logic (e.g., removing events from Google Calendar, flipping Supabase status to `cancelled`, dispatching confirmation alerts).
   - **False / Exception Path:** Routes to a specialized fallback node (`Format Fallback Data`) configured to run safely without breaking empty-array item lineages, then dispatches failure notifications.
5. **Parallel Alerting (`Telegram Bots`):** Sends concurrent Markdown alerts to both clinic staff and patient channels without payload collisions.
# 🏥 Clinic Voice AI Assistant — Engineering Handover & Milestone Report

**Date:** September 23, 2026 (16:50 IST)  
**Repository State:** Main workflow published and committed  
**Scope:** Completion of Reschedule Pipeline & Database Preparation for Multi-Doctor Scaling

---

## 📌 Executive Summary

All core branches for single-doctor appointment management (New Booking, Reschedule Success, Slot Conflict Rejection, and No Prior Booking Found Fallback) have been fully implemented, calibrated, tested end-to-end, and published live in n8n. 

In addition, the architectural foundation for multi-doctor scaling has been laid: the Supabase `doctors` registry table has been created, populated with initial doctor records (`Dr. Smith` and `Dr. Mark`), and Google Calendar separation rules have been defined.

---

## 🛠️ Key Achievements & Verifications Completed Today

### 1. Reschedule Fallback 2: "No Prior Booking Found"
- **Problem Resolved:** Earlier test runs inadvertently queried cached phone numbers from previous executions (Row #65 / Rahul), causing the router to evaluate `True` rather than recognizing an unknown caller.
- **Dynamic Supabase Query Fixed:** Updated `Find Booking to Reschedule` HTTP URL to resolve dynamically from multiple candidate payload keys with safe URL encoding:
  ```text
  [https://optbmkbhrrzavrogmoyl.supabase.co/rest/v1/appointments?patient_phone=eq](https://optbmkbhrrzavrogmoyl.supabase.co/rest/v1/appointments?patient_phone=eq).{{ encodeURIComponent($('Information Extractor').first().json.output?.patient_phone || $('Information Extractor').first().json.patient_phone || $('Vapi Post-Call Ingest').first().json.message?.customer?.number || '') }}
---

## Prerequisites

- **n8n:** Self-hosted (Docker) or Cloud instance.
- **Vapi Account:** Configured voice assistant pointing end-of-call server webhooks to your n8n webhook URL.
- **Supabase Project:** PostgreSQL database storing appointment rows (with columns for `caller_phone`, `patient_name`, `status`, `calendar_event_id`, etc.).
- **Google Cloud Console:** OAuth2 credentials or Service Account with Google Calendar API scope enabled.
- **Telegram:** Two bot tokens created via [@BotFather](https://t.me/botfather) (one for clinic staff triage alerts, one for patient updates).

---

## Setup & Installation

### 1. Import Workflow to n8n
1. Clone or download this repository.
2. Open your n8n workspace.
3. Click the menu icon (**`...`**) in the top-right corner > **Import from File**.
4. Select `clinic-voice-triage.json`.

### 2. Configure Credentials
Update the following credentials in your n8n instance:
- **Supabase API:** Add your Supabase Project URL and Service Role Key. *(Note: Ensure secrets are configured via n8n Credential Manager and not hardcoded directly inside HTTP node headers).*
- **Google Calendar OAuth2:** Authenticate with your Google account.
- **Telegram Bot 1 (Staff):** Enter the Bot Token and destination Chat/Channel ID.
- **Telegram Bot 2 (Patient):** Enter the Bot Token and dynamic/static Chat ID.

### 3. Activate Webhook
- Activate the workflow in n8n.
- Copy the **Production Webhook URL** from `Vapi Post-Call Ingest` and paste it into the **Server URL** setting in your Vapi assistant dashboard.

---

## Security & Sanitization

- **Credentials Sanitized:** Ensure that raw tokens (`sb_secret_*`, OAuth client secrets, or private API keys) are never committed to version control. Use environment variables or n8n Credential objects.
---

## Update Log: Conflict Detection & Branching Fixes
- **Conflict Evaluation:** Calibrated `If (New Slot is Free?)` using strict Boolean logic to accurately separate available slots from collisions.
- **Routing Fix:** Disabled `Always Output Data` on conditional nodes to eliminate duplicate executions and prevent dummy records on inactive branches.
- **Dynamic Slot Preparation:** Updated `Prepare Slot Dates` to dynamically map slot windows (`timeMin` / `timeMax` buffers) from Supabase and extractor inputs.
- **Notification Templates:** Fixed Telegram staff rejection template mappings to resolve `Invalid DateTime` and missing patient details.
- **Resilience:** Configured auto-retry policies on LLM extraction nodes to handle upstream provider downtime.
