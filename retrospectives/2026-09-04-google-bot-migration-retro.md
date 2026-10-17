# Retrospective: Google First-Class Bot Migration

## Executive Summary
This retrospective records the design and implementation process for migrating `allusio-ai/org-assistant`, its deployment automation scripts, and `synapto-plugin-google-chat` from human user OAuth credentials to first-class Google Chat App (Service Account) bot authentication.

## Iterative Evolution & Paradigm Shifts

### 1. Identity Separation (Host VM vs. Application Bot)
- **Initial Idea:** Use ambient Application Default Credentials (ADC) from host metadata servers when running on GCP.
- **Architectural Shift:** Lead Architect mandated strict isolation between the host VM infrastructure identity and the application bot identity.
- **Resolution:** The bot identity (`org-assistant@${PROJECT_ID}.iam.gserviceaccount.com`) is configured exclusively via explicit `GoogleServiceAccountCredentials` key JSON in configuration files. Host VM metadata fallback is prohibited.

### 2. People API Elimination
- **Initial Architecture:** `synapto-plugin-google-chat` queried `people.googleapis.com/v1/people/me` to determine the user ID of the human account.
- **Flaw Identified:** Added external network overhead, fragile permissions, and unnecessary coupling.
- **Resolution:** Bot identity and self-message checks match `sender.type == "BOT"` and `client_id` directly from configuration without calling the People API.

### 3. Pub/Sub Ingestion Streamlining
- **Initial Architecture:** `read.rs` called `maintain_subscriptions` to register workspace events dynamically.
- **Subagent Review Finding:** First-class Google Chat Apps configure Cloud Pub/Sub delivery natively in Google Cloud Console.
- **Resolution:** The dynamic subscription maintenance loop was deleted. Ingestion pulls directly from the provisioned `chat-events-sub`.

## Attribution
- **RFC Protocol & Core Design:** Main Coding Agent
- **Identity Isolation Directive & Service Account Naming (`org-assistant`):** Lead Architect
- **Channel Wiring & Native Pub/Sub Ingestion Analysis:** Zero-Context Subagent

## Verification
- Unit test `test_service_account_credentials_deserialization` passed.
- `cargo check` and `cargo clippy` passed on `synapto-plugin-google-chat` and `org-assistant` with 0 warnings and 0 errors.
