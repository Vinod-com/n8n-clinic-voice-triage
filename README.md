[Clinic Voice AI Agent System Prompt & Config.md](https://github.com/user-attachments/files/32703497/Clinic.Voice.AI.Agent.System.Prompt.Config.md)
# Clinic Voice AI Agent: System Prompts, Schemas & Test Payloads

This document defines the core prompt configuration, structured JSON schema, triage routing rules, and mock test fixtures for the AI Voice Agent orchestration layer (Vapi + Gemini + n8n + Supabase + Google Calendar + Telegram).

---

## 1. Information Extractor Node Configuration

### 1.1 JSON Input Schema (Draft-07)
Enforces deterministic extraction across clinical intents, symptom-to-specialty classification, and temporal expressions.

```json
{
  "$schema": "[http://json-schema.org/draft-07/schema#](http://json-schema.org/draft-07/schema#)",
  "type": "object",
  "properties": {
    "patient_intent": {
      "type": "string",
      "enum": ["book_appointment", "reschedule", "cancel", "emergency"],
      "description": "Primary clinical intent of the call."
    },
    "urgency": {
      "type": "string",
      "enum": ["standard", "emergency"],
      "description": "Strictly set to 'emergency' if acute symptoms are reported. Otherwise 'standard'."
    },
    "specialty": {
      "type": "string",
      "enum": ["General Medicine", "ENT", "Dentistry", "Pulmonology"],
      "description": "Clinical department mapped from symptoms: 'Dentistry' for tooth/gum issues; 'ENT' for ear, nose, throat; 'Pulmonology' for chest/asthma issues; 'General Medicine' for stomach ache, fever, cough, checkups, or unspecified issues."
    },
    "patient_name": {
      "type": "string",
      "description": "Full name of the patient mentioned in the transcript. If not stated, return an empty string."
    },
    "patient_phone": {
      "type": ["string", "null"],
      "description": "Caller phone number if explicitly spoken in audio, normalized to E.164. Otherwise null."
    },
    "doctor_name": {
      "type": ["string", "null"],
      "description": "Requested practitioner name if specified. Otherwise null."
    },
    "appointment_start": {
      "type": ["string", "null"],
      "description": "Target appointment start timestamp in ISO 8601 format. Otherwise null."
    },
    "appointment_end": {
      "type": ["string", "null"],
      "description": "Target appointment end timestamp in ISO 8601 format. Otherwise null."
    },
    "chief_complaint": {
      "type": ["string", "null"],
      "description": "Summary of symptoms or reason for the call. Otherwise null."
    }
  },
  "required": ["patient_intent", "urgency", "specialty", "patient_name"]
}
