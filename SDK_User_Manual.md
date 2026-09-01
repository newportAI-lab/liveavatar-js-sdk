# Facemarket Live Avatar SDK User Manual (v2.2.0)

This manual corresponds to the npm package **`@sanseng/liveavatar-js-sdk` version 2.2.0**. The SDK is built on **LiveKit Client** and encapsulates live avatar audio/video downlink, microphone/camera uplink, session text, and the HTTP control plane (for fetching connection configurations in Auth mode).

---

## 1. Overview

The SDK provides a unified entry point via **`createClient` → `SDKClient`**, which is responsible for:

- Subscription and routing of audio/video tracks, as well as local capture and publishing.
- Encoding, decoding, and distribution of session-side text and protocol messages.
- Connection lifecycle management and state snapshots (for easier reconnection and troubleshooting).
- Optional chroma key (green screen) processing and performance metric reporting.

Integrators can complete the setup via constructor parameters, connection APIs, and event subscriptions without needing to understand internal SDK modules (e.g., `transport` implementation details).

---

## 2. Breaking Change Notice (v1.3.0)

**This version is incompatible with v1.2.1.**

v1.3.0 adapts to changes in the backend interface, and the logic for receiving text messages has changed. Please ensure the backend has been updated before upgrading. Re-test all text message-related functionality after upgrading.

---

## 3. Initialization

The SDK distinguishes between two mutually exclusive modes via `ClientOptions.connectConfig`: **Direct** and **Auth**. These two modes differ in their configuration sources, the availability of `setAuthToken` / `updateConnectionConfig`, and their `reconnect` refresh behaviors.

### 2.1 Direct Mode

**Description** The caller directly provides the LiveKit `sfuUrl` and `userToken` within the constructor parameters. The SDK does not pull room configurations via HTTP.

**Prerequisites**

- A non-empty `sfuUrl` (LiveKit WebSocket URL) and `userToken` (Room Access Token) must be provided.
- If the business logic requires changing the room or token during runtime, the new configuration must be injected via `updateConnectionConfig` before the next reconnection (refer to Behavioral Notes).

**Initialization**

```ts
import { createClient } from '@sanseng/liveavatar-js-sdk';

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
import { createClient } from '@sanseng/liveavatar-js-sdk';

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
| **`updateConnectionConfig`**| Available; applies to the Direct config used in the **next** `preConnect()` / `connect()` or `reconnect()`. | Unavailable (throws error). |
| **`reconnect()` Refresh** | Uses `refreshConfig()` to read Direct path config (includes updates via `replaceDirectConfig`). | Uses `refreshConfig()` to re-fetch via HTTP. |

---

## 4. Quick Start

### 4.1 Installation

```bash
npm install @sanseng/liveavatar-js-sdk
```

### 4.2 Minimal Workflow (Auth Example)

```ts
import { createClient } from '@sanseng/liveavatar-js-sdk';

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

client.events.on('sdk:connected', ({ livekitConnected, httpConnected, allConnected }) => {
  console.log('channels', { livekitConnected, httpConnected, allConnected });
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

### 4.3 Minimal Workflow (Direct Example)

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

## 5. Core Concepts

### 5.1 questionId (Turn Correlation Identifier)

`questionId` is the correlation key for a **single complete Q&A turn** (User Query → Server Processing → Streaming/Full Answer). It is used to link:

- `conversation:question:sent`
- `conversation:answer:waiting`
- `conversation:server:message`
- `conversation:asr:received` / `conversation:asr:chunk`
- `conversation:answer:chunk`
- `conversation:answer:completed`

**Convention**: `questionId` is a short string (typically **4 characters** in practice) that is only guaranteed to be unique within the current session; it is **not suitable** for use as a long-term persistent primary key.

**Strong Recommendation**: Upon receiving `conversation:answer:completed`, clear the local buffers (text concatenation, UI states, etc.) associated with that `questionId` to prevent memory growth and display errors caused by ID reuse.

### 5.2 Control Plane and Media Plane

- **Media Plane**: Handled by `LiveKitService`, which manages LiveKit room connections, audio/video track subscription and publishing, and the transmission of session text and protocol messages via the **Data Channel**.
- **Control Plane (Auth)**: Handled by `HttpController` + `ConfigManager`. This plane is responsible for fetching the `livekitUrl`, `token`, `roomId`, and optional server-side `videoOptions` (such as green screen toggles) in Auth Mode. There is **no** independent "Business WebSocket" channel parallel to LiveKit exposed in the external API.

### 5.3 Video Rendering Modes

- `video.renderMode === 'raw'`: Remote tracks are rendered directly onto the `<video>` element (default behavior depends on options).
- `video.renderMode === 'processed'`: Renders via a Canvas processing pipeline (e.g., for green screen/chroma key).
- **In Auth Mode**, the green screen configuration returned by the server is mapped to `renderMode` / `greenScreen` (refer to `ConfigManager._fromAuthPayload`). If neither the local nor remote side provides an explicit configuration, the SDK's internal strategy will determine the mode.

---

## 6. Configuration (`createClient` Parameters)

`createClient(options: ClientOptions)`. The definition of `ClientOptions` can be found in `src/types/ClientOptions.d.ts`.

### 6.1 Top-level Fields

| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `connectConfig` | `direct \| auth` Union | **Yes** | Connection mode and mode-specific configurations. |
| `audio` | `AudioOptions` | No | Input/Output audio options. |
| `video` | `VideoOptions` | No | Rendering container, chroma key, `renderMode`, `fitMode`, etc. |
| `reconnect` | `{ maxAttempts?, delay? }` | No | LiveKit reconnection: `maxAttempts` defaults to `3`; `delay` defaults to `10` (**seconds**), converted internally to ms. |
| `http` | `{ baseURL?, headers? }` | No | Root path and default headers for Auth and HTTP control interfaces. |
| `performanceMonitor` | `PerformanceMonitorOptions` | No | Enabled by default; set `enabled: false` to disable. |
| `sandbox` | `boolean` | No | Sandbox toggle (based on backend business logic agreement). |
| `debug` | `boolean \| DebugOptions` | No | Optional Debug, LiveKit verbose, and A/V diagnostic configuration; see Section 7, “Debug and A/V Diagnostics.” |
| `plugins` | `readonly SDKPlugin[]` | No | Plugins installed at construction. Pass an `SDKPlugin` imported from a plugin package; one failed install does not prevent client construction. See the [Plugin Integration Guide](./SDK_Plugin_Integration_Guide.md). |

This chapter lists constructor parameters only. See Section 7 for the full Debug logging, A/V diagnostics, automatic video-stall alerts, and experimental frame-metadata worker configuration and behavior.

### 6.2 `connectConfig`

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

### 6.3 `video` (`VideoOptions`)

| Field | Description |
| :--- | :--- |
| `containerElement` | The container where the video is mounted (**Ensure it exists and is inserted into the DOM**). |
| `renderMode` | `'raw'` \| `'processed'` |
| `greenScreen` | `{ enabled, chromaKey?, similarity?, smoothness?, despillStrength? }` |
| `camera` | `{ publishToLiveKit?: boolean }` | No | Default `false`. When `true`, `startCamera()` publishes the track to the LiveKit room. Can be overridden per-call via `startCamera({ publishToLiveKit: true })`. |
| `fitMode` | `'contain' \| 'cover' \| 'fill' \| 'none'` |
| `debug` | Inherited from `BaseOptions`. |

### 6.4 `audio` (`AudioOptions`)

| Field | Description |
| :--- | :--- |
| `input` | `{ deviceId?: string, sampleRate?: number, channelCount?: number, sampleSize?: number, noiseSuppression?: boolean, voiceIsolation?: boolean, bitDepth?: number, echoCancellation?: boolean, autoGainControl?: boolean, constraints?: MediaTrackConstraints }` |
| `output` | `{ enabled?: boolean, volume?: number, muted?: boolean }` Default values for the playback side. |

**Note**: `channelCount` is optional and defaults to `1`. `noiseSuppression`, `echoCancellation`, and `autoGainControl` default to `true` when not explicitly set.

### 6.5 `performanceMonitor`

| Field | Description |
| :--- | :--- |
| `enabled` | Defaults to `true`; set to `false` to disable built-in performance collection. |
| `reporter` | `(metric: PerformanceMetricRecord) => void` Custom reporting callback. |

### 6.6 `MicrophoneStats`

| Field | Description |
| :--- | :--- |
| `bytesSent` | Number of bytes sent. |
| `packetsSent` | Number of packets sent. |
| `packetsLost` | Number of lost packets. |
| `roundTripTime` | Round-trip time in milliseconds. |

---

## 7. Debug and A/V Diagnostics

### 7.1 Basic Debug and LiveKit Verbose

`debug: true` and `debug: {}` enable SDK Debug logs only; only `debug: { verbose: true }` enables LiveKit JS SDK internal Debug logs. SDK and LiveKit loggers are runtime-global in the current page: each client creation applies its configuration; SDK logging returns to `ERROR` when `debug` is `false` or omitted, and LiveKit logging returns to `info` unless `verbose: true`. Disposing a client also restores both defaults. Therefore, the last created or disposed SDK client affects the log level of other SDK or LiveKit clients in the same page.

### 7.2 Periodic A/V Diagnostics (`debug.avSync`)

Use the following configuration to sample remote audio/video receiver statistics after the LiveKit Room connects; no polling runs by default:

```ts
debug: {
  avSync: { enabled: true, intervalMs: 2000, historyLimit: 60 },
}
```

Background sampling waits up to 10 seconds for the first remote audio or video track accepted by the renderer-participant filter. On timeout, it stops for the current connection session and retains `remote-track-wait-timeout` for upload. This timeout does not affect manual `captureRemoteAvSyncDiagnostics()`.

Offline evidence is returned by pages from `getAvSyncRawTelemetry(query?)`, not `[RTC remote-media]` Console output: the no-argument call returns at most the latest five minutes and 200 logs, while `range`, `pageSize`, and `page.nextCursor` retrieve further records. Log queries and fact-only summaries run in a Dedicated Worker; manual capture still reads current WebRTC stats on the browser main thread. `avSyncLogRetention` reports retained counts, storage errors, and drops. Room disconnect freezes the final session for paged queries; reconnect replaces it, while `dispose()` and failed connection attempts clear it.

### 7.3 Automatic Video-Stall Alerts

With `debug.avSync` enabled, callers do not need to poll `getAvSyncAnalysis()` to receive automatic video-pipeline alerts. After the shared `<video>` has emitted `playing`, the SDK confirms receive, decode, and browser-presentation stalls as `video_receive`, `video_decode`, and `video_render` after three compatible periodic samples. `video_render` is detected only after a supported `requestVideoFrameCallback` has produced a presentation baseline while the page is visible. The SDK does not reconnect, re-subscribe, or restart playback automatically.

Automatic alerts form one continuous incident lifecycle: the first confirmed incident emits `phase: "raised"`; the same confirmed classification remains active but intentionally emits no duplicate event or `video-pipeline-stalled` log; a newly confirmed receive/decode/render classification on the same video track emits `phase: "updated"`; only direct positive progress at the affected receive/decode/render layer emits `phase: "recovered"`. Therefore, the absence of a new alert event or retained log **does not mean recovery**.

Every lifecycle payload has an opaque `alertId`. A `raised`, `updated`, and final `recovered` for one active incident retain the same ID. `updated` supplies `previousFinding` and may change `finding`; applications must keep the incident active until `recovered`, rather than using no new alert as a normal signal:

```ts
const activeAlerts = new Map<string, AvSyncAlertEvent>();

client.events.on(`sdk:avSyncAlert`, (alert) => {
  if (alert.phase === "recovered") {
    activeAlerts.delete(alert.alertId);
    return;
  }

  // Both raised and updated mean the incident remains active.
  activeAlerts.set(alert.alertId, alert);
});
```

All alert phases reach the stable public `sdk:avSyncAlert` event and can also be received by installed plugins; see the [Plugin Integration Guide](./SDK_Plugin_Integration_Guide.md) for plugin integration. Only the first `raised` writes the retained `video-pipeline-stalled` diagnostic fact. The SDK never uploads alerts or logs automatically. Listening to the public event without polling `getAvSyncAnalysis()` covers these automatic periodic-observation transitions. However, `getAvSyncAnalysis()` also evaluates the retained window for track binding, transport degradation, audio concealment, playback, and relative drift. Those manual-analysis findings can become alerts only after the application calls the analysis API.

### 7.4 Experimental Frame Metadata Worker

Periodic A/V statistics and TimeSync do not enable the experimental frame metadata worker by default. To enable it, periodic diagnostics must also be enabled and `frameMetadata: true` must be set:

```ts
debug: {
  avSync: { enabled: true, frameMetadata: true },
}
```

Its purpose is to add correlatable frame metadata to **remote video frames**. When the publisher advertises the relevant packet-trailer feature, the track supports metadata lookup, and an associated TimeSync RTP timestamp is available, the SDK requests the LiveKit frame metadata worker and associates `frameId`, `userTimestamp`, and `userData` with receiver baselines and video-presentation events. This adds bounded `frameMetadata` samples and metadata-bearing TimeSync/receive-side evidence to `getAvSyncRawTelemetry()`, helping investigate the path from receipt of a publisher video frame to its presentation.

When `userData` is an ASCII-decimal publisher Unix capture timestamp in microseconds, the metadata association is fresh, and the frame is not a duplicate, the SDK also fills `videoCaptureToDisplayRawMs` and its relative change for capture-to-display diagnostic evidence. If `userData` is not in that format, it is preserved as-is and the timing fields remain `null`; the SDK never invents a timestamp or precise latency.

This option is **not** required for regular WebRTC receiver stats, periodic sampling, TimeSync, the RTP-based relative-drift timeline, or automatic `video_receive` / `video_decode` / `video_render` alerts. Those capabilities continue when it is disabled; only per-frame metadata and capture-to-display evidence are omitted. Metadata applies only to remote video and depends on browser/worker support, publisher feature advertisement, track lookup, and TimeSync RTP correlation. When any prerequisite is unavailable, telemetry preserves the unavailable or unobserved fact and the associated metadata fields are `null` or empty.

`userData` is retained in bounded diagnostics telemetry as raw Base64, so enable this option only when that payload is suitable for client-side diagnostic data. If the worker fails to initialize or encounters a runtime error, the SDK reports it with the `[RTC frame-metadata-worker]` prefix in Chrome DevTools and retains the worker state as a diagnostic-log fact. The normal LiveKit connection and base A/V diagnostics continue to work, and diagnostics APIs do not reject because of that failure.

### 7.5 Plugin import and configuration

For plugin development, extension points, and runtime semantics, see the [Plugin Integration Guide](./SDK_Plugin_Integration_Guide.md). To use a plugin, import an `SDKPlugin` object from its own package. Replace the placeholders below with the package and export supplied by an actual entry in “Currently Supported Plugin Packages.”

```ts
import { createClient } from '@sanseng/liveavatar-js-sdk';
import { plugin } from '<plugin-package>';

const options = {
  connectConfig: {
    type: 'direct',
    config: { sfuUrl: 'wss://example.test', userToken: 'token' },
  },
} as const;

// Choose one: construction-time installation
const configuredClient = createClient({ ...options, plugins: [plugin] });

// Choose one: runtime installation on an instance without that plugin ID
const runtimeClient = createClient(options);
const result = runtimeClient.installPlugin(plugin);
if (result.status === 'rejected') {
  console.warn(result.pluginId, result.reason, result.message);
}

runtimeClient.uninstallPlugin(plugin.id);
```

Do not include the same plugin ID in both `plugins` and `installPlugin()` on the same instance. Check the runtime installation result. `uninstallPlugin()` returns `true` when it removed an active plugin and `false` when that ID is not active in the instance.

#### Currently Supported Plugin Packages

| Plugin package | Purpose | Status |
| --- | --- | --- |
| None | No standalone plugin package is published or registered for v2.2.0. SDK extension points are not directly installable plugin packages. | No available package |

This table will list package names, installation commands, and links once actual plugin packages are released.

---

## 8. Core API Methods

All methods below are defined in `SDKClient` (`src/client/SDKClient.ts`). Except for the constructor, `setAuthToken`, and `updateConnectionConfig`, most media and session APIs require a **successful `connect()`** (verified internally via `sessionState.isConnected`).

### 8.1 Connection & Lifecycle

| Method | Description |
| :--- | :--- |
| `preConnect(): Promise<boolean>` | Pre-fetches and caches connection config (TTL is ~60s, see `PRECONNECT_CACHE_DURATION_MS`). Throws on failure. |
| `connect(): Promise<void>` | Establishes the Coordinator, Domain Controllers, and LiveKit connection. Calls pre-connect internally if no valid cache exists. |
| `disconnect(): Promise<void>` | Stops capture, disconnects LiveKit, and cleans up HTTP sessions. The instance can still be used for `connect()` or `reconnect()`. |
| `reconnect(): Promise<ConnectionSnapshot>` | Manual reconnection: `disconnect()` is called first, then config is refreshed based on mode before calling `connect()`. Returns a snapshot if reconnection is not allowed. |
| `installPlugin(plugin): SDKPluginInstallResult` | Atomically installs a Plugin API v1 plugin. Rejection is returned as data and does not affect the SDK lifecycle. |
| `uninstallPlugin(pluginId: string): boolean` | Atomically removes contributions, invalidates old handles, and calls plugin dispose. |
| `dispose(): void` | Releases all resources. The instance **must not** be used after this call. |

**Constraint**: Due to browser policies, **audio playback** and **microphone capture** should ideally be triggered by `connect()` / `startAudioCapture()` within a user gesture (e.g., click) callback.

### 8.2 Authentication & Connection Config

| Method | Description |
| :--- | :--- |
| `setAuthToken(token: string): void` | Sets the auth token; used by HTTP and Config components in **Auth Mode**. |
| `updateConnectionConfig(config: DirectConnectionConfig): void` | **Direct Mode only**. Validates and stages `sfuUrl` / `userToken`. **Does not affect the current session**; takes effect during the next `preConnect()` / `connect()` or `reconnect()` via `replaceDirectConfig`. |

### 8.3 Media

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
| `getMicrophoneAudioLevel(): number \| null` | Gets the microphone audio level (0.0-1.0). |
| `getMicrophoneStats(): Promise<MicrophoneStats \| null>` | Gets microphone transmission statistics. |
| `isMicrophoneSilent(): Promise<boolean \| null>` | Detects if the microphone is sending silence. |
| `startCamera(options?: { publishToLiveKit?: boolean }): Promise<void>` | Opens the camera; optionally publishes to LiveKit room. Defaults to local preview only. |
| `stopCamera(): void` | Stops the camera. |
| `getCameraStream(): MediaStream \| null` | Returns the local stream for previewing. |
| `getCameraTrack(): MediaStreamTrack \| null` | Returns the local media track. |
| `attachCameraTo(videoElement: HTMLVideoElement): void` | Binds the local camera stream to a `<video>` element. |
| `detachCameraFrom(videoElement: HTMLVideoElement): void` | Detaches the camera track from a video element. |
| `startAnalyserEmission(): void` | Starts emitting remote audio analyser data (~60fps) via `media:audio:analyserData`. If no remote audio track has arrived yet, the analyser chain is automatically deferred and will begin emitting once the track is subscribed. |
| `stopAnalyserEmission(): void` | Stops remote audio analyser emission and releases AudioContext. |
| `restartAnalyserEmission(): void` | Restarts the remote audio analyser chain. |
| `startLocalAnalyserEmission(): void` | Starts emitting local microphone analyser data via `media:audio:localAnalyserData`. If `startAudioCapture()` has not been called yet, the analyser chain is automatically deferred and will begin emitting once the microphone stream becomes available (auto-retried via 100ms polling, max 10s timeout). |
| `stopLocalAnalyserEmission(): void` | Stops local microphone analyser emission. |
| `restartLocalAnalyserEmission(): void` | Restarts the local microphone analyser chain. |

### 8.4 Session

| Method | Description |
| :--- | :--- |
| `sendTextQuestion(text: string): Promise<string>` | Sends a text query; returns a **Message UID** (used as `questionId` in events for turn correlation). |
| `interrupt(): Promise<void>` | Sends an interruption control event (`control.interrupt`). |

### 8.5 State & Observability

| Member | Description |
| :--- | :--- |
| `get isConnected(): boolean` | Whether the session is currently connected. |
| `get connectionSnapshot(): ConnectionSnapshot` | Synchronous read-only snapshot: `http.connected`, `rtc.connected`, `rtc.hasVideoTrack`, `overall.state`. |
| `get sessionId(): string \| undefined` | Server-assigned session ID. **Auth mode only**; returns `undefined` in direct mode and logs a debug message. |
| `setPerformanceMetricReporter(reporter?: PerformanceMetricReporter): void` | Sets or updates the performance metric reporting callback. |
| `captureRemoteAvSyncDiagnostics(): Promise<AvSyncDiagnosticsSnapshot>` | Actively captures remote audio/video receiver statistics and this connection's track lifecycle nodes; available without debug configuration. |
| `getAvSyncRawTelemetry(query?): Promise<RemoteAvSyncRawTelemetry>` | Reads current bounded context plus one `avSyncLogs` page. The default is the latest five minutes and 200 logs; use `range`, `pageSize`, and `page.nextCursor` to paginate. A fact-only window summary is enabled by default and can be disabled with `includeSummary: false`; log queries and aggregation run in a Dedicated Worker. |
| `getAvSyncAnalysis(query?): Promise<AvSyncAnalysisReport>` | Asynchronously analyzes the complete retained window in the telemetry Worker. The caller controls polling cadence; the report exposes stable `status`, `finding`, `features`, `layers`, and bounded limitations, and ordinary evidence gaps do not reject the promise. Automatic video-stall notification does not depend on calling this method. |
| `get events(): PublicEmitterAPI` | Accesses the event emitter (supports `on`, `off`, `once` only). |

All `getAvSyncAnalysis()` counters are **deltas within the requested analysis window**. Browser WebRTC cumulative receiver counters are never treated as a current anomaly. Important `features` fields are:

- `audioObservationDurationMs`: total observation time across valid remote-audio delta intervals.
- `audioConcealedSamplesDelta` / `audioConcealmentEventsDelta`: audio concealment increments in the window.
- `audioConcealedSamplesPerSecond` / `audioConcealmentEventsPerSecond`: the corresponding increments divided by the observation duration; `null` when no valid audio delta interval exists.
- `videoPacketsLostDelta`, `videoFramesReceivedDelta`, `videoFramesDroppedDelta`, and `videoFramesDecodedDelta`: remote-video increments in the window; received/decoded progression helps distinguish receive/assembly from decode stalls.

For multiple remote participants, the global audio-rate denominator is the sum of valid remote-audio channel observation durations. `participants` and alert participant IDs contain only identities with remote-media evidence in the requested window; local and connection-only identities are excluded.

### 8.6 Version Information

| Member | Description |
| :--- | :--- |
| `SDKClient.version: string` | SDK version string. |
| `VERSION: string` | Directly exported version constant. |

---

## 9. Public Events

Events are subscribed to via `client.events.on(eventName, listener)`. Only the event names in the following whitelist are supported; other names will result in an error.

**Connection**

### `sdk:connected`

- **Trigger**: Dispatched when the aggregated state of internal `livekit` and `http` connections changes and differs from the previous state.
- **Payload**: `{ livekit: boolean; http: boolean; all: boolean; livekitConnected: boolean; httpConnected: boolean; allConnected: boolean }`.
  - `livekitConnected` / `httpConnected` / `allConnected` always reflect the **current** connection state and are symmetric with `sdk:disconnected`. `auth` mode: `allConnected === livekitConnected && httpConnected`; `direct` mode: `allConnected === livekitConnected`.
  - Legacy fields `livekit` / `http` / `all` are **deprecated** (kept for backward compatibility); will be removed in the next major version.
- **Description**: Used for a high-level overview of whether both channels are ready. Recommended for new code to use the new fields.

### `sdk:disconnected`

- **Trigger**: Dispatched when a disconnection in either path causes a change in the aggregated state.
- **Payload**: `{ livekit, http, all, livekitConnected, httpConnected, allConnected, reason? }`.
  - **New fields** `livekitConnected` / `httpConnected` / `allConnected` always reflect the **current** connection state and are symmetric with `sdk:connected`; all three are `false` after a full disconnect. Recommended for new code.
  - **Legacy fields** `livekit` / `http` / `all` are **deprecated** (kept for backward compatibility); on `sdk:disconnected` they carry the inverted semantic (`true` after disconnect). Will be removed in the next major version.
  - `allConnected` (and legacy `all`) are mode-aware: `auth` mode is `livekitConnected && httpConnected`; `direct` mode is `livekitConnected`.
  - `reason` is forwarded from the inner `inner:sdk:disconnected.reason`.
- **Description**: Can be cross-referenced with `connectionSnapshot` for validation. To check "fully disconnected → ready to reconnect", use `payload.allConnected === false`.

### `sdk:error`

- **Trigger**: Dispatched when an internal error is mapped to a secure external payload.
- **Payload**: `{ message: string; code: string }` (`code` is the string value of `ErrorCode`).

### `sdk:pluginFault`

- **Trigger**: An isolated plugin Observer/Transform/Provider/CoreService/dispose failure.
- **Payload**: Only `pluginId`, optional `contributionId` / `{ id, major, kind }` point descriptor, `phase`, `reason`, and `occurredAtMs`. It never exposes Error, stack, tokens, business text, raw protocol payloads, or private plugin data, and is excluded from the Operational Log Observer.

### `sdk:connectionStateChanged`

- **Trigger**: When LiveKit connection state changes (disconnected/connecting/connected/reconnecting).
- **Payload**: `{ state: ConnectionState }`, where `ConnectionState` enum: `disconnected` | `connecting` | `connected` | `reconnecting` | `signalReconnecting`.

### `sdk:avSyncAlert`

- **Trigger**: This is the lifecycle of one A/V incident: `raised` appears on first confirmed attention evidence, `updated` appears when the same active automatic video pipeline is confirmed with another receive/decode/render classification, and `recovered` appears only when direct positive progress is observed at the affected layer. Sustained identical faults are intentionally deduplicated; no new event or log is not recovery. Under `debug.avSync`, a playing video whose RTP packets, received frames, and decoded frames all make no progress for three compatible periodic samples automatically reports `video_receive`; confirmed decode and presentation stalls additionally report `video_decode` / `video_render`. `insufficient_evidence` alone does not raise an alert.
- **Payload**: A bounded `AvSyncAlertEvent` containing `phase`, stable opaque `alertId`, `finding`, `previousFinding`, `status`, `confidence`, `sessionId`, `capturedAtMs`, participant IDs, an English message, and compact boolean features. `previousFinding` is null for raised and identifies the preceding abnormal classification for updated/recovered. Participant IDs identify only remote media participants that produced the finding; raw logs, media objects, local identities, and connection-only identities are never exposed. All phases go only to the public event and plugin handlers; only the first raised alert creates the retained stall log, and the SDK does not upload externally.

### `sdk:connectionQualityChanged`

- **Trigger**: When participant connection quality changes (driven by LiveKit RTC layer).
- **Payload**: `{ quality: ConnectionQuality; participantId: string; isLocal: boolean }`, where `ConnectionQuality` enum: `excellent` | `good` | `poor` | `lost` | `unknown`.

### `sdk:participant:disconnected`

- **Trigger**: When a remote participant disconnects from the LiveKit room.
- **Payload**: `{ participantId: string }`.

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
| `media:audio:trackAdded` | Remote participant's audio track is subscribed. | `undefined` |
| `media:audio:trackRemoved` | Remote participant's audio track is unsubscribed. | `undefined` |
| `media:audio:speakingChanged` | Remote participant's speaking state changed. | `{ participantId: string; isSpeaking: boolean }` |
| `media:audio:analyserData` | Remote audio analyser data ready at ~60fps. | `{ frequencyData: Uint8Array; timeDomainData: Uint8Array }` |

**Local Microphone**

| Event | Trigger | Payload |
| :--- | :--- | :--- |
| `media:audio:captureStarted` | Local input capture started. | `undefined` |
| `media:audio:captureStopped` | Local input capture stopped. | `undefined` |
| `media:audio:frameData` | Raw audio frame data with sample rate. | `{ data: Float32Array; sampleRate: number }` |

**Local Microphone Analyser**

| Event | Trigger | Payload |
| :--- | :--- | :--- |
| `media:audio:localAnalyserData` | Local microphone analyser data at ~60fps. | `{ frequencyData: Uint8Array; timeDomainData: Uint8Array }` |

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
| `conversation:server:message` | Server-side message. | `{ questionId; rawText; representations; type }`; deprecated `message === rawText` |
| `conversation:asr:received` | ASR final result received. | `{ questionId; rawText; representations }`; deprecated `text === rawText` |
| `conversation:asr:chunk` | ASR text chunk received. | `{ questionId; rawText; representations; isComplete }`; deprecated `text === rawText` |
| `conversation:answer:chunk` | Answer text chunk received. | `{ questionId; rawDelta; rawText; representations; isComplete }`; deprecated `chunk === rawDelta` |
| `conversation:answer:completed` | Single turn answer completed. | `{ questionId; rawText; representations }`; deprecated `fullAnswer === rawText` |

`representations` contains JSON-safe projections ordered by successful plugin installation and setup registration. Equal `mediaType` values do not overwrite each other. New code should use only `rawDelta` / `rawText`. Legacy fields remain in 2.2.0 without runtime warnings and can be removed only in a future major release.

**Session**

| Event | Trigger | Payload |
| :--- | :--- | :--- |
| `session:closing` | Server-initiated session closure. | `Record<string, unknown>` |

---

## 10. Full Usage Example

```ts
import { createClient, type PerformanceMetricRecord, VERSION } from '@sanseng/liveavatar-js-sdk';

console.log('SDK version:', VERSION);

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

  client.events.on('conversation:answer:chunk', ({ questionId, rawDelta, rawText }) => {
    const prev = answerByQuestion.get(questionId) ?? '';
    answerByQuestion.set(questionId, prev + rawDelta);
    document.getElementById('answer')!.textContent = rawText;
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

## 11. Video Chroma Key (Green Screen) Debugging Guide

Before enabling Chroma Key, ensure the following settings are applied (or leave them unconfigured for the SDK's internal auto-detection):

- `video.renderMode = 'processed'`
- `greenScreen.enabled = true`

### 11.1 Tuning Recommendations

1. **Background Color Selection (`chromaKey`)**
   - Recommended to pick colors from actual video screenshots.
   - Avoid using generic pure green (#00FF00).
   - Only green hues are supported.
   - Default: `[0, 255, 0]` (pure green).

2. **Similarity (`similarity`)**
   - Default: `0.4`
   - Start from `0.3` and adjust incrementally.
   - Values too high may erroneously remove person details (e.g., hair or clothing).

3. **Green Spill Suppression (`despillStrength`)**
   - Default: `1.15`
   - Used to reduce green reflections/fringes on the subject's edges.

4. **Edge Smoothness (`smoothness`)**
   - Default: `0.25`
   - Improves blending for hair and semi-transparent areas.

---

## 12. FAQ

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

## 13. Error Codes

The following are the string values for the `ErrorCode` enum (`src/errors/ErrorCodes.ts`), consistent with the `code` field in `sdk:error` and thrown `SDKError` objects.

### 13.1 HTTP & Control Plane

| Code | Description & Recommendation |
| :--- | :--- |
| `SDK_AUTH_TOKEN_FAILED` | Auth HTTP request failed. Check token and gateway URL. |
| `SDK_GET_LIVEKIT_CONFIG_FAILED` | Failed to fetch room config. Ensure backend returns `livekitUrl` / `roomToken`. |
| `SDK_SWITCH_VIDEO_FAILED` | Control logic for switching video failed. Check logs for details. |
| `SDK_INTERRUPT_CONVERSATION_FAILED` | Failed to send interruption command. |
| `HTTP_CONTROLLER_NOT_AVAILABLE` | HTTP controller is not ready. Avoid calling HTTP-dependent operations. |

### 13.2 SDK Lifecycle

| Code | Description & Recommendation |
| :--- | :--- |
| `SDK_PRECONNECT_FAILED` | Pre-connection failed. Check network, Direct params, or Auth token. |
| `SDK_CONNECT_FAILED` | Connection failed. Check preceding error messages. |
| `SDK_INITIALIZATION_FAILED` | Failed during initialization. Check console stack trace. |
| `SDK_DISCONNECT_FAILED` | Error during disconnect. Log the error and consider creating a new instance. |
| `SDK_RECONNECT_FAILED` | Manual reconnection failed. |
| `SDK_NOT_CONNECTED` | API called without an active connection. Call `connect()` or check `isConnected`. |
| `SDK_INVALID_STATE_TRANSITION` | Illegal state transition (e.g., calling `updateConnectionConfig` in Auth mode). |
| `SDK_PLUGIN_CONTEXT_INACTIVE` | A retained registrar/CoreService proxy was used after unload or SDK disposal; stop using that handle. |
| `SDK_ERROR` | Generic fallback error code. |

### 13.3 LiveKit / RTC

| Code | Description & Recommendation |
| :--- | :--- |
| `LIVEKIT_CONNECT_FAILED` | Room connection failed. Verify URL/token/TURN network status. |
| `LIVEKIT_SEND_VIDEO_AVAILABLE_STATE_FAILED` | Failed to report video available state. |
| `LIVEKIT_SEND_TEXT_DATA_FAILED` | Failed to send text data via LiveKit Data Channel. |
| `LIVEKIT_DATA_MESSAGE_PARSE_ERROR` | Failed to parse data message. Check protocol version compatibility. |
| `LIVEKIT_UNPUBLISH_MICROPHONE_FAILED` | Failed to unpublish the microphone track. |

### 13.4 Audio

| Code | Description & Recommendation |
| :--- | :--- |
| `AUDIO_CAPTURE_START_FAILED` | Mic start failed. Check user gestures, HTTPS, and permissions. |
| `AUDIO_CAPTURE_FAILED` | Capture interrupted. |
| `AUDIO_INVALID_SAMPLE_RATE` / `AUDIO_INVALID_CHANNEL` / `AUDIO_INVALID_CODEC` | Parameter mismatch with device or protocol. |
| `AUDIO_CONTROLLER_NOT_AVAILABLE` | Audio controller not created or already released. |
| `AUDIO_OUTPUT_DISABLED` | Thrown by `getAudioElement()` when `output.enabled === false`. |

### 13.5 Camera

| Code | Description |
| :--- | :--- |
| `CAMERA_CONTROLLER_NOT_AVAILABLE` | Camera controller is unavailable. |

### 13.6 Session & State Machine

| Code | Description & Recommendation |
| :--- | :--- |
| `CONVERSATION_CONTROLLER_NOT_AVAILABLE` | Conversation controller is unavailable. |
| `STATE_MACHINE_INVALID_STATE_TRANSITION` | Internal state machine received illegal transition. Provide logs for feedback. |

### 13.7 Utilities & Others

| Code | Description |
| :--- | :--- |
| `OBJECT_DISPOSED` | Instance has been `dispose()`-ed. |
| `NO_AVATARID` | Missing `avatarId` in Auth configuration. |
| `NO_AUTH_TOKEN` | Missing authentication token (validated at runtime). |
| `INVALID_CONNECT_CONFIG` | Invalid Direct configuration during `createClient`. |

---

## 14. Performance Monitoring & Troubleshooting

### 14.1 Built-in Metrics

Enabled by default (`performanceMonitor.enabled !== false`). The metric names under `PerformanceMetricName` include:

| `metric` | Meaning |
| :--- | :--- |
| `connect_to_first_frame_ms` | Time from `connect()` call to completion of first frame rendering. |
| `text_send_to_text_response_ms` | Time from sending text query to receiving the first text response chunk. |
| `text_send_to_audio_response_ms` | Time from sending text query to receiving the first audio response chunk. |
| `no_speech_report_to_audio_response_ms` | Time from "no-speech" report to receiving the audio response. |

Data structure: `PerformanceMetricRecord` (contains `metric`, `durationMs`, `startedAt`, `endedAt`, and optional `questionId`).

### 14.2 Custom Reporting

```ts
import { createClient, type PerformanceMetricRecord } from '@sanseng/liveavatar-js-sdk';

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

### 14.3 Troubleshooting Recommendations

- **Slow First Frame**: Check region latency, TURN server status, and whether repeated `connect()` retries are occurring.
- **Slow Text Response**: Correlate with backend LLM/business processing time and check for main-thread blocking.
- **Audio Significantly Slower Than Text**: Inspect the TTS pipeline and remote audio track subscription states.

---

_Document version consistent with package version: 2.2.0._
