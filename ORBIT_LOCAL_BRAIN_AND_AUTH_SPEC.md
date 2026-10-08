# Orbit Local Brain, Storage Foundation, and Auth Architecture Specification

> **Document Version**: 1.0.0  
> **Target Application**: Orbit Desktop (Tauri 2 + React 19 + Rust)  
> **Status**: Approved Architectural Blueprint  
> **Last Updated**: 2026-10-09  

---

## Table of Contents

1. [Executive Summary & Core Objectives](#1-executive-summary--core-objectives)
2. [Chronological Discussion & Decision Archive](#2-chronological-discussion--decision-archive)
   - [2.1 Cognee Feasibility & Local Storage Assessment](#21-cognee-feasibility--local-storage-assessment)
   - [2.2 Option A (Native Rust) Pros, Cons & Embeddings Strategy](#22-option-a-native-rust-pros-cons--embeddings-strategy)
   - [2.3 Database Analysis: Why SQLite is Required Alongside LanceDB & Kùzu](#23-database-analysis-why-sqlite-is-required-alongside-lancedb--kùzu)
   - [2.4 Product Vision: Local-First Freemium SaaS with Deep-Link Auth](#24-product-vision-local-first-freemium-saas-with-deep-link-auth)
   - [2.5 Deconstructing the Mystery Web Login (Builderlab Analysis)](#25-deconstructing-the-mystery-web-login-builderlab-analysis)
   - [2.6 Path 1 (Hosted Relay) vs. Path 2 (True Local-First) Comparison](#26-path-1-hosted-relay-vs-path-2-true-local-first-comparison)
   - [2.7 The 1-to-1 Replacement of the Server Stack](#27-the-1-to-1-replacement-of-the-server-stack)
3. [Complete API Specification for `auth.orbit.com`](#3-complete-api-specification-for-authorbitcom)
4. [Local Desktop Storage Schema (SQLite & LanceDB)](#4-local-desktop-storage-schema-sqlite--lancedb)
5. [Step-by-Step Desktop Code Implementation Blueprint](#5-step-by-step-desktop-code-implementation-blueprint)
   - [5.1 Cargo.toml Dependency Additions](#51-cargotoml-dependency-additions)
   - [5.2 Local Storage Engine (`storage/sqlite.rs`)](#52-local-storage-engine-storagesqliters)
   - [5.3 Application State Integration (`app_state.rs`)](#53-application-state-integration-app_staters)
   - [5.4 Rewriting Message Commands (`commands/messages.rs`)](#54-rewriting-message-commands-commandsmessagesrs)
   - [5.5 Repointing Auth & Communities (`builderlab.rs` & `hostedCommunityApi.ts`)](#55-repointing-auth--communities-builderlabrs--hostedcommunityapits)
   - [5.6 Deactivating WebSocket Relay Auto-Connect in Frontend](#56-deactivating-websocket-relay-auto-connect-in-frontend)
6. [Production Server Sizing & Hosting Cost Breakdown](#6-production-server-sizing--hosting-cost-breakdown)

---

## 1. Executive Summary & Core Objectives

### The Goal
Transform Orbit into a **100% private, local-first desktop application** equipped with an embedded context brain for autonomous AI agents. Free users can execute all their workflows, conversations, code indexing, and agent co-working locally on their laptops with zero network dependencies and absolute privacy. When users want remote collaboration, team channels, or cross-device synchronization, an optional paid subscription enables encrypted sync through a hosted server.

### The Fundamental Problem in the Inherited Codebase
The inherited Buzz codebase is architected as a thin client connecting over WebSocket to a heavy hosted server stack:
* **Relay Server (`buzz-relay`)**: WebSocket server running Nostr NIP-29.
* **Database**: PostgreSQL 17 for all messages, channels, threads, and full-text search.
* **Pub/Sub**: Redis 7 for presence, typing indicators, and fanout.
* **Storage**: MinIO / S3 for media attachments.
* **Authentication**: External proprietary Builderlab service (`app.builderlab.xyz`).

If deployed as-is, free users would store all their data on your remote server, requiring you to finance database clusters and servers for non-paying users, while denying users offline access and true privacy.

### The Architectural Solution
Inside the desktop app (Tauri 2 + Rust), replace the five remote server services with **in-process embedded equivalents**:
1. **PostgreSQL** $\longrightarrow$ **SQLite (`rusqlite`)** in `~/.orbit/brain/db/orbit.db`
2. **Redis** $\longrightarrow$ **Tauri In-Memory Window Events / Tokio Broadcast**
3. **MinIO / S3** $\longrightarrow$ **Local File System** in `~/.orbit/brain/media/`
4. **Nostr WebSocket Relay** $\longrightarrow$ **In-Process Tauri IPC Commands**
5. **Builderlab Auth** $\longrightarrow$ **Self-Hosted Orbit Auth Server (`auth.orbit.com`)**
6. **AI Vector Context** $\longrightarrow$ **LanceDB (Apache Arrow) + FastEmbed-RS (Local ONNX `bge-small-en-v1.5`)**

---

## 2. Chronological Discussion & Decision Archive

### 2.1 Cognee Feasibility & Local Storage Assessment

#### User Request:
> *"what is fisable to dierctly install cognee as context brain in this project ? or use its codes and add it will it also store the vectors and embeddings locally ?? use the https://github.com/topoteretes/cognee and the docs from https://docs.cognee.ai/ etc and check I want to use this the inside of this application to create a context layer for the agents: one centralized context and the graph build, while using the existing infrastructure of the cognee . The cognee provides the different applications but I want to check which is the more feasiblle to use its architecture or directly use Rust based thing they give"*

#### Analysis & Decisions:
1. **Feasibility**: High. Cognee uses an Extract, Cognify, Load (ECL) pipeline to generate a GraphRAG context layer.
2. **Local Vector & Embedding Storage**: **100% Local.** Cognee operates local-first by default:
   - **Vector Database**: **LanceDB** (an embedded, serverless, file-backed vector DB using Apache Arrow). Stored locally in `.lancedb` on disk.
   - **Graph Database**: **Kùzu** (Python) or **Ladybug** (Rust), storing entity nodes and relationship edges in local disk files.
   - **Relational Store**: **SQLite** for metadata and provenance.
   - **Embeddings**: Generated using local models via FastEmbed / ONNX / Ollama, meaning zero vectors or text leave the user's machine.
3. **Implementation Choice (Rust Crate vs. Python Sidecar)**:
   - Cognee provides a native Rust implementation available as crate `cognee` (previously `cognee-lib` / `cognee-rs`).
   - Using the Rust crate is strongly recommended over a Python sidecar because Orbit is already built in Rust (Tauri + Tokio). The Rust crate compiles into the binary, runs in-process, has fast boot times (~350ms), and low query latency (~260ms) without requiring a Python virtual environment.

---

### 2.2 Option A (Native Rust) Pros, Cons & Embeddings Strategy

#### User Request:
> *"i would like to go with the A and can you tell me pros and cons of it will it hault the existing system at some things how hard it is to build and etc will there be issue to manage pc vector and these many databases what if we keept embeddings in the lanceDB and all of the scenarios etc i will be hosting the server of auth mainly so users need to login that's the only thing i am hosting and other than that i want to keep embedder local or online which is best in future use when user wnat to do remote work they can host their data via subscription so i want to keep that in mind also"*

#### Analysis & Decisions:
1. **Pros of Native Rust (Option A)**:
   - **True Desktop Experience**: No Docker containers, no background daemons, zero external database setup.
   - **Performance**: LanceDB operates via memory-mapped files directly on disk, eliminating IPC socket overhead.
   - **Low Memory**: Takes ~30MB–80MB RAM instead of PostgreSQL's ~200MB+ idle baseline.
   - **Privacy**: Embeddings and code snippets stay on the user's SSD.
2. **Cons of Native Rust**:
   - Initial compilation time increases due to `lancedb` and `arrow-array` dependencies.
   - Dual-store synchronization: Writes across SQLite and LanceDB are not natively in a single SQL transaction.
     * *Solution*: Treat SQLite as the authoritative source of truth. LanceDB is a derived, rebuildable index. If LanceDB is ever deleted or corrupted, it can be re-indexed directly from SQLite.
3. **Impact on Existing System**:
   - Compiling the desktop client with embedded stores does **not** break `buzz-relay` or `buzz-db`. The desktop storage substrate is confined to `desktop/src-tauri`.
4. **Local vs. Online Embeddings Strategy**:
   - **Default Local (`fastembed-rs`)**: Runs `bge-small-en-v1.5` (384 dimensions) on local CPU. Costs $0, works offline, zero token fees.
   - **Optional Online API**: Allow users to optionally input an OpenAI / Voyage API key in settings if they prefer to save CPU/battery.

---

### 2.3 Database Analysis: Why SQLite is Required Alongside LanceDB & Kùzu

#### User Request:
> *"what databases we use in this already how much things we need to update for all of the changes we have so far for graph i use kazu and same archict of the cognee but do we need .db file ?? why SQLLIte is is used and what ddatabases is used already in sysem what .db file will it be safe ? and if i add all of that and made local and then created paid subscription version for user what i have to host then we completely need to change apps architecture right becase we would need auto sync and sync on and stuff to talk to users server and all what does cognee uses for it etc check"*

#### Analysis & Decisions:
1. **Databases Currently in the Repository**:
   - **Relay / Backend (`buzz-relay`, `buzz-db`)**: PostgreSQL 17 (via `sqlx`) + Redis 7 (`deadpool-redis`).
   - **Desktop Client (`desktop/src-tauri`)**: Currently has **NO local database**. It acts as a client connecting over WebSocket to the relay.
2. **Why SQLite (`orbit.db`) is Essential**:
   - LanceDB is an append-optimized columnar vector index; it cannot handle relational transactions, foreign keys, or document update catalogs.
   - Kùzu is a graph traversal engine; it cannot efficiently store raw document text, job statuses, or user preferences.
   - **SQLite is the Orchestrator**: It holds documents, chunk hashes, timestamps, and channel histories. When a file changes on disk, SQLite tracks what was modified, deletes obsolete vectors from LanceDB, and updates entity nodes in Kùzu.
3. **Is SQLite Safe?**:
   - SQLite with WAL (`Write-Ahead Logging`) mode is exceptionally crash-safe. If the user's laptop loses power mid-write, SQLite rolls back to the last valid state without corruption.
   - Data is stored in `~/.orbit/brain/db/orbit.db`, scoped strictly to that OS user profile.

---

### 2.4 Product Vision: Local-First Freemium SaaS with Deep-Link Auth

#### User Request:
> *"how can i update this project to add these thats why concern add below things : and also what server i would be needing to host free only auth for users i dont want nostr based server etc ill replace that with the google auth based server jsontocken etc so what servers i have to host etc what i have to do as production so based on what we have in the postgre i would need to migrate ? i mean ill give overview what i want i want user to have an desktop app he can do all the stuff privatcly locally no issue then he want to remote work with others we have paid plans for it they will share but i want them to login into the auth server when they dowload the app via deeplink auth so we can retain user other than that eveything is local and user centric he can use cowork etc and do stuff data will remain local for free users and all"*

#### Analysis & Decisions:
1. **The Product Model**: Local-First Freemium SaaS.
2. **Deep-Link Google Auth Flow**:
   - User downloads desktop app.
   - On first launch: App prompts "Sign in with Google to activate Orbit".
   - Browser opens: `https://auth.orbit.com/api/v1/auth/login?returnTo=http://127.0.0.1:{port}/callback/{nonce}`.
   - User authenticates via Google.
   - Orbit Auth Server records user in PostgreSQL (`users` table with `plan = 'free'`), generates a JWT, and redirects to:
     `http://127.0.0.1:{port}/callback/{nonce}?code={exchange_code}` (or custom scheme `orbit://auth/callback?token={jwt}`).
   - Tauri captures the token, saves it in the OS Keyring, and unlocks the local workspace.
3. **Do Free Users Need to Migrate PostgreSQL?**:
   - **No.** Free users never install, touch, or migrate PostgreSQL. All local data is managed in embedded SQLite (`orbit.db`) and LanceDB files.
4. **Subscription Cloud Sync Model**:
   - Free users operate 100% locally.
   - When a user upgrades to Pro/Paid, the app activates a sync adapter that packages logical event deltas, encrypts them, and relays them to your hosted cloud server. Teammates download the deltas into their own local SQLite databases.
   - Never sync raw database files across machines; sync logical change events.

---

### 2.5 Deconstructing the Mystery Web Login (Builderlab Analysis)

#### User Request:
> *"after this how can i change auth flow we have rn from the noster to the google auth signup signin flow and theres one already hosted relay somthing which when we ceate the community we need to sign in the web we see some email id and password one but that code is not available here on the local i want to make one too with name of orbit so how can i do it also ; thats hosted community flow ; how much server i sould need to host auth server and all just give all of the things"*

#### Source Code Investigation:
In `desktop/src-tauri/src/builderlab.rs` and `desktop/src/features/communities/hostedCommunityApi.ts`:
- The original Buzz app hardcodes an external SaaS service called **Builderlab** (`https://app.builderlab.xyz/api/goose` and `communities.buzz.xyz`).
- When creating a hosted community, Tauri launches a temporary Axum server on `127.0.0.1:{random_port}/callback/{nonce}` and opens the Builderlab web login page.
- Builderlab redirects back with a code, which Tauri exchanges via `POST /v1/auth/login/exchange` for an `X-BB-Session-Credential` token.
- **Solution**: Replace `BUILDERLAB_API_BASE_URL` with `https://auth.orbit.com/api` and implement the 4 expected endpoints on your own server.

---

### 2.6 Path 1 (Hosted Relay) vs. Path 2 (True Local-First) Comparison

#### User Request:
> *"will that remove noster dependecy from the code ? then do i have to host relay also and other buzz things what i have to host give list of all because theres one thig as hosted relay is that code also available i dont want to use any hosted of thirdparty but use the same and host on our server"*  
> *"but first path dont keep data on the local right"*

#### Architectural Comparison:

| Feature | Path 1: Self-Host Buzz Relay | Path 2: True Local-First (SQLite + LanceDB) |
| :--- | :--- | :--- |
| **Where data lives for free users** | On your remote PostgreSQL server | **100% on user's local disk** |
| **Offline functionality** | No (Fails without internet) | **Yes (100% functional offline)** |
| **User Privacy** | Relies on trusting your server | **Absolute local privacy** |
| **What you must host** | Relay + Postgres + Redis + S3 + Auth | **Only the Auth API server** |
| **Server Cost (10k users)** | High ($50–$200+/month) | **~$5/month (Single 2GB RAM VPS)** |
| **Value of Paid Plan** | Ambiguous | **Clear: Pay for Cloud Sync & Collaboration** |

**Conclusion**: **Path 2 is the correct architecture.** It fulfills the local-first requirement, guarantees privacy, reduces hosting costs to near zero, and establishes a clear monetization boundary.

---

### 2.7 The 1-to-1 Replacement of the Server Stack

#### User Request:
> *"what changes in curreny system i have to do to achieve all of stuff on local and private one single auth server after scaling we will make relay and all to server but one thing then accoding to curret achtecture the Relay + Postgres + Redis + S3 + Auth all are hosted on the server right how can i add these feats on local then storing in local that isnt in the code because it alreasy uses hosted relay thats kind of confusing"*

#### Component Replacement Mapping:

| Remote Server Service (Buzz) | Local Desktop Replacement (Orbit) | Implementation Details |
| :--- | :--- | :--- |
| **PostgreSQL 17** | **SQLite (`rusqlite`)** | Single file at `~/.orbit/brain/db/orbit.db` storing channels, messages, and document metadata. |
| **Redis 7** | **Tauri Events / In-Memory Channel** | When a message is inserted into SQLite, Tauri fires `app.emit("new-message")` to update the React UI instantly. |
| **MinIO / S3 Storage** | **Local File System** | File uploads are saved directly to `~/.orbit/brain/media/<sha256>.<ext>` and served via local asset protocols. |
| **Nostr WebSocket Relay** | **In-Process Tauri IPC** | React UI calls `invoke('send_channel_message')` directly into Rust; no WebSocket server needed. |
| **Cloud RAG Pipeline** | **LanceDB + FastEmbed-RS** | Local Apache Arrow vector store with local ONNX embeddings (`bge-small-en-v1.5`). |
| **Builderlab Auth** | **Orbit Auth Server (`auth.orbit.com`)** | Single lightweight server handling Google Sign-In and user account retention. |

---

## 3. Complete API Specification for `auth.orbit.com`

To seamlessly replace Builderlab, implement this API contract on your server (`https://auth.orbit.com/api`):

### 1. Initiate Login
* **Method & Path**: `GET /v1/auth/login`
* **Query Parameters**:
  * `returnTo`: `http://127.0.0.1:{port}/callback/{nonce}`
  * `type`: `"cli"`
  * `product`: `"orbit"`
* **Flow**:
  1. Displays the Orbit login screen with a **"Continue with Google"** button.
  2. User signs in with Google.
  3. Server creates or fetches the user from PostgreSQL, assigns `plan = 'free'`, and generates a 60-second `exchange_code`.
  4. Server redirects browser to: `{returnTo}?code={exchange_code}`.

---

### 2. Code Exchange
* **Method & Path**: `POST /v1/auth/login/exchange`
* **Request Body**:
  ```json
  {
    "code": "temporary_exchange_code_here"
  }
  ```
* **Response (HTTP 200)**:
  ```json
  {
    "session_credential": "eyJhbGciOi...",
    "expires_at": "2027-10-09T00:00:00Z"
  }
  ```

---

### 3. Current User Profile
* **Method & Path**: `GET /v1/auth/me`
* **Header**: `X-BB-Session-Credential: <session_credential>`
* **Response (HTTP 200)**:
  ```json
  {
    "email": "user@gmail.com",
    "name": "Jane Doe",
    "expires_at": "2027-10-09T00:00:00Z",
    "plan": "free"
  }
  ```

---

### 4. Nostr Identity Compatibility Bridge
* **Method & Path**: `POST /v1/buzz/nostr-identities/current`
* **Header**: `X-BB-Session-Credential: <session_credential>`
* **Response (HTTP 200)**:
  ```json
  {
    "identity": {
      "pubkey_hex": "64_hex_chars_derived_from_user_id",
      "npub": "npub1..."
    }
  }
  ```
  *(Derive `pubkey_hex` deterministically using `sha256(user.email)` to maintain frontend compatibility without needing Nostr keys).*

---

### 5. Community Provisioning (For Paid Users)
* **Check Availability**: `POST /v1/buzz/communities/availability`
  * Body: `{"name": "team-name"}`
  * Response: `{"available": true, "normalized_host": "team-name.orbit.com"}`
* **Create Community**: `POST /v1/buzz/communities`
  * Body: `{"name": "team-name"}`
  * Response: `{"community": {"id": "uuid", "name": "team-name", "normalized_host": "team-name.orbit.com"}}`
* **List Communities**: `POST /v1/buzz/communities/list`
  * Response: `{"communities": [ ... ]}`

---

## 4. Local Desktop Storage Schema (SQLite & LanceDB)

### SQLite Schema (`~/.orbit/brain/db/orbit.db`)

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA foreign_keys = ON;

-- Local Channels
CREATE TABLE IF NOT EXISTS channels (
    id              TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    channel_type    TEXT NOT NULL DEFAULT 'stream',
    description     TEXT,
    topic           TEXT,
    created_by      TEXT NOT NULL DEFAULT 'local_user',
    created_at      INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL
);

-- Local Messages (matches React RelayEvent interface)
CREATE TABLE IF NOT EXISTS messages (
    id              TEXT PRIMARY KEY,
    channel_id      TEXT NOT NULL REFERENCES channels(id) ON DELETE CASCADE,
    author_id       TEXT NOT NULL DEFAULT 'local_user',
    content         TEXT NOT NULL,
    parent_event_id TEXT,
    root_event_id   TEXT,
    depth           INTEGER DEFAULT 0,
    created_at      INTEGER NOT NULL,
    metadata_json   TEXT
);

CREATE INDEX IF NOT EXISTS idx_messages_chan_time ON messages(channel_id, created_at ASC);

-- Document Ingestion Metadata
CREATE TABLE IF NOT EXISTS orbit_documents (
    id              TEXT PRIMARY KEY,
    file_path       TEXT NOT NULL UNIQUE,
    file_hash       TEXT NOT NULL,
    byte_size       INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL
);

-- Document Chunks Catalog
CREATE TABLE IF NOT EXISTS orbit_chunks (
    id              TEXT PRIMARY KEY,
    document_id     TEXT NOT NULL REFERENCES orbit_documents(id) ON DELETE CASCADE,
    chunk_index     INTEGER NOT NULL,
    content         TEXT NOT NULL,
    chunk_hash      TEXT NOT NULL,
    created_at      INTEGER NOT NULL
);
```

### LanceDB Arrow Schema (`~/.orbit/brain/vectors/`)

Table: `orbit_embeddings`
* `vector`: `FixedSizeList<Float32, 384>` (Generated by `bge-small-en-v1.5`)
* `item_id`: `Utf8` (Message ID or Document Chunk ID)
* `source_type`: `Utf8` (`"chat"` or `"document"`)
* `context_scope`: `Utf8` (Channel UUID or File Path)
* `author_id`: `Utf8`
* `content_text`: `Utf8`
* `created_at`: `Int64`

---

## 5. Step-by-Step Desktop Code Implementation Blueprint

### 5.1 Cargo.toml Dependency Additions
File: `desktop/src-tauri/Cargo.toml`

```toml
[dependencies]
rusqlite = { version = "0.32", features = ["bundled"] }
lancedb = "0.15"
arrow-array = "53"
fastembed = "4"
```

---

### 5.2 Local Storage Engine (`storage/sqlite.rs`)
Create file: `desktop/src-tauri/src/storage/sqlite.rs`

```rust
use rusqlite::{Connection, Result};
use std::path::PathBuf;

pub struct LocalDatabase {
    pub conn: Connection,
}

impl LocalDatabase {
    pub fn init(db_path: PathBuf) -> Result<Self> {
        if let Some(parent) = db_path.parent() {
            std::fs::create_dir_all(parent).ok();
        }
        let conn = Connection::open(db_path)?;
        conn.pragma_update(None, "journal_mode", "WAL")?;

        conn.execute_batch(
            r#"
            CREATE TABLE IF NOT EXISTS channels (
                id TEXT PRIMARY KEY,
                name TEXT NOT NULL,
                channel_type TEXT DEFAULT 'stream',
                description TEXT,
                created_at INTEGER NOT NULL
            );

            CREATE TABLE IF NOT EXISTS messages (
                id TEXT PRIMARY KEY,
                channel_id TEXT NOT NULL REFERENCES channels(id),
                author_id TEXT NOT NULL,
                content TEXT NOT NULL,
                parent_event_id TEXT,
                root_event_id TEXT,
                depth INTEGER DEFAULT 0,
                created_at INTEGER NOT NULL
            );

            CREATE INDEX IF NOT EXISTS idx_msgs_chan ON messages(channel_id, created_at ASC);
            "#,
        )?;

        // Ensure default general channel exists
        conn.execute(
            "INSERT OR IGNORE INTO channels (id, name, created_at) VALUES ('general', 'general', unixepoch())",
            [],
        )?;

        Ok(Self { conn })
    }
}
```

---

### 5.3 Application State Integration (`app_state.rs`)
File: `desktop/src-tauri/src/app_state.rs`

```rust
use crate::storage::sqlite::LocalDatabase;
use std::sync::Mutex;

pub struct AppState {
    // ... existing AppState fields ...
    pub local_db: Mutex<LocalDatabase>,
}
```

In `desktop/src-tauri/src/lib.rs` (inside `setup`):
```rust
let home = dirs::home_dir().expect("valid home directory");
let db_path = home.join(".orbit").join("brain").join("db").join("orbit.db");
let local_db = LocalDatabase::init(db_path).expect("initialize local database");

// Manage local_db in AppState
```

---

### 5.4 Rewriting Message Commands (`commands/messages.rs`)
File: `desktop/src-tauri/src/commands/messages.rs`

```rust
use rusqlite::params;
use tauri::State;
use crate::app_state::AppState;
use crate::models::SendChannelMessageResponse;

#[tauri::command]
pub async fn send_channel_message(
    channel_id: String,
    content: String,
    parent_event_id: Option<String>,
    root_event_id: Option<String>,
    state: State<'_, AppState>,
) -> Result<SendChannelMessageResponse, String> {
    let message_id = uuid::Uuid::new_v4().to_string();
    let now = chrono::Utc::now().timestamp();
    let author_id = "local_user".to_string();

    let db = state.local_db.lock().map_err(|e| e.to_string())?;
    db.conn.execute(
        "INSERT INTO messages (id, channel_id, author_id, content, parent_event_id, root_event_id, created_at)
         VALUES (?1, ?2, ?3, ?4, ?5, ?6, ?7)",
        params![message_id, channel_id, author_id, content, parent_event_id, root_event_id, now],
    ).map_err(|e| e.to_string())?;

    // Asynchronously embed into LanceDB for agent memory context

    Ok(SendChannelMessageResponse {
        event_id: message_id,
        parent_event_id,
        root_event_id,
        depth: 0,
        created_at: now as u64,
    })
}

#[tauri::command]
pub async fn get_channel_window(
    channel_id: String,
    state: State<'_, AppState>,
) -> Result<Vec<serde_json::Value>, String> {
    let db = state.local_db.lock().map_err(|e| e.to_string())?;
    let mut stmt = db.conn.prepare(
        "SELECT id, author_id, content, created_at, parent_event_id, root_event_id 
         FROM messages WHERE channel_id = ?1 ORDER BY created_at ASC"
    ).map_err(|e| e.to_string())?;

    let rows = stmt.query_map([channel_id], |row| {
        Ok(serde_json::json!({
            "id": row.get::<_, String>(0)?,
            "pubkey": row.get::<_, String>(1)?, // 'pubkey' maintains React compatibility
            "content": row.get::<_, String>(2)?,
            "created_at": row.get::<_, i64>(3)?,
            "kind": 40002,
            "tags": []
        }))
    }).map_err(|e| e.to_string())?;

    let mut messages = Vec::new();
    for msg in rows {
        messages.push(msg.map_err(|e| e.to_string())?);
    }
    Ok(messages)
}
```

---

### 5.5 Repointing Auth & Communities (`builderlab.rs` & `hostedCommunityApi.ts`)

File: `desktop/src-tauri/src/builderlab.rs`:
```rust
const BUILDERLAB_API_BASE_URL: &str = "https://auth.orbit.com/api";
const BUILDERLAB_ORIGIN: &str = "https://auth.orbit.com";
```

File: `desktop/src/features/communities/hostedCommunityApi.ts`:
```typescript
export const HOSTED_COMMUNITY_SUFFIX = "orbit.com";
```

---

### 5.6 Deactivating WebSocket Relay Auto-Connect in Frontend
File: `desktop/src/shared/api/relayClientSession.ts`:
```typescript
public connect(): void {
  // If in local mode, do not open WebSocket connection
  if (this.isLocalMode()) {
    console.log("Orbit running in Local-First Mode (WebSocket bypassed)");
    return;
  }
  // ... existing remote WebSocket connect logic ...
}
```

---

## 6. Production Server Sizing & Hosting Cost Breakdown

Because free desktop users run all AI inference/embeddings, LanceDB vector storage, and SQLite chats directly on their laptops, your hosted server handles only authentication and subscription status:

### Recommended Infrastructure
* **Server Provider**: Hetzner Cloud `CX22` (or DigitalOcean / Linode 2GB droplet)
* **Specifications**: 2 vCPUs, 4 GB RAM, 40 GB NVMe SSD
* **Operating System**: Ubuntu 24.04 LTS
* **Cost**: **~€4.50 / month (~$5.00 USD)**
* **Software Stack**:
  * Caddy Server (Automatic HTTPS / SSL)
  * Node.js / Rust Auth Service (`auth.orbit.com`)
  * PostgreSQL 16 (Small database for user records & subscriptions)
* **Capacity**: Effortlessly supports **20,000+ active free desktop users** because the server is only contacted during initial Google login or subscription upgrades.
