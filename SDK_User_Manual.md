# Facemarket Live Avatar SDK User Manual (v1.0.0)

This manual corresponds to the npm package **`@sanseng/livekit-ws-sdk` version 1.0.0**. The SDK is built on **LiveKit Client** and encapsulates live avatar audio/video downlink, microphone/camera uplink, session text, and the HTTP control plane (for fetching connection configurations in Auth mode).

---

## 1. Overview

The SDK provides a unified entry point via **`createClient` → `SDKClient`**, which is responsible for:

- Subscription and routing of audio/video tracks, as well as local capture and publishing.
- Encoding, decoding, and distribution of session-side text and protocol messages.
- Connection lifecycle management and state snapshots (for easier reconnection and troubleshooting).
- Optional chroma key (green screen) processing and performance metric reporting.

Integrators can complete the setup via constructor parameters, connection APIs, and event subscriptions without needing to understand internal SDK modules (e.g., `transport` implementation details).

---

## 2. Initialization

The SDK distinguishes between two mutually exclusive modes via `ClientOptions.connectConfig`: **Direct** and **Auth**. These two modes differ in their configuration sources, the availability of `setAuthToken` / `updateConnectionConfig`, and their `reconnect` refresh behaviors.

### 2.1 Direct Mode

**Description** The caller directly provides the LiveKit `sfuUrl` and `userToken` within the constructor parameters. The SDK does not pull room configurations via HTTP.

**Prerequisites**

- A non-empty `sfuUrl` (LiveKit WebSocket URL) and `userToken` (Room Access Token) must be provided.
- If the business logic requires changing the room or token during runtime, the new configuration must be injected via `updateConnectionConfig` before the next reconnection (refer to Behavioral Notes).

**Initialization**

```ts
import { createClient } from '@sanseng/livekit-ws-sdk';

const client = createClient({
  connectConfig: {
    type: 'direct',
    config: {
      sfuUrl: 'wss://your-livekit-host',
      userToken: 'your-room-token',
    },
  },
  video: {
    containerElement: document.getElementById('avatar')!,
  },
});
```

**Behavioral Notes**

- **`preConnect()` / `connect()`**: Resolves the `livekitUrl` and `token` from the current Direct configuration (or the staged replacement configuration from `updateConnectionConfig` / `reconnect` phases), writes them to the context, and establishes the LiveKit connection.
- **`reconnect()`**: After disconnecting the current session, it calls `refreshConfig()`. In Direct mode, it re-reads the configuration from the `ConfigManager` (including updates applied via `updateConnectionConfig` through `replaceDirectConfig`), without triggering any Auth HTTP calls to pull room data.
- **Token Refresh**: The SDK does not automatically "refresh" server-issued tokens. Token expiry must be handled by the business side by fetching a new token, calling `updateConnectionConfig`, and then triggering `reconnect()`.

**Usage Restrictions**

- Calling `updateConnectionConfig` while in Auth mode is unsupported (will throw `SDK_INVALID_STATE_TRANSITION`).
- Direct mode does not support `videoOptions` override strategies returned via HTTP interfaces. Video parameters such as green screen rely on the `video` options provided at construction or runtime methods like `setRenderFitMode` (unless the business implements its own local configuration sync).

**Use Cases** Existing backends that stably issue LiveKit tokens, private cloud deployments, or rapid integration during the debugging phase.

---

### 2.2 Auth Mode

**Description** The caller provides an `avatarId` (and optionally `authToken` or `avatarVoice`). The SDK fetches the authentication and room configuration (`ConnectionConfig`) via the `HttpController`, and can merge server-returned `videoOptions` (e.g., green screen toggles) into the context.

**Prerequisites**

- A non-empty `connectConfig.config.avatarId` must be provided.
- A valid authentication token must be accessible during `preConnect()` / `connect()`: either passed via `connectConfig.config.authToken` or written to the context by calling `setAuthToken(token)` after instantiation but before connecting.
- A functional HTTP service (default or custom `http.baseURL`) must be provided for fetching authentication and LiveKit room configurations.

**Initialization**

```ts
import { createClient } from '@sanseng/livekit-ws-sdk';

const client = createClient({
  connectConfig: {
    type: 'auth',
    config: {
      avatarId: 'your-avatar-id',
      avatarVoice: 'optional-voice-id',
      // authToken can be omitted here and set later via client.setAuthToken('...')
    },
  },
  http: {
    baseURL: 'https://your-api.example.com/...',
    headers: {
      /* Optional */
    },
  },
  video: {
    containerElement: document.getElementById('avatar')!,
  },
});

client.setAuthToken('jwt-or-business-token');
```

**Behavioral Notes**

- **`preConnect()` / `connect()`**: Fetches the token and connection payload via HTTP, resolves them into the `livekitUrl`, `token`, `roomId`, and optional `videoOptions`, and then establishes the LiveKit connection.
- **`reconnect()`**: After disconnecting, it calls `refreshConfig()`, which **re-triggers the HTTP request** to fetch the latest `ConnectionConfig` (the cache is cleared and reloaded).
- **`setAuthToken`**: Updates the token in the context. If the `HttpController` already exists, it synchronizes the authentication retrieval logic.

**Usage Restrictions**

- Manually replacing the LiveKit URL/Token via `updateConnectionConfig` is not supported in Auth mode (sessions should be refreshed server-side followed by a `reconnect()` call to re-pull data).
- `connectConfig.config.authToken` and `setAuthToken` serve as the same semantic data source; ensure at least one is valid before the initial `connect()`.

**Use Cases** Production environments where the business backend manages centralized authentication and distributes room parameters or video strategies (like green screen) uniformly.

---

### 2.3 Mode Comparison

| Feature | Direct Mode | Auth Mode |
| :--- | :--- | :--- |
| **Config Source** | `sfuUrl` and `userToken` from constructor parameters. | `ConnectionConfig` returned from HTTP interface. |
| **Business HTTP Required** | No (Optional `http` config if using other HTTP capabilities). | **Yes** (required for auth and room config). |
| **Required Fields** | `sfuUrl` + `userToken` | `avatarId` + valid `authToken` (constructor or `setAuthToken`). |
| **`setAuthToken`** | Not used for resolving LiveKit connection config. | Mandatory or recommended to inject before connecting. |
| **`updateConnectionConfig`**| Available; applies to the Direct config used in the **next** `reconnect()`. | Unavailable (throws error). |
| **`reconnect()` Refresh** | Uses `refreshConfig()` to read Direct path config (includes updates via `replaceDirectConfig`). | Uses `refreshConfig()` to re-fetch via HTTP. |

---

## 3. Quick Start

### 3.1 Installation

```bash
npm install @sanseng/livekit-ws-sdk
```

### 3.2 Minimal Workflow (Auth Example)

```ts
import { createClient } from '@sanseng/livekit-ws-sdk';

const el = document.getElementById('avatar');
if (!el) throw new Error('container missing');

const client = createClient({
  connectConfig: { type: 'auth', config: { avatarId: 'demo-avatar' } },
  http: { baseURL: 'https://your-dispatcher.example.com/...' },
  video: { containerElement: el, fitMode: 'contain' },
  debug: true,
});

// Must be issued by your business backend. The specific request implementation is omitted here.
client.setAuthToken('REPLACE_WITH_TOKEN_FROM_YOUR_BACKEND');

client.events.on('sdk:connected', ({ livekit, http, all }) => {
  console.log('channels', { livekit, http, all });
});

try {
  await client.preConnect();
  await client.connect();
} catch (e) {
  console.error(e);
}

// Start microphone after a user click (to satisfy browser autoplay/capture policies)
document.getElementById('mic')?.addEventListener('click', async () => {
  await client.startAudioCapture();
});
```

### 3.3 Minimal Workflow (Direct Example)

```ts
const client = createClient({
  connectConfig: {
    type: 'direct',
    config: { sfuUrl: 'wss://...', userToken: '...' },
  },
  video: { containerElement: document.getElementById('avatar')! },
});

await client.connect();
```

---

## 4. Core Concepts

### 4.1 questionId (Turn Correlation Identifier)

`questionId` is the correlation key for a **single complete Q&A turn** (User Query → Server Processing → Streaming/Full Answer). It is used to link:

- `conversation:question:sent`
- `conversation:answer:waiting`
- `conversation:server:message`
- `conversation:asr:received` / `conversation:asr:streaming`
- `conversation:answer:chunk`
- `conversation:answer:completed`

**Convention**: `questionId` is a short string (typically **4 characters** in practice) that is only guaranteed to be unique within the current session; it is **not suitable** for use as a long-term persistent primary key.

**Strong Recommendation**: Upon receiving `conversation:answer:completed`, clear the local buffers (text concatenation, UI states, etc.) associated with that `questionId` to prevent memory growth and display errors caused by ID reuse.

### 4.2 Control Plane and Media Plane

- **Media Plane**: Handled by `LiveKitService`, which manages LiveKit room connections, audio/video track subscription and publishing, and the transmission of session text and protocol messages via the **Data Channel**.
- **Control Plane (Auth)**: Handled by `HttpController` + `ConfigManager`. This plane is responsible for fetching the `livekitUrl`, `token`, `roomId`, and optional server-side `videoOptions` (such as green screen toggles) in Auth Mode. There is **no** independent "Business WebSocket" channel parallel to LiveKit exposed in the external API.

### 4.3 Video Rendering Modes

- `video.renderMode === 'raw'`: Remote tracks are rendered directly onto the `<video>` element (default behavior depends on options).
- `video.renderMode === 'processed'`: Renders via a Canvas processing pipeline (e.g., for green screen/chroma key).
- **In Auth Mode**, the green screen configuration returned by the server is mapped to `renderMode` / `greenScreen` (refer to `ConfigManager._fromAuthPayload`). If neither the local nor remote side provides an explicit configuration, the SDK's internal strategy will determine the mode.

---

## 5. Configuration (`createClient` Parameters)

`createClient(options: ClientOptions)`. The definition of `ClientOptions` can be found in `src/types/ClientOptions.d.ts`.

### 5.1 Top-level Fields

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `connectConfig` | `direct \| auth` Union | **Yes** | Connection mode and mode-specific configurations. |
| `audio` | `AudioOptions` | No | Input/Output audio options. |
| `video` | `VideoOptions` | No | Rendering container, chroma key, `renderMode`, `fitMode`, etc. |
| `reconnect` | `{ maxAttempts?, delay? }` | No | LiveKit reconnection: `maxAttempts` defaults to `3`; `delay` defaults to `10` (**seconds**), converted internally to ms. |
| `http` | `{ baseURL?, headers? }` | No | Root path and default headers for Auth and HTTP control interfaces. |
| `performanceMonitor`| `PerformanceMonitorOptions`| No | Enabled by default; set `enabled: false` to disable. |
| `sandbox` | `boolean` | No | Sandbox toggle (based on backend business logic agreement). |
| `debug` | `boolean` | No | Enables debug logging. |

### 5.2 `connectConfig`

**Direct Mode**

| Field | Type | Required |
| :--- | :--- | :--- |
| `type` | `'direct'` | **Yes** |
| `config.sfuUrl` | `string` | **Yes** |
| `config.userToken` | `string` | **Yes** |

**Auth Mode**

| Field | Type | Required |
| :--- | :--- | :--- |
| `type` | `'auth'` | **Yes** |
| `config.avatarId` | `string` | **Yes** |
| `config.authToken` | `string` | No (Can be used interchangeably or combined with `setAuthToken`). |
| `config.avatarVoice` | `string` | No |

### 5.3 `video` (`VideoOptions`)

| Field | Description |
| :--- | :--- |
| `containerElement` | The container where the video is mounted (**Ensure it exists and is inserted into the DOM**). |
| `renderMode` | `'raw'` \| `'processed'` |
| `greenScreen` | `{ enabled, chromaKey?, similarity?, smoothness?, despillStrength? }` |
| `fitMode` | `'contain' \| 'cover' \| 'fill' \| 'none'` |
| `debug` | Inherited from `BaseOptions`. |

### 5.4 `audio` (`AudioOptions`)

| Field | Description |
| :--- | :--- |
| `input` | `{ deviceId?: string, sampleRate?: number; noiseSuppression?: boolean;}` (Device constraints, etc.) |
| `output` | `{ enabled?: boolean, volume?: number, muted?: boolean }` Default values for the playback side. |

### 5.5 `performanceMonitor`

| Field | Description |
| :--- | :--- |
| `enabled` | Defaults to `true`; set to `false` to disable built-in performance collection. |
| `reporter` | `(metric: PerformanceMetricRecord) => void` Custom reporting callback. |

---

## 6. Core API Methods

All methods below are defined in `SDKClient` (`src/client/SDKClient.ts`). Except for the constructor, `setAuthToken`, and `updateConnectionConfig`, most media and session APIs require a **successful `connect()`** (verified internally via `sessionState.isConnected`).

### 6.1 Connection & Lifecycle

| Method | Description |
| :--- | :--- |
| `preConnect(): Promise<boolean>` | Pre-fetches and caches connection config (TTL is ~60s, see `PRECONNECT_CACHE_DURATION_MS`). Throws on failure. |
| `connect(): Promise<void>` | Establishes the Coordinator, Domain Controllers, and LiveKit connection. Calls pre-connect internally if no valid cache exists. |
| `disconnect(): Promise<void>` | Stops capture, disconnects LiveKit, and cleans up HTTP sessions. The instance can still be used for `connect()` or `reconnect()`. |
| `reconnect(): Promise<ConnectionSnapshot>` | Manual reconnection: `disconnect()` is called first, then config is refreshed based on mode before calling `connect()`. Returns a snapshot if reconnection is not allowed. |
| `dispose(): void` | Releases all resources. The instance **must not** be used after this call. |

**Constraint**: Due to browser policies, **audio playback** and **microphone capture** should ideally be triggered by `connect()` / `startAudioCapture()` within a user gesture (e.g., click) callback.

### 6.2 Authentication & Connection Config

| Method | Description |
| :--- | :--- |
| `setAuthToken(token: string): void` | Sets the auth token; used by HTTP and Config components in **Auth Mode**. |
| `updateConnectionConfig(config: DirectConnectionConfig): void` | **Direct Mode only**. Validates and stages `sfuUrl` / `userToken`. **Does not affect the current session**; takes effect during the next `reconnect()` via `replaceDirectConfig`. |

### 6.3 Media

| Method | Description |
| :--- | :--- |
| `setRenderFitMode(mode: RenderFitMode): void` | Updates the video object-fit mode. |
| `startAudioCapture(): Promise<void>` | Opens the microphone and publishes to LiveKit. |
| `stopAudioCapture(): Promise<void>` | Stops microphone publishing. |
| `setVolume(volume: number): void` | Sets playback volume (`0` to `1`). |
| `getVolume(): number` | Gets current playback volume. |
| `mute()` / `unmute()` | Controls playback muting. |
| `get isMuted` | Returns whether playback is muted. |
| `get isAudioCapturing` | Returns whether the microphone is actively publishing. |
| `startCamera(): Promise<void>` | Opens the camera and publishes. |
| `stopCamera(): void` | Stops the camera. |
| `getCameraStream(): MediaStream \| null` | Returns the local stream for previewing. |
| `getCameraTrack(): MediaStreamTrack \| null` | Returns the local media track. |
| `attachCameraTo(el: HTMLVideoElement): void` | Binds the local camera stream to a `<video>` element. |

### 6.4 Session

| Method | Description |
| :--- | :--- |
| `sendTextQuestion(text: string): Promise<string>` | Sends a text query; returns a **Message UID** (used as `questionId` in events for turn correlation). |
| `interrupt(): Promise<void>` | Sends an interruption control event (`control.interrupt`). |

### 6.5 State & Observability

| Member | Description |
| :--- | :--- |
| `get isConnected(): boolean` | Whether the session is currently connected. |
| `get connectionSnapshot(): ConnectionSnapshot` | Synchronous read-only snapshot: `http.connected`, `rtc.connected`, `rtc.hasVideoTrack`, `overall.state`. |
| `setPerformanceMetricReporter(cb?: PerformanceMetricReporter): void` | Sets or updates the performance metric reporting callback. |
| `get events(): PublicEmitterAPI` | Accesses the event emitter (supports `on`, `off`, `once` only). |

---

## 7. Public Events

Events are subscribed to via `client.events.on(eventName, listener)`. Only the event names in the following whitelist are supported; other names will result in an error.

**Connection**

### `sdk:connected`

- **Trigger**: Dispatched when the aggregated state of internal `livekit` and `http` connections changes and differs from the previous state.
- **Payload**: `{ livekit: boolean; http: boolean; all: boolean }`, where `all === livekit && http`.
- **Description**: Used for a high-level overview of whether both channels are ready.

### `sdk:disconnected`

- **Trigger**: Dispatched when a disconnection in either path causes a change in the aggregated state.
- **Payload**: Same structure as `sdk:connected`: `{ livekit: boolean; http: boolean; all: boolean }` (refer to runtime payload if it differs from `reason?` in type definitions).
- **Description**: Can be cross-referenced with `connectionSnapshot` for validation.

### `sdk:error`

- **Trigger**: Dispatched when an internal error is mapped to a secure external payload.
- **Payload**: `{ message: string; code: string }` (`code` is the string value of `ErrorCode`).

**Video**

| Event | Trigger | Payload |
| :--- | :--- | :--- |
| `media:video:available` | Remote video is available (first frame semantics driven by RTC layer). | `undefined` |
| `media:video:unavailable` | Remote video is no longer available. | `undefined` |
| `media:video:trackAdded` | Video track added. | `undefined` (de-duplication handled internally). |
| `media:video:trackRemoved` | Video track removed. | `undefined` |

**Remote Audio**

| Event | Trigger | Payload |
| :--- | :--- | :--- |
| `media:audio:trackAdded` | Remote audio track added. | `undefined` |
| `media:audio:trackRemoved` | Remote audio track removed. | `undefined` |

**Local Microphone**

| Event | Trigger | Payload |
| :--- | :--- | :--- |
| `media:audio:captureStarted` | Local input capture started. | `undefined` |
| `media:audio:captureStopped` | Local input capture stopped. | `undefined` |

**Local Camera**

| Event | Trigger | Payload |
| :--- | :--- | :--- |
| `media:camera:started` | Camera capture started. | `undefined` |
| `media:camera:stopped` | Camera capture stopped. | `undefined` |

**Playback Volume**

| Event | Trigger | Payload |
| :--- | :--- | :--- |
| `media:audio:volumeChanged` | Volume level changed. | `{ volume: number }` |
| `media:audio:muted` | Output muted. | `undefined` |
| `media:audio:unmuted` | Output unmuted. | `undefined` |

**Conversation**

| Event | Trigger | Payload |
| :--- | :--- | :--- |
| `conversation:question:sent` | Question successfully sent. | `{ questionId: string; text: string }` |
| `conversation:answer:waiting` | Waiting for answer. | `{ questionId: string }` |
| `conversation:server:message` | Server-side message. | `{ questionId: string; message: string; type: string }` |
| `conversation:asr:received` | ASR final result received. | `{ questionId: string; text: string }` |
| `conversation:asr:chunk` | ASR text chunk received. | `{ questionId: string; text: string; isComplete: boolean }` |
| `conversation:answer:chunk` | Answer text chunk received. | `{ questionId: string; chunk: string }` (End of stream is determined by the `completed` event). |
| `conversation:answer:completed` | Single turn answer completed. | `{ questionId: string; fullAnswer: string }` |

---

## 8. Full Usage Example

```ts
import { createClient, type PerformanceMetricRecord } from '@sanseng/livekit-ws-sdk';

async function main() {
  const container = document.getElementById('avatar');
  if (!container) return;

  const client = createClient({
    connectConfig: {
      type: 'auth',
      config: { avatarId: 'demo-avatar' },
    },
    http: { baseURL: 'https://your-api.example.com/s2/aigc/api/vih_dispatcher' },
    video: { containerElement: container, fitMode: 'contain' },
    performanceMonitor: {
      reporter: (m: PerformanceMetricRecord) => console.log('[perf]', m.metric, m.durationMs),
    },
  });

  client.setAuthToken('REPLACE_WITH_TOKEN_FROM_YOUR_BACKEND');

  const answerByQuestion = new Map<string, string>();

  client.events.on('conversation:answer:chunk', ({ questionId, chunk }) => {
    const prev = answerByQuestion.get(questionId) ?? '';
    answerByQuestion.set(questionId, prev + chunk);
    document.getElementById('answer')!.textContent = prev + chunk;
  });

  client.events.on('conversation:answer:completed', ({ questionId }) => {
    answerByQuestion.delete(questionId);
  });

  client.events.on('sdk:error', ({ code, message }) => {
    console.error(code, message);
  });

  document.getElementById('connect')!.onclick = async () => {
    try {
      await client.connect();
    } catch (e) {
      console.error(e);
    }
  };

  document.getElementById('send')!.onclick = async () => {
    if (!client.isConnected) return;
    const text = (document.getElementById('q') as HTMLInputElement).value;
    if (text) await client.sendTextQuestion(text);
  };

  document.getElementById('interrupt')!.onclick = async () => {
    await client.interrupt();
  };

  document.getElementById('reconnect')!.onclick = async () => {
    await client.reconnect();
  };

  document.getElementById('disconnect')!.onclick = async () => {
    await client.disconnect();
  };

  document.getElementById('mic')!.onclick = async () => {
    if (!client.isAudioCapturing) await client.startAudioCapture();
    else await client.stopAudioCapture();
  };
}

main();
```

**Updating room/token in Direct Mode**:

```ts
client.updateConnectionConfig({ sfuUrl: 'wss://new-host', userToken: 'new-token' });
await client.reconnect();
```

---

## 9. Video Chroma Key (Green Screen) Debugging Guide

Before enabling Chroma Key, ensure the following settings are applied (or leave them unconfigured for the SDK's internal auto-detection):

- `video.renderMode = 'processed'`
- `greenScreen.enabled = true`

### 9.1 Tuning Recommendations

1. **Background Color Selection (`chromaKey`)**
   - Recommended to pick colors from actual video screenshots.
   - Avoid using generic pure green (#00FF00).
   - Only green hues are supported.

2. **Similarity (`similarity`)**
   - Start from `0.3` and adjust incrementally.
   - Values too high may erroneously remove person details (e.g., hair or clothing).

3. **Green Spill Suppression (`despillStrength`)**
   - Used to reduce green reflections/fringes on the subject's edges.

4. **Edge Smoothness (`smoothness`)**
   - Improves blending for hair and semi-transparent areas.

---

## 10. FAQ

**No video display or black screen**

1. Verify if a valid DOM node has been passed to `video.containerElement`.
2. Ensure `connect()` was successful and `client.isConnected === true`.
3. Check if the `media:video:available` or at least `media:video:trackAdded` event was received (can be used to trigger loading states).

**`preConnect` / `connect` failure in Auth Mode**

1. Check if `http.baseURL` points to the correct environment and if `setAuthToken` was called before connecting.
2. Enable `debug: true` to inspect the console logs and check the `code` in the `sdk:error` event.

**`reconnect()` still uses old token in Direct Mode**

1. Ensure `updateConnectionConfig` was called *before* `reconnect()`. This method **does not** affect the current session; it only applies to subsequent reconnection attempts.

**Stuttering with Green Screen**

1. Try reducing the rendering resolution of the container or switch to `renderMode: 'raw'` to verify if it is a CPU/GPU bottleneck.

---

## 11. Error Codes

The following are the string values for the `ErrorCode` enum (`src/errors/ErrorCodes.ts`), consistent with the `code` field in `sdk:error` and thrown `SDKError` objects.

### 11.1 HTTP & Control Plane

| Code | Description & Recommendation |
| :--- | :--- |
| `SDK_AUTH_TOKEN_FAILED` | Auth HTTP request failed. Check token and gateway URL. |
| `SDK_GET_LIVEKIT_CONFIG_FAILED` | Failed to fetch room config. Ensure backend returns `livekitUrl` / `roomToken`. |
| `SDK_SWITCH_VIDEO_FAILED` | Control logic for switching video failed. Check logs for details. |
| `SDK_INTERRUPT_CONVERSATION_FAILED`| Failed to send interruption command. |
| `HTTP_CONTROLLER_NOT_AVAILABLE` | HTTP controller is not ready. Avoid calling HTTP-dependent operations. |

### 11.2 SDK Lifecycle

| Code | Description & Recommendation |
| :--- | :--- |
| `SDK_PRECONNECT_FAILED` | Pre-connection failed. Check network, Direct params, or Auth token. |
| `SDK_CONNECT_FAILED` | Connection failed. Check preceding error messages. |
| `SDK_INITIALIZATION_FAILED` | Failed during initialization. Check console stack trace. |
| `SDK_DISCONNECT_FAILED` | Error during disconnect. Log the error and consider creating a new instance. |
| `SDK_RECONNECT_FAILED` | Manual reconnection failed. |
| `SDK_NOT_CONNECTED` | API called without an active connection. Call `connect()` or check `isConnected`. |
| `SDK_INVALID_STATE_TRANSITION` | Illegal state transition (e.g., calling `updateConnectionConfig` in Auth mode). |
| `SDK_ERROR` | Generic fallback error code. |

### 11.3 LiveKit / RTC

| Code | Description & Recommendation |
| :--- | :--- |
| `LIVEKIT_CONNECT_FAILED` | Room connection failed. Verify URL/token/TURN network status. |
| `LIVEKIT_SEND_VIDEO_AVAILABLE_STATE_FAILED` | Failed to report video available state. |
| `LIVEKIT_SEND_TEXT_DATA_FAILED` | Failed to send text data via LiveKit Data Channel. |
| `LIVEKIT_DATA_MESSAGE_PARSE_ERROR` | Failed to parse data message. Check protocol version compatibility. |
| `LIVEKIT_UNPUBLISH_MICROPHONE_FAILED` | Failed to unpublish the microphone track. |

### 11.4 Audio

| Code | Description & Recommendation |
| :--- | :--- |
| `AUDIO_CAPTURE_START_FAILED` | Mic start failed. Check user gestures, HTTPS, and permissions. |
| `AUDIO_CAPTURE_FAILED` | Capture interrupted. |
| `AUDIO_INVALID_SAMPLE_RATE` / `AUDIO_INVALID_CHANNEL` / `AUDIO_INVALID_CODEC` | Parameter mismatch with device or protocol. |
| `AUDIO_INVALID_HEADER_LENGTH` / `AUDIO_INVALID_TYPE` / `AUDIO_INVALID_RESERVED` | Illegal uplink packet header. |
| `AUDIO_CONTROLLER_NOT_AVAILABLE` | Audio controller not created or already released. |

### 11.5 Camera

| Code | Description |
| :--- | :--- |
| `CAMERA_CONTROLLER_NOT_AVAILABLE` | Camera controller is unavailable. |

### 11.6 Session & State Machine

| Code | Description & Recommendation |
| :--- | :--- |
| `CONVERSATION_CONTROLLER_NOT_AVAILABLE` | Conversation controller is unavailable. |
| `STATE_MACHINE_INVALID_STATE_TRANSITION` | Internal state machine received illegal transition. Provide logs for feedback. |

### 11.7 Utilities & Others

| Code | Description |
| :--- | :--- |
| `OBJECT_DISPOSED` | Instance has been `dispose()`-ed. |
| `NO_AVATARID` | Missing `avatarId` in Auth configuration. |
| `NO_AUTH_TOKEN` | Missing authentication token (validated at runtime). |
| `INVALID_CONNECT_CONFIG` | Invalid Direct configuration during `createClient`. |

---

## 12. Performance Monitoring & Troubleshooting

### 12.1 Built-in Metrics

Enabled by default (`performanceMonitor.enabled !== false`). The metric names under `PerformanceMetricName` include:

| `metric` | Meaning |
| :--- | :--- |
| `connect_to_first_frame_ms` | Time from `connect()` call to completion of first frame rendering. |
| `text_send_to_text_response_ms` | Time from sending text query to receiving the first text response chunk. |
| `text_send_to_audio_response_ms` | Time from sending text query to receiving the first audio response chunk. |
| `no_speech_report_to_audio_response_ms` | Time from "no-speech" report to receiving the audio response. |

Data structure: `PerformanceMetricRecord` (contains `metric`, `durationMs`, `startedAt`, `endedAt`, and optional `questionId`).

### 12.2 Custom Reporting

```ts
import { createClient, type PerformanceMetricRecord } from '@sanseng/livekit-ws-sdk';

const client = createClient({
  connectConfig: { type: 'auth', config: { avatarId: 'demo' } },
  performanceMonitor: {
    reporter: (metric: PerformanceMetricRecord) => {
      console.log('[perf]', metric.metric, `${metric.durationMs}ms`, metric.questionId);
    },
  },
});
```

Or update at runtime:

```ts
client.setPerformanceMetricReporter((metric) => {
  /* Report to OTLP / Custom analytics service */
});
```

### 12.3 Troubleshooting Recommendations

- **Slow First Frame**: Check region latency, TURN server status, and whether repeated `connect()` retries are occurring.
- **Slow Text Response**: Correlate with backend LLM/business processing time and check for main-thread blocking.
- **Audio Significantly Slower Than Text**: Inspect the TTS pipeline and remote audio track subscription states.

---

*Document version consistent with package version: 1.0.0.*
