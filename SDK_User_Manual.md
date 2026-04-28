
# Real-time Audio/Video Interactive SDK User Manual (v1.0.0)

This manual corresponds to the npm package **`@sanseng/livekit-ws-sdk` version 1.0.0**. The SDK is built upon **LiveKit Client** and encapsulates digital human A/V downlink, microphone/camera uplink, session text, and an HTTP control plane (for fetching connection configurations in Auth mode).

---

## 1. Overview

The SDK provides a unified entry point via **`createClient` → `SDKClient`**, responsible for:

- Audio/Video track subscription, routing, local capture, and publishing.
- Encoding, decoding, and distribution of session-side text and protocol messages.
- Connection lifecycle management and snapshots (for easier reconnection and troubleshooting).
- Optional green screen processing and performance metric reporting.

Users can complete integration via constructor parameters, connection APIs, and event subscriptions without needing to understand internal modules (e.g., `transport` implementation details).

---

## 2. Initialization

The SDK distinguishes between two mutually exclusive modes via `ClientOptions.connectConfig`: **Direct** and **Auth**. These modes differ in configuration sources, the availability of `setAuthToken` / `updateConnectionConfig`, and `reconnect` refresh behavior.

### 2.1 Direct Mode

**Description** The caller directly provides the required LiveKit `sfuUrl` and `clientToken` in the constructor. The SDK does not pull room configurations via HTTP.

**Prerequisites**
- Must provide a non-empty `sfuUrl` (LiveKit WebSocket URL) and `clientToken` (Room Access Token).
- If the room or token changes during runtime, the new configuration must be injected via `updateConnectionConfig` before the next reconnection.

**Initialization**
```ts
import { createClient } from '@sanseng/livekit-ws-sdk';

const client = createClient({
  connectConfig: {
    type: 'direct',
    config: {
      sfuUrl: 'wss://your-livekit-host',
      clientToken: 'your-room-token',
    },
  },
  video: {
    containerElement: document.getElementById('avatar')!,
  },
});
```

**Behavioral Notes**
- **`preConnect()` / `connect()`**: Resolves the `livekitUrl` and `token` from the current Direct config to establish the LiveKit connection.
- **`reconnect()`**: After disconnecting, it calls `refreshConfig()`. In Direct mode, it re-reads the config from the manager (including updates from `updateConnectionConfig`) without triggering HTTP calls.
- **Token Expiry**: The SDK does not automatically refresh server-issued tokens. Users must obtain a new token, call `updateConnectionConfig`, and then trigger `reconnect()`.

### 2.2 Auth Mode

**Description** The caller provides an `avatarId` (and optionally `authToken`, `avatarVoice`). The SDK pulls the `ConnectionConfig` via `HttpController`, merging server-returned `videoOptions` (like green screen settings) into the context.

**Prerequisites**
- Must provide a non-empty `avatarId`.
- A valid auth token must be available before `connect()`: either via `connectConfig.config.authToken` or by calling `client.setAuthToken(token)` after instantiation.
- A functional HTTP service (default or custom `http.baseURL`) must be provided.

**Initialization**
```ts
const client = createClient({
  connectConfig: {
    type: 'auth',
    config: { avatarId: 'your-avatar-id' },
  },
  http: { baseURL: 'https://your-api.example.com/...' },
  video: { containerElement: document.getElementById('avatar')! },
});

client.setAuthToken('your-business-jwt');
```

**Behavioral Notes**
- **`reconnect()`**: Calls `refreshConfig()`, which **re-triggers the HTTP request** to fetch the latest room parameters.
- **`setAuthToken`**: Updates the context token; if the `HttpController` is already active, it synchronizes the authentication logic immediately.

---

### 2.3 Mode Comparison

| Feature | Direct Mode | Auth Mode |
| :--- | :--- | :--- |
| **Config Source** | Constructor `sfuUrl`, `clientToken` | HTTP API Response |
| **Business HTTP Required** | Optional | **Yes** (to fetch room config) |
| **Required Fields** | `sfuUrl` + `clientToken` | `avatarId` + `authToken` |
| **`reconnect()` Refresh** | Reads local/updated Direct config | Re-fetches via HTTP |

---

## 3. Quick Start

### 3.1 Installation
```bash
npm install @sanseng/livekit-ws-sdk
```

### 3.2 Basic Flow (Auth Example)
```ts
import { createClient } from '@sanseng/livekit-ws-sdk';

const client = createClient({
  connectConfig: { type: 'auth', config: { avatarId: 'demo-avatar' } },
  http: { baseURL: 'https://your-api.example.com' },
  video: { containerElement: document.getElementById('avatar')! },
});

client.setAuthToken('TOKEN_FROM_BACKEND');

client.events.on('sdk:connected', ({ all }) => {
  if (all) console.log('SDK fully connected and ready');
});

await client.connect();

// Trigger microphone after user gesture for browser policy compliance
document.getElementById('start-btn').onclick = () => client.startAudioCapture();
```

---

## 4. Core Concepts

### 4.1 questionId
The `questionId` is the correlation key for a **complete Q&A turn** (User Query → Server Processing → Streaming/Full Answer).
* **Convention**: Usually a short string (4 characters). It is unique within a session but **not** suitable as a long-term database primary key.
* **Best Practice**: Clear local buffers (UI states, text concatenation) associated with a `questionId` after receiving the `conversation:answer:completed` event.

### 4.2 Control Plane vs. Media Plane
* **Media Plane**: Handled by `LiveKitService`. Covers room connections, track sub/pub, and Data Channel messages (Session text/protocol).
* **Control Plane (Auth)**: Handled by `HttpController`. Responsible for fetching tokens and `videoOptions` (like green screen toggles).

---

## 5. API Methods (SDKClient)

### 5.1 Connection Lifecycle
- **`preConnect(): Promise<boolean>`**: Pre-fetches and caches connection config (TTL ~60s).
- **`connect(): Promise<void>`**: Establishes connection to all controllers and LiveKit.
- **`reconnect(): Promise<ConnectionSnapshot>`**: Manual reconnection. Disconnects current session, refreshes config based on mode, and reconnects.
- **`disconnect(): Promise<void>`**: Stops capture and cleans up session.

### 5.2 Media Control
- **`startAudioCapture()` / `stopAudioCapture()`**: Manage local microphone publishing.
- **`startCamera()` / `stopCamera()`**: Manage local camera publishing.
- **`setVolume(0..1)` / `setRenderFitMode(mode)`**: UI and output controls.

### 5.3 Session
- **`sendTextQuestion(text: string): Promise<string>`**: Sends a text query and returns the message UID (used as `questionId`).
- **`interrupt(): Promise<void>`**: Sends an interruption signal to the server.

---

## 6. Events

| Event | Trigger Timing | Payload |
| :--- | :--- | :--- |
| `sdk:connected` | Channels (LiveKit & HTTP) are ready | `{ livekit, http, all }` |
| `sdk:error` | Internal error mapped to safe payload | `{ message, code }` |
| `conversation:answer:chunk` | Partial answer text received | `{ questionId, chunk }` |
| `conversation:answer:completed` | Final answer text received | `{ questionId, fullAnswer }` |
| `media:video:available` | Remote video stream is ready | `undefined` |

---

## 7. Troubleshooting & Performance

### 7.1 Built-in Metrics
If `performanceMonitor` is enabled, the SDK tracks:
- `connect_to_first_frame_ms`: Time from `connect()` to first rendered frame.
- `text_send_to_text_response_ms`: Latency between query and first text chunk.

### 7.2 Common Issues
1. **Black Screen**: Ensure `containerElement` is a valid DOM node and `media:video:available` has fired.
2. **Auth Failure**: Check `http.baseURL` and ensure `setAuthToken` is called before `connect()`.
3. **Green Screen Lag**: Try reducing the container size or switch to `renderMode: 'raw'` to check for GPU bottlenecks.

---

_Document Version: 1.0.0_
