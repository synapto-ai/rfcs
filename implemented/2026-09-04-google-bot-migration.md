# [MVP] Google First-Class Bot Migration for Google Plugins and Deployment Scripts

## 1. User-Centric Motivation & Goal
Users interact with the assistant in Google Workspace (Google Chat). Currently, the assistant uses personal OAuth credentials and acts as a clone of a human user. This causes confusion because messages from the assistant appear as messages from the user. Setup scripts also require manual OAuth consent screens and user-linked refresh token generation. Migrating to first-class Google Chat App (bot) authentication allows the assistant to operate under its own official bot identity. The assistant will receive direct mentions and space messages cleanly, post messages with its own avatar and name, and manage reactions without impersonating team members. The bot identity is strictly decoupled from the VM infrastructure identity to preserve security boundaries and identity isolation.

## 2. Executive Summary
- **Context:** `allusio-ai/org-assistant`, its deployment scripts (`scripts/setup_gcp_project.sh`), and `synapto-plugins/google-chat` connect to Google Cloud and Google Workspace APIs. Currently, authentication uses 3-legged OAuth refresh tokens associated with a human user account (`ASSISTANT_USER_EMAIL`).
- **Motivation:** Impersonating a human user creates security risks, requires manual OAuth consent flows, creates confusion in group spaces, and complicates infrastructure automation. Relying on ambient VM identities for bot logic conflates infrastructure privileges with application-layer bot capabilities.
- **Overarching Goals:** Standardize all Google Workspace bot operations on explicit, dedicated Google Chat Bot Service Account authentication. Maintain strict separation between the Infrastructure Identity (VM runner) and the Bot Identity (Google Chat App).
- **Purpose:** Replace human OAuth credentials in `synapto-plugin-google-chat` and `org-assistant/scripts` with a dedicated Google Cloud Service Account configured exclusively for Chat App authentication.
- **Problem:**
  - `google-chat` relies on `OAuthClientCredentials` (client ID, client secret, refresh token) and queries `people.googleapis.com` to resolve its identity.
  - `setup_gcp_project.sh` configures user OAuth consent screens, creates desktop OAuth clients, and binds Pub/Sub permissions to a human user account (`ASSISTANT_USER_EMAIL`).
- **Solution:**
  - Introduce an explicit `GoogleServiceAccountKey` configuration in `synapto-plugin-google-chat` for the Bot Identity (using `https://www.googleapis.com/auth/chat.bot` and `https://www.googleapis.com/auth/pubsub` scopes). Disallow ambient VM metadata fallback for the bot persona.
  - Update `org-assistant/scripts/setup_gcp_project.sh` to create and provision a dedicated Bot Service Account (`org-assistant@${PROJECT_ID}.iam.gserviceaccount.com`), generate its key, bind Pub/Sub subscriber permissions to this account, and remove all user OAuth setup steps.
  - Remove `ASSISTANT_USER_EMAIL` requirements from `scripts/load_client_config.sh`.

## 3. Current State Analysis
- **Runtime Authentication Flow:**
  - `GoogleChatConfig` contains `google_auth::oauth_client::OAuthClientCredentials` with `client_id`, `client_secret`, and `refresh_token`.
  - `OAuthClient` requests bearer tokens from `https://oauth2.googleapis.com/token` via `grant_type=refresh_token`.
- **Runtime Identity Resolution:**
  - The plugin queries `people.googleapis.com/v1/people/me` using the OAuth token to extract the user's resource name (`people/{user_id}`).
  - The plugin uses this human user ID to ignore self-messages and to detect mentions.
- **Infrastructure Setup Scripts:**
  - `scripts/setup_gcp_project.sh` creates an OAuth brand and outputs instructions to manually create a Desktop OAuth client and configure test users for `ASSISTANT_USER_EMAIL`.
  - The script binds `roles/pubsub.subscriber` on `chat-events-sub` to `user:$ASSISTANT_USER_EMAIL`.
  - `scripts/load_client_config.sh` requires `ASSISTANT_USER_EMAIL` in `encrypted.clients.json`.
- **Flaws:**
  - **Impersonation:** The system fakes human user identity instead of presenting a distinct bot persona.
  - **Manual Friction:** 3-legged OAuth requires interactive browser logins, manual consent grant, and refresh token extraction.
  - **Coupling to People API:** Resolving bot identity via People API adds unnecessary API surface and latency.
  - **Misconfigured Infrastructure IAM:** Granting Pub/Sub subscriber to a human user prevents automated headless Service Account execution.

## 4. Structural Hierarchy
- **Mission:** Explicit Separation of Bot Identity from Infrastructure Identity.
- **Goal:** Decouple runtime and setup scripts from user OAuth and enforce dedicated Service Account authentication for the Bot.
- **Atomic Responsibilities:**
  - **Bot Identity (`GoogleServiceAccountCredentials`):** Dedicated Service Account credentials for Google Chat API and Pub/Sub. Isolated from host VM permissions.
  - **`GoogleAuthClient` (Auth Service):** Generates and caches Google Cloud Service Account Bearer tokens using RSA-SHA256 JWT assertions (via `jsonwebtoken`) with the `https://www.googleapis.com/auth/chat.bot https://www.googleapis.com/auth/pubsub` scopes.
  - **`synapto-plugin-google-chat` (Plugin):**
    - Implements `ChatPlugin` (`peer_input_text_tx`, `cognitive_output_text_rx`, `cognitive_state_rx`) and `DocumentProviderPlugin` (`add_document_tx`).
    - Uses Bot Service Account Bearer tokens for Google Chat REST APIs and Pub/Sub pulls.
    - Filters self-messages using the Chat App identity (`sender.type == "BOT"` or sender matching `client_id`).
    - Cleans up existing bot reactions by matching `users/{client_id}` without calling People API.
    - Receives native Chat App Pub/Sub messages and posts messages/reactions under the official App identity.
  - **`setup_gcp_project.sh` (Deployment Automation):**
    - Provisions `org-assistant` Service Account and generates `google_org_assistant_service_account.json`.
    - Binds `roles/pubsub.subscriber` on `chat-events-sub` to `serviceAccount:org-assistant@${PROJECT_ID}.iam.gserviceaccount.com`.
    - Outputs Google Chat App configuration instructions.
  - **`load_client_config.sh` (Configuration Loader):**
    - Validates client configuration without requiring `ASSISTANT_USER_EMAIL`.

## 5. Data Models

### Google Bot Service Account Schema
```rust
#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct GoogleServiceAccountCredentials {
    pub project_id: String,
    pub private_key_id: String,
    pub private_key: String,
    pub client_email: String,
    pub client_id: String,
    #[serde(default)]
    pub auth_uri: Option<String>,
    #[serde(default)]
    pub token_uri: Option<String>,
}

#[derive(Serialize, Deserialize, Clone, Debug, Default)]
pub struct GoogleChatConfig {
    pub credentials: GoogleServiceAccountCredentials,
    #[serde(default)]
    pub spaces: Vec<String>,
    #[serde(default)]
    pub google_project_id: String,
}
```

### JSON Configuration Example (`config.json`)
```json
{
  "plugins": {
    "google_chat": {
      "GoogleChatPlugin": {
        "google_project_id": "my-gcp-project",
        "spaces": ["spaces/AAAAAAAAAAA"],
        "credentials": {
          "project_id": "my-gcp-project",
          "private_key_id": "key123456",
          "private_key": "-----BEGIN PRIVATE KEY-----\nMIIEvgIBADANBgk...\n-----END PRIVATE KEY-----\n",
          "client_email": "org-assistant@my-gcp-project.iam.gserviceaccount.com",
          "client_id": "123456789012345678901",
          "token_uri": "https://oauth2.googleapis.com/token"
        }
      }
    }
  }
}
```

### Client Inventory Configuration Example (`encrypted.clients.json`)
```json
{
  "TACR": {
    "use_case": "org-assistant",
    "project_id": "my-gcp-project",
    "vm_zone": "europe-west1-b",
    "machine_type": "e2-medium",
    "vm_name": "org-assistant",
    "support_email": "admin@example.com"
  }
}
```

## 6. Component Architecture
- **JWT Token Minting:** `GoogleAuthClient` signs a RSA-SHA256 JWT using `jsonwebtoken` with `private_key`, `client_email`, and scopes `https://www.googleapis.com/auth/chat.bot https://www.googleapis.com/auth/pubsub`. It posts the assertion to `token_uri` to obtain a short-lived Bearer token.
- **Event Ingestion (Pub/Sub):** The plugin pulls messages from `projects/{google_project_id}/subscriptions/chat-events-sub` using the Bot Service Account token. Native Google Chat App Pub/Sub event delivery is configured directly in Google Cloud Console, removing the `workspaceevents.googleapis.com` subscription maintenance loop.
- **Self-Message Filtering:** When an event is received, `read.rs` ignores messages where the sender's type is `BOT` or matches `users/{client_id}`, preventing recursive loops.
- **Document Forwarding:** If an incoming message contains file attachments or links, `read.rs` sends an `AddDocumentRequest` via `add_document_tx`.
- **State & Reaction Synchronization:** `lib.rs` listens to `cognitive_state_rx` (`Thinking`, `Idle`) and sends `ProcessingStateUpdate` to `reactions.rs`. `reactions.rs` posts emoji reactions via Google Chat API and cleans up prior bot reactions matching `users/{client_id}`.
- **Message Dispatch:** `write.rs` sends text responses directly to `/v1/{space}/messages` using the Bot Service Account Bearer token. Messages are posted under the official Chat App name and avatar.

## 7. Architectural Diagrams

### System Channel Wiring and Message Flow
```mermaid
flowchart TD
    Config[GoogleChatConfig Credentials] --> AuthClient[GoogleAuthClient]
    AuthClient -->|RSA-SHA256 JWT| GoogleOAuth[Google OAuth Token Endpoint]
    GoogleOAuth -->|Bearer Token| AuthClient

    PubSub[Google Cloud PubSub chat-events-sub] -->|Pull with Bot Token| ReadTask[read_from_chat_task]
    ReadTask -->|Filter Self BOT Messages| PeerInput[peer_input_text_tx]
    ReadTask -->|Forward Attachments| DocsChannel[add_document_tx]
    DocsChannel --> DocumentEngine[Synapto Document Pipeline]

    PeerInput --> SynaptoCore[Synapto Core Processing]
    SynaptoCore --> OutputText[cognitive_output_text_rx]
    SynaptoCore --> StateUpdate[cognitive_state_rx]

    StateUpdate --> ReactionTask[reaction_listener_task]
    ReactionTask -->|Delete/Add Reaction with Bot Token| ChatReactions[Google Chat Reactions API]

    OutputText --> WriteTask[cognitive_output_text_task]
    WriteTask -->|POST /v1/spaces/.../messages with Bot Token| GoogleChatAPI[Google Chat REST API]
```

### Identity Separation Boundary
```mermaid
flowchart LR
    subgraph Host Infrastructure
        VM[GCP Compute Instance]
        VMSA[VM Infrastructure Service Account]
        VM -.->|Provisioning & Deployment| VMSA
    end

    subgraph Bot Application Layer
        BotPlugin[synapto-plugin-google-chat]
        BotKey[google_org_assistant_service_account.json]
        BotSA[org-assistant Service Account]
        BotPlugin -->|Explicit Credential Load| BotKey
        BotKey -->|Authenticate As| BotSA
        BotSA -->|chat.bot & pubsub scopes| GoogleChat[Google Chat Spaces]
    end
```

## 8. Robustness & Pragmatic Security Trade-offs
- **Identity Isolation (Security Rule):** Bot credentials are explicitly injected via configuration and never inferred from host VM metadata. If the VM is compromised, bot credentials do not grant VM admin permissions, and VM permissions do not grant Chat App privileges.
- **Accepted Risks (MVP):**
  - **In-Memory Token Cache:** Token caching uses an in-memory `RwLock<Option<AccessTokenInfo>>` with expiry check rather than a distributed token store.
  - **Basic Rate Limit Handling:** Rate limiting on Google Chat REST API calls relies on standard HTTP 429 response retry with simple backoff.
  - **Static Pub/Sub Subscription:** Expects `chat-events-sub` and `chat-events-topic` to be provisioned by `setup_gcp_project.sh` before runtime startup.
- **Mitigation Strategy:**
  - Private keys are stored in encrypted `sops` configuration files (`encrypted.config.json`) and decrypted only during local development or on-target VM initialization.
  - Pub/Sub subscription message retention is capped at 10 minutes with auto-ack to prevent backlog buildup during assistant restarts.

## 9. Systemic Consequences
- **Codebase Audit Results:**
  - `allusio-ai/synapto-plugins/google-chat/Cargo.toml`: Add `jsonwebtoken` dependency for RSA JWT signing.
  - `allusio-ai/synapto-plugins/google-chat/src/google_auth/oauth_client.rs`: Refactored to `GoogleAuthClient` generating Service Account Bearer tokens.
  - `allusio-ai/synapto-plugins/google-chat/src/lib.rs`: Configuration struct updated from `OAuthClientCredentials` to `GoogleServiceAccountCredentials`.
  - `allusio-ai/synapto-plugins/google-chat/src/read.rs`: Removed People API call (`/v1/people/me`) and `maintain_subscriptions` loop; bot identity filtered using `client_id` and message `BOT` type.
  - `allusio-ai/synapto-plugins/google-chat/src/reactions.rs`: Removed People API lookup; reactions cleaned up and dispatched using Bot Service Account token and `client_id`.
  - `allusio-ai/org-assistant/scripts/setup_gcp_project.sh`: Refactored to create `org-assistant` service account, generate key, grant `roles/pubsub.subscriber`, and remove OAuth brand/client instructions.
  - `allusio-ai/org-assistant/scripts/load_client_config.sh`: Removed `ASSISTANT_USER_EMAIL` validation.
  - `allusio-ai/org-assistant/clients/TACR/encrypted.config.json`: Replaced `google_oauth_client_credentials` with `google_service_account_credentials`.
- **Breaking Changes:**
  - `GoogleChatConfig` schema changes from OAuth client credentials to Service Account credentials. Existing deployments must re-run `just setup-client` and encrypt the new Service Account key.
- **Documentation Updates:**
  - Update `allusio-ai/org-assistant/README.md` or setup instructions to document the Service Account bot setup process.

## 10. Definition of Done (DoD)
- [x] 1. Add `jsonwebtoken` to `synapto-plugin-google-chat/Cargo.toml` and implement RSA-SHA256 JWT Service Account token minting in `google_auth`.
- [x] 2. Remove `OAuthClientCredentials` and 3-legged OAuth endpoints from `synapto-plugin-google-chat`.
- [x] 3. Remove People API calls (`people.googleapis.com`) and Workspace Events subscription loop (`maintain_subscriptions`) from `read.rs` and `reactions.rs`.
- [x] 4. Update `GoogleChatConfig` to accept `GoogleServiceAccountCredentials`.
- [x] 5. Update `synapto-plugin-google-chat` unit and scenario tests to mock Service Account token exchange.
- [x] 6. Refactor `org-assistant/scripts/setup_gcp_project.sh` to create `org-assistant` Service Account, generate key, grant Pub/Sub permissions to this account, and remove OAuth consent/desktop client steps.
- [x] 7. Update `org-assistant/scripts/load_client_config.sh` to eliminate `ASSISTANT_USER_EMAIL`.
- [x] 8. Verify cargo clippy and unit tests pass on `google-chat` and `org-assistant`.
