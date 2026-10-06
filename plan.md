# Plan: Self-Hosted Warp Backend (Session Sharing, Drive, Firebase Auth)

## Goal

Build a minimal, self-hosted backend that enables the three cloud features the Warp
client depends on most:

1. **Firebase Authentication** — identity and token management
2. **Warp Drive** — cloud sync of notebooks, workflows, and folders
3. **Session Sharing** — real-time shared terminal sessions

All three can be wired to a custom build of the Warp client by overriding environment
variables or patching channel config.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  Warp Client                                                        │
│  WARP_SERVER_ROOT_URL=https://your-server                           │
│  WARP_WS_SERVER_URL=wss://your-server/graphql/v2                    │
│  firebase_auth_api_key=<your Firebase project key>                  │
└──────────┬─────────────────────────────────────────────────────────┘
           │
    ┌──────┴────────────────────────────────────┐
    │              Your Backend                  │
    │                                            │
    │  ┌─────────────┐  ┌────────────────────┐  │
    │  │  GraphQL API │  │  Session Sharing   │  │
    │  │  (Drive +    │  │  WebSocket Relay   │  │
    │  │   Auth ops)  │  │  (sessions.*)      │  │
    │  └──────┬───────┘  └────────────────────┘  │
    │         │                                   │
    │  ┌──────┴──────┐  ┌──────────────────────┐ │
    │  │  PostgreSQL  │  │  Redis (pub/sub for  │ │
    │  │  (objects,   │  │  session relay)      │ │
    │  │   users)     │  └──────────────────────┘ │
    │  └─────────────┘                            │
    └────────────────────────────────────────────┘
           │
    ┌──────┴────────┐
    │  Google        │
    │  Firebase      │
    │  (your project)│
    └───────────────┘
```

---

## Phase 1: Firebase Authentication

**Effort:** Low. Firebase is Google's managed service; you only need your own project.

### Steps

1. **Create a Firebase project** at [console.firebase.google.com](https://console.firebase.google.com).
   - Enable **Email/Password** and **Google** sign-in providers under Authentication.
   - Note your **Web API Key** (shown in Project Settings → General).

2. **Patch the Warp client channel config** in
   `crates/warp_core/src/channel/config.rs` → `WarpServerConfig::production()`:
   ```rust
   firebase_auth_api_key: "<your-firebase-web-api-key>".into(),
   ```

3. **Implement the `mint_custom_token` GraphQL mutation** on your server. This is
   called by the client after a browser-based login flow to exchange a Warp server
   session for a Firebase custom token. Use the **Firebase Admin SDK** (Node.js,
   Python, Go, etc.) to call `auth.createCustomToken(uid)`.

4. **Implement the `create_anonymous_user` GraphQL mutation** for users who skip
   login. Create an anonymous Firebase user via Firebase Admin SDK and return the
   custom token.

5. **Token proxy endpoints** (optional fallback):
   - `POST /proxy/token?key={api_key}` — proxies Firebase refresh token exchange
   - `POST /proxy/customToken?key={api_key}` — proxies custom token sign-in

   These are only hit when the client cannot reach Firebase directly
   (see `FirebaseToken::proxy_url` in `crates/warp_server_auth/src/credentials.rs`).

### Firebase REST API calls made by the client

The client talks to Firebase directly via:
- `POST https://identitytoolkit.googleapis.com/v1/accounts:signInWithCustomToken?key={api_key}`
- `POST https://securetoken.googleapis.com/v1/token` (refresh)

No server-side implementation is needed for these — they go directly to Google.

---

## Phase 2: Warp Drive (GraphQL API + Real-Time Subscriptions)

**Effort:** High. Drive maps to a large subset of the GraphQL schema.

### GraphQL Schema

The full schema is at `crates/warp_graphql_schema/api/schema.graphql` (~5,000 lines,
625 type/input/enum definitions). Implement only the Drive-relevant subset.

### Data Model

Store the following object types in PostgreSQL:

| Type | Table | Notes |
|---|---|---|
| `Notebook` | `notebooks` | Content stored as CRDT or JSON blob |
| `Workflow` | `workflows` | Command + description |
| `Folder` | `folders` | Hierarchical, `parent_uid` FK |
| `GenericStringObject` | `generic_string_objects` | Arbitrary key-value cloud objects |
| `User` | `users` | Firebase UID as primary key |

### Required GraphQL Operations

**Queries:**
- `getUser` — return user profile and settings
- `getWorkspacesMetadataForUser` — workspace + team membership
- `syncMerkleTree` — client-side conflict detection for Drive sync

**Mutations:**
- `createNotebook`, `updateNotebook`, `deleteObject`, `trashObject`, `untrashObject`
- `createWorkflow`, `updateWorkflow`
- `createFolder`, `updateFolder`
- `createGenericStringObject`, `updateGenericStringObject`
- `bulkCreateObjects` — batch creation on first sync
- `setObjectLinkPermissions`, `addObjectGuests`, `updateObjectGuests`, `removeObjectGuest`
- `moveObject`
- `setUserIsOnboarded`, `updateUserSettings`
- `generateApiKey`, `expireApiKey`

**Subscriptions:**
- `warpDriveUpdates` — real-time object change events pushed to the client over
  `graphql-transport-ws` WebSocket protocol (see
  `crates/graphql/src/api/subscriptions/get_warp_drive_updates.rs`)

### Authentication for GraphQL

The client sends one of these headers per request:
- `Authorization: ****** — verified with Firebase Admin SDK
- `Authorization: ApiKey <key>` — verified against `api_keys` table

### Implementation Stack (recommended)

- **Language:** TypeScript (Node.js) or Go
- **GraphQL server:** Apollo Server / graphql-yoga (Node.js) or gqlgen (Go)
- **Real-time:** `graphql-ws` library for subscription transport
- **Database:** PostgreSQL with Prisma or TypeORM (Node.js) / sqlx (Go)
- **Auth middleware:** Firebase Admin SDK token verification on every request

### Merkle Tree Sync

The client uses a Merkle tree for efficient sync (see `crates/graphql/src/api/queries/sync_merkle_tree.rs`).
For an initial implementation, skip this and return an empty/full diff — the client
will fall back to fetching all objects. Implement Merkle sync in a later phase.

---

## Phase 3: Session Sharing WebSocket Relay

**Effort:** Medium. The protocol is fully defined in this repo.

### Protocol

The client implements the full protocol in `crates/session_sharing_protocol/`.
The relay server must speak the same binary/JSON message format.

Key protocol concepts:
- **Sharer** — the terminal owner; opens the session and streams terminal events
- **Viewer** — guests who connect to observe or collaborate
- Messages: `UpstreamMessage` (client→server) and `DownstreamMessage` (server→client)
  defined in `session_sharing_protocol::sharer` and `::viewer`

### Server Responsibilities

1. **Session lifecycle:**
   - Accept `InitPayload` from sharer, create a session record, return a `SessionId`
   - Accept viewer connections with the session link/ID
   - On sharer disconnect, send `SessionEnded` to all viewers

2. **Message relay:**
   - Buffer terminal events from the sharer (scrollback, ordered events)
   - Fan out sharer events to all connected viewers
   - Relay viewer input/control actions back to the sharer
   - Track participant presence and send `ParticipantPresenceUpdate`

3. **Authentication:**
   - Verify Firebase ID token in the WebSocket init payload
   - Enforce `LinkAccessLevel` and `TeamAccessLevel` permissions

4. **Reconnection:**
   - Issue a `ReconnectToken` to sharers so they can resume on disconnect without
     losing the session

### Session Sharing Server URL

Configure the client via:
```bash
# In crates/warp_core/src/channel/config.rs WarpServerConfig::production():
session_sharing_server_url: Some("wss://your-sessions-server".into()),
```
Or at runtime:
```bash
SESSION_SHARING_SERVER_URL=wss://your-sessions-server ./script/run
```

### Implementation Stack (recommended)

- **Language:** Go or Rust (tokio + axum + tokio-tungstenite)
- **State:** In-process for single-node; Redis pub/sub for multi-node
- **Scalability:** Each session is a small broadcast channel; a single node handles
  thousands of concurrent sessions

---

## Phase 4: Client Build Configuration

Modify `crates/warp_core/src/channel/config.rs` to point the client at your services:

```rust
impl WarpServerConfig {
    pub fn production() -> Self {
        Self {
            server_root_url: "https://your-server.example.com".into(),
            rtc_server_url: "wss://your-server.example.com/graphql/v2".into(),
            session_sharing_server_url: Some("wss://your-sessions.example.com".into()),
            firebase_auth_api_key: "<your-firebase-web-api-key>".into(),
            iap_config: None,
        }
    }
}
```

Or for local development without patching the binary:
```bash
WITH_LOCAL_SERVER=1 \
  SERVER_ROOT_URL=https://your-server.example.com \
  WS_SERVER_URL=wss://your-server.example.com/graphql/v2 \
  ./script/run
```

---

## Phase 5: Deployment

### Docker Compose (single-node)

```yaml
services:
  graphql-api:
    build: ./services/graphql-api
    environment:
      DATABASE_URL: postgres://...
      FIREBASE_PROJECT_ID: your-project-id
    ports:
      - "8080:8080"

  session-relay:
    build: ./services/session-relay
    environment:
      REDIS_URL: redis://redis:6379
      FIREBASE_PROJECT_ID: your-project-id
    ports:
      - "8081:8081"

  postgres:
    image: postgres:16
    volumes:
      - pg_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
```

### Infrastructure checklist

- [ ] TLS (required — client enforces HTTPS/WSS in production builds)
- [ ] Firebase service account key available to server processes
- [ ] PostgreSQL with migrations applied
- [ ] Redis for session relay fan-out (optional for single-node)
- [ ] Reverse proxy (nginx / Caddy) for routing `/graphql` vs `/sessions`

---

## Implementation Order

1. **Firebase Auth** — unblock login immediately; low risk, external service
2. **GraphQL API stub** — return minimal `getUser` so the client can start up
3. **Drive mutations** — implement CRUD for notebooks/workflows/folders
4. **Drive subscriptions** — add real-time push via `graphql-ws`
5. **Session sharing relay** — implement WebSocket relay from protocol spec
6. **Merkle tree sync** — optimize Drive sync for large object counts

---

## Key Files for Reference

| File | Purpose |
|---|---|
| `crates/warp_core/src/channel/config.rs` | Server URL and Firebase key config |
| `crates/warp_graphql_schema/api/schema.graphql` | Full GraphQL schema |
| `crates/graphql/src/api/mutations/` | All GraphQL mutations the client uses |
| `crates/graphql/src/api/queries/` | All GraphQL queries |
| `crates/graphql/src/api/subscriptions/get_warp_drive_updates.rs` | Drive real-time subscription |
| `crates/warp_server_auth/src/credentials.rs` | Firebase token exchange flow |
| `crates/firebase/src/lib.rs` | Firebase REST API response types |
| `crates/session_sharing_protocol/` | Full session sharing message protocol |
| `app/src/terminal/shared_session/sharer/network.rs` | Sharer WebSocket client |
| `app/src/terminal/shared_session/viewer/network.rs` | Viewer WebSocket client |
