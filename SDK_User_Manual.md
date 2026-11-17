# Facemarket Live Avatar SDK User Manual (v2.8.0)

This manual corresponds to the npm package **`@sanseng/liveavatar-js-sdk` version 2.8.0**. The SDK is built on **LiveKit Client** and encapsulates live avatar audio/video downlink, microphone/camera uplink, session text, and the HTTP control plane (for fetching connection configurations in Auth mode).

---

## 1. Overview

The SDK provides a unified entry point via **`createClient` → `SDKClient`**, which is responsible for:

- Subscription and routing of audio/video tracks, as well as local capture and publishing.
- Encoding, decoding, and distribution of session-side text and protocol messages.
- Connection lifecycle management and state snapshots (for easier reconnection and troubleshooting).
- Optional chroma key (green screen) processing and performance metric reporting.

Integrators can complete the setup via constructor parameters, connection APIs, and event subscriptions without needing to understand internal SDK modules (e.g., `transport` implementation details).

---

## 2. Upgrade Notes

For a migration from 1.x to **2.8.0**, follow the [SDK 1.x → 2.8.0 Upgrade Guide](./SDK_v1_to_v2_Upgrade_Guide.md) for connection, event, microphone, and performance-reporting changes. It uses v1.3.2/v1.3.3 as the main baseline and lists additional checks for earlier 1.x versions. Historical verification remains limited to v1.3.2/v1.3.3 → v2.3.0; coverage through 2.8.0 does not establish real integration acceptance for every path. Version 2.8.0 is currently unreleased.

### Upgrading from 2.7.x to 2.8.0

- `startAudioCapture()` only waits for microphone enablement and LiveKit publication; it no longer automatically creates an SDK-owned sampling graph. Explicitly await `startMicLevel()` before reading with synchronous, read-only `getMicLevel()`, and call `stopMicLevel()` when finished. The compatible `getMicrophoneAudioLevel()` alias also requires explicit monitoring startup; it is deprecated for removal in 3.x. Drive the icon from startup and `isAudioCapturing`, independently of level availability.
- The first read after successful `startMicLevel()` returns real RMS, including silence (`0`); disabled monitoring, no active microphone, preparation, or unavailability returns `null`. Sampling failure does not stop publication. Handle cancellation (`AbortError`) and nonfatal `AUDIO_INPUT_MONITORING_UNAVAILABLE` independently. Monitoring may register early, but do not await it before triggering capture.
- Capture stop, disconnect, and disposal immediately revoke old startup authority. Cancelled startup can resolve without enabling capture; check current state when operations race. The SDK reclaims late tracks and isolates old-resource cleanup; sampling recovery never reacquires a microphone or switches devices. See the local microphone sections.
- Browser permission failures classified as `permission` from structured evidence use `warn` instead of `error` in operational logs; public errors and Promise rejections remain unchanged. Error-only consumers must include warn; see the logging section.

### Upgrading from 2.6.x to 2.7.0

- Direct adds optional `sessionId`. After configuration is first successfully resolved/applied, `client.sessionId` changes from the previous `undefined` to a caller ID or generated ID with an `sdk-` prefix. Update application checks that assumed the getter was always empty. Staging through `updateConnectionConfig()` alone does not change the active value; see Section 5.2.
- Logging V3, performance V2, AV Sync Observer V2, and diagnostics CoreService V3 are optional new extension points requiring SDK **2.7.0** or later. Plugin API remains **1**. Older extension points remain available; select one version of a fact stream to avoid duplicate uploads. SDK upgrades do not automatically migrate plugins, collector schemas, or queries.

### Upgrading from 2.5.x to 2.6.0

- `performanceMonitor.reporter` and `setPerformanceMetricReporter()` remain accepted but no longer invoke callbacks; default `[performance-metric]` Console output is also removed. Migrate existing uploads to the `SDK_PERFORMANCE_METRIC_V1` plugin in Section 14.2. The legacy signatures will be removed in 3.0.0.
- Console levels are scoped to each Client; creating or disposing another Client does not override an existing Client's `debug` setting. Logging V2 and pre-disposal Observer handoff are optional; see Sections 7.5–7.6 and the Plugin Integration Guide.

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

- **`preConnect()`** resolves and caches the current or staged Direct configuration without establishing a LiveKit connection. `connect()` uses valid cached configuration or resolves it before connecting.
- **`reconnect()`**: Disconnects the current session and reconnects using the latest staged Direct configuration, without fetching room configuration through Auth HTTP.
- **Token Refresh**: The SDK does not automatically "refresh" server-issued tokens. Token expiry must be handled by the business side by fetching a new token, calling `updateConnectionConfig`, and then triggering `reconnect()`.

**Usage Restrictions**

- Calling `updateConnectionConfig` while in Auth mode is unsupported (will throw `SDK_INVALID_STATE_TRANSITION`).
- Direct mode does not support `videoOptions` override strategies returned via HTTP interfaces. Video parameters such as green screen rely on the `video` options provided at construction or runtime methods like `setRenderFitMode` (unless the business implements its own local configuration sync).

**Use Cases** Existing backends that stably issue LiveKit tokens, private cloud deployments, or rapid integration during the debugging phase.

---

### 2.2 Auth Mode

**Description** The caller provides an `avatarId` (and optionally `authToken` or `avatarVoice`). The SDK fetches authentication and room configuration over HTTP and applies server-returned `videoOptions`, such as green-screen settings.

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

- **`preConnect()`** fetches and caches connection configuration over HTTP without establishing a LiveKit connection. `connect()` uses valid cached configuration or fetches it before connecting.
- **`reconnect()`**: Disconnects and **fetches the latest room configuration over HTTP**.
- **`setAuthToken()`** updates the token for subsequent authentication and configuration requests but does not invalidate cached room configuration. If fresh configuration is required immediately and `reconnect()` is disallowed, dispose the old instance and create a new one with the new token.

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
| **`reconnect()` Refresh** | Uses the latest staged Direct configuration. | Fetches the latest configuration over HTTP. |

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

**Caution**: SDK-generated `questionId` values are short random strings without session-level collision checking. They can collide and must not be treated as unique or as persistent primary keys; use application-owned identifiers and account for correlation collisions.

**Strong Recommendation**: Upon receiving `conversation:answer:completed`, clear the local buffers (text concatenation, UI states, etc.) associated with that `questionId` to prevent memory growth and display errors caused by ID reuse.

### 5.2 Session correlation (Since 2.7.0)

Optional Direct `sessionId` and the unified-session extension points below are available since SDK **2.7.0**. Plugins using the new points must require SDK **2.7.0** or later. Plugin API remains **1**.

Direct `connectConfig.config` and `updateConnectionConfig()` accept `sessionId?: string`. Strings are trimmed. Omitted, `undefined`, empty, whitespace-only, or runtime `null` values select an SDK-generated ID with an `sdk-` prefix. The suffix uses `crypto.randomUUID()` when available, with a timestamp and random-string fallback otherwise. Treat it as an opaque string rather than parsing its suffix format. Other non-string values fail with `SDK_PRECONNECT_FAILED`. This is a business correlation ID, not a credential; Direct never populates the Auth-only `serverSessionId`.

```ts
const client = createClient({
  connectConfig: {
    type: 'direct',
    config: { sfuUrl, userToken, sessionId: 'business-session-42' },
  },
});
// Undefined until configuration is successfully resolved/applied; RTC need not connect first.
await client.preConnect();
console.log(client.sessionId); // 'business-session-42'
client.updateConnectionConfig({ sfuUrl, userToken, sessionId: 'next-session' });
// Staging keeps the getter and active connection unchanged until configuration is applied.
```

New outputs use `SDKSessionCorrelation`: `{ clientInstanceId, sessionId, sessionEpoch }`. Within one Client, epochs start at **1**, increase for new lifecycles, and are never reused. Caller-supplied `sessionId` values may repeat. Use `clientInstanceId + sessionEpoch` to isolate lifecycles and retain `sessionId` for business joins; a session ID alone is not a persistence uniqueness key. `attemptId` still identifies only one first-frame measurement.

- Direct compares normalized `sfuUrl`, `userToken`, and caller `sessionId`. Unchanged configuration, cache hits, refresh, and internal reconnect retain the lifecycle. Changing only the ID also starts a new epoch when applied; staging alone does not.
- Explicit `disconnect()` ends the lifecycle. Preparing configuration again starts a new epoch and generates a new automatic ID; a caller ID may repeat. Passive disconnect and internal reconnect teardown do not independently end it.
- Auth starts a real configuration fetch with a provisional ID prefixed with `sdk-`, using the same UUID-first and fallback rules. Retries and shared in-flight requests retain that epoch. A successful server ID binds within the same epoch; only Log V3 emits `session.id-bound` with the new ID at top-level `sessionId` and only `previousSessionId` in `data`. If the server omits its ID, new outputs retain the local ID. Auth `client.sessionId` keeps its existing server-ID getter contract.
- Asynchronous work and media resources capture their owning epoch. Delayed results, retained telemetry, and historical fields never borrow attribution from the current session.

Migrate in phases: use `SDK_OPERATIONAL_LOGS_V3` (`sdk.log`, schemaVersion 3), `SDK_PERFORMANCE_METRIC_V2` or `SDK_PERFORMANCE_METRICS_V2` (batch `sdk.performance-metrics`, version 2), and `SDK_AVSYNC_ALERTS_V2`, `SDK_AVSYNC_ANALYSES_V2`, `SDK_AVSYNC_SESSION_ENDED_V2`, and `SDK_AVSYNC_DIAGNOSTICS_V3`. New contracts use application `sessionId` plus `sessionEpoch`, name the media diagnostics ID `avsyncSessionId`, and omit `monitoringSessionId`. See the plugin guide for registration examples.

Log V1/V2, performance V1, old AV Sync extension points, direct Client diagnostics methods, and public events retain their fields, schemas, timing, and meanings. `monitoringSessionId` still joins legacy outputs; legacy AV Sync `sessionId` remains a media diagnostics ID. Log V3 uses top-level application `sessionId` for the local/bound ID and separate media `avsyncSessionId`; it does not duplicate Auth `serverSessionId`. Plugin API remains **1**.

The current SkyWalking plugin **1.1.0** consumes only old contracts. Keep its matching old extension points until a separate plugin migration; an SDK upgrade does not migrate the plugin or collector. Migrate the plugin, collector schema, and queries before the planned **3.0** removal of legacy `monitoringSessionId` outputs. That removal is not part of this change.

### 5.3 Control Plane and Media Plane

- **Media Plane**: Connects to LiveKit rooms, subscribes to and publishes audio/video, and transmits session text and protocol messages through the **Data Channel**.
- **Control Plane (Auth)**: Fetches over HTTP the `livekitUrl`, `token`, `roomId`, and optional server-side `videoOptions` (such as green screen toggles) in Auth Mode. There is **no** independent "Business WebSocket" channel parallel to LiveKit exposed in the external API.

### 5.4 Video Rendering Modes

- `video.renderMode === 'raw'`: Remote tracks are rendered directly onto the `<video>` element (default behavior depends on options).
- `video.renderMode === 'processed'`: Renders via a Canvas processing pipeline (e.g., for green screen/chroma key).
- **In Auth Mode**, server green-screen configuration may be mapped to `renderMode` / `greenScreen`.

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
| `config.sessionId` | `string` | No; since 2.7.0, trimmed, automatically generated when omitted |

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
| `greenScreen` | `{ enabled, chromaKey?, similarity?, smoothness?, despillStrength?, isBackgroundKeying? }`; `isBackgroundKeying` defaults to `false`. |
| `camera` | `{ publishToLiveKit?: boolean }` | No | Default `false`. When `true`, `startCamera()` publishes the track to the LiveKit room. Can be overridden per-call via `startCamera({ publishToLiveKit: true })`. |
| `fitMode` | `'contain' \| 'cover' \| 'fill' \| 'none'` |
| `debug` | Inherited from `BaseOptions`. |

### 6.4 `audio` (`AudioOptions`)

| Field | Description |
| :--- | :--- |
| `input` | `{ deviceId?: string, sampleRate?: number, channelCount?: number, sampleSize?: number, noiseSuppression?: boolean, voiceIsolation?: boolean, bitDepth?: number, echoCancellation?: boolean, autoGainControl?: boolean, constraints?: MediaTrackConstraints }` |
| `output` | `{ enabled?: boolean, volume?: number, muted?: boolean }` Default values for the playback side. |

**Input precedence**: explicit top-level input fields override the same member in `constraints`; other standard constraint members are preserved and passed to `getUserMedia`. `sampleSize` is passed to the capture request, `bitDepth` is a deprecated alias used only when `sampleSize` is omitted, and `constraints.sampleSize` is the final fallback. `noiseSuppression`, `echoCancellation`, and `autoGainControl` default to `true` only when neither the top-level field nor `constraints` sets them, so an explicit `false` is not overwritten. Local PCM processing uses `channelCount` `1` when no numeric value is available.

A successful capture request does not prove the device accepted every constraint. Read the actual browser-reported values from `getMicrophoneStats().settings`.

### 6.5 `performanceMonitor`

| Field | Description |
| :--- | :--- |
| `enabled` | Defaults to `true`; set to `false` to disable built-in performance collection. |
| `reporter` | `(metric: PerformanceMetricRecord) => void`. **Inert: configuration is ignored, callbacks are never invoked; removed in 3.0.0.** |

### 6.6 `MicrophoneStats`

| Field | Description |
| :--- | :--- |
| `bytesSent` | Number of bytes sent. |
| `packetsSent` | Number of packets sent. |
| `packetsLost` | Number of lost packets. |
| `roundTripTime` | **Legacy:** raw LiveKit outbound RTP round-trip value in seconds, retained for compatibility. Unavailable values are `0`. Use `roundTripTimeMs` for a defined unit. |
| `roundTripTimeMs` | Round-trip time in milliseconds, converted from the sender-stat seconds value. `null` when unavailable. |
| `availability` | Per-field evidence flags: `bytesSent`, `packetsSent`, `packetsLost`, and `roundTripTimeMs`. A legacy `0` with `false` means unavailable; a `0` with `true` is a real zero. |
| `settings` | Actual `MediaStreamTrack.getSettings()` values normalized to `MicrophoneTrackSettings`, or `null` when the browser cannot report them. |
| `sampledAtMs` | Query timestamp from `performance.now()`. Each call re-reads sender stats and track settings; nothing is cached. |

`MicrophoneTrackSettings` contains nullable `deviceId`, `groupId`, `sampleRate`, `channelCount`, `sampleSize`, `echoCancellation`, `noiseSuppression`, `autoGainControl`, and `voiceIsolation` values. `getMicrophoneStats()` still returns `null` when there is no active microphone, no sender report, or the sender-stat request fails; a settings read failure alone does not suppress statistics.

---

## 7. Debug and A/V Diagnostics

### 7.1 Basic Debug and LiveKit Verbose

Ordinary SDK Console logs are scoped to each Client: `debug` omitted or `false` defaults to `ERROR`, while `true` or an object (including `{}`) defaults to `DEBUG`. Creating or disposing another Client does not change that level.

Explicit `Logger.setLevel()` calls immediately affect Clients already created; later Clients start with their own `debug` setting. `Logger.setPrefix()` sets a shared SDK Console prefix.

`debug: { verbose: true }` enables LiveKit Debug logs. Within one SDK module copy, LiveKit stays at `debug` while any undisposed Client requests verbose, otherwise it uses `info`. Disconnect/reconnect retains that setting and `dispose()` releases it. LiveKit levels are global and may affect other LiveKit instances in the same page; separate SDK copies do not coordinate.

### 7.2 Periodic A/V Diagnostics (`debug.avSync`)

Use the following configuration to sample remote audio/video receiver statistics after the LiveKit Room connects; no polling runs by default:

```ts
debug: {
  avSync: { enabled: true, intervalMs: 2000, historyLimit: 60 },
}
```

Background sampling waits up to 10 seconds for the first remote audio or video track accepted by the renderer-participant filter. On timeout, it stops for the current connection session and retains `remote-track-wait-timeout` for upload. This timeout does not affect manual `captureRemoteAvSyncDiagnostics()`.

Offline evidence is returned by pages from `getAvSyncRawTelemetry(query?)`, not `[RTC remote-media]` Console output: the no-argument call returns at most the latest five minutes and 200 logs, while `range`, `pageSize`, and `page.nextCursor` retrieve further records. When the telemetry Worker is available, log queries and fact-only summaries run in a Dedicated Worker; if no Worker exists at creation, they use the memory fallback. Manual capture still reads current WebRTC stats on the browser main thread. `avSyncLogRetention` reports retained counts, storage errors, and drops. Room disconnect freezes the final session for paged queries; reconnect replaces it, while `dispose()` and failed connection attempts clear it.

Pagination contract: time endpoints are inclusive. `recent.durationMs` must be positive and finite; wall-clock endpoints must be finite with `fromMs <= toMs`. `pageSize` must be an integer from 1 to 1000 (default 200). Invalid parameters, empty/malformed cursors, cross-session cursors, and cursor/range conflicts reject with `SDKError`, code `RTC_INVALID_AV_SYNC_TELEMETRY_QUERY`. Parameters are validated even without a current or frozen session.

The first-page cursor fixes the session, range and sequence ceiling; continuation may change pageSize. `hasMore: true` always includes a usable `nextCursor`; the final page returns false/null. `summary` aggregates all available logs in the complete query window under that ceiling, independently of the current page. `includeSummary: false` returns null and skips aggregation. `currentDiagnosis` is the current call's diagnosis context. `availableFrom/ToWallClockMs` describe retained session evidence under the ceiling; `truncated` reflects retention eviction overlapping the window, not ordinary paging. Later logs cannot enter old-cursor pages or summaries, but retention eviction can reduce evidence on later pages; the SDK does not retain a server-style window snapshot.

The telemetry Worker and fallback when no Worker exists at creation share this pagination algorithm. LocalStore owns the retention budget and eviction of sealed segments; the Worker queries an incremental memory mirror of retained segments and the open tail, without waiting for persistence or rereading IndexedDB. Evidence with pending or failed writes can remain in the mirror; storage status reflects what was known when the request was captured and does not imply pending writes succeeded. Fallback queries only the memory evidence it actually owns, without equivalent persistent historical coverage. Runtime raw telemetry or evidence query failures propagate and do not trigger automatic fallback; analysis returns `insufficient_evidence` when the analysis Worker is unavailable or mirror evidence is incomplete. This telemetry Worker is separate from the experimental frame metadata worker in Section 7.4.

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

#### Observer Handoff During Disposal

Since **2.6.0**, plugins can set `terminalObserverDelivery: 'sync-before-dispose'` to synchronously receive pending runnable Observer callbacks before Client disposal, including session-end logs already produced. Applications keep calling `client.dispose()`; no extra handling is needed without a plugin.

The SDK does not await uploads or persistence. A plugin must receive/enqueue data before the callback's first `await`, then use its own upload resources from `dispose()` for subsequent work; it cannot rely on disposed SDK services. This option does not change normal callback delivery or explicit `uninstallPlugin()` behavior and does not guarantee successful upload after page exit.

Plugins using this capability must require SDK **2.6.0** or later as their peer dependency. Plugin API stays at `1`. See the [Plugin Integration Guide](./SDK_Plugin_Integration_Guide.md#51-terminal-observer-handoff) for the full contract and example.

#### Currently Supported Plugin Packages

| Plugin package | Purpose | Status |
| --- | --- | --- |
| `@sanseng/liveavatar-js-sdk-markdownrender` | Markdown rendering; see the plugin package documentation for exports, configuration and compatibility. | Listed plugin package |
| `@sanseng/liveavatar-js-sdk-skywalking` | SkyWalking integration; reviewed 1.1.0 uses legacy SDK contracts. See the plugin documentation for upload coverage. | Separate migration required for unified-session contracts |

SkyWalking 1.1.0 does not consume the unified-session contracts added in 2.7.0. Migrate the plugin and collector separately. The SDK does not own plugin authorization, queueing, retries, or delivery.

---

### 7.6 Operational Logs and Diagnostic Evidence

The SDK does not upload logs automatically. Applications use plugins to receive and send records to their own platforms; upgrading the SDK does not automatically change an existing plugin's uploads.

| Plugin capability | Purpose | Minimum SDK version |
| --- | --- | --- |
| `SDK_OPERATIONAL_LOGS_V1` | Continue receiving existing operational logs | Existing plugin compatibility range |
| `SDK_OPERATIONAL_LOGS_V2` | Receive `SDKLogRecordV2` with operation outcomes, error reasons, and session correlation | `2.6.0` |
| `SDK_AVSYNC_DIAGNOSTICS_V2` | Read retained A/V evidence by page using `getAvSyncLogEvidence(query?)` | `2.6.0` |
| `SDK_OPERATIONAL_LOGS_V3` | Log V3 with unified application session correlation (Section 5.2) | `2.7.0` |
| `SDK_PERFORMANCE_METRIC_V2` / `SDK_PERFORMANCE_METRICS_V2` | Individual/batch performance measurements with application session ownership | `2.7.0` |
| `SDK_AVSYNC_ALERTS_V2` / `SDK_AVSYNC_ANALYSES_V2` / `SDK_AVSYNC_SESSION_ENDED_V2` | A/V Observers separating application and media diagnostics identities | `2.7.0` |
| `SDK_AVSYNC_DIAGNOSTICS_V3` | A/V snapshots, telemetry, analysis, and log evidence with unified correlation | `2.7.0` |

V1 remains available. Choose one real-time log version for the same purpose to avoid duplicate uploads. In V2, use `clientInstanceId + monitoringSessionId` to correlate monitoring sessions. `serverSessionId` identifies the server session and `avsyncSessionId` identifies the A/V diagnostics session; keep them distinct.

Plugins call `getAvSyncLogEvidence()` on the service obtained through `registrar.consume(SDK_AVSYNC_DIAGNOSTICS_V2)`. It does not start extra sampling; enable `debug.avSync` in advance for continuous evidence. The first query returns retained evidence without warm-up or repeated calls. Query parameters and pagination match `getAvSyncRawTelemetry()`. See the [Plugin Integration Guide](./SDK_Plugin_Integration_Guide.md#41b-structured-logs-and-av-sync-evidence-v2) for an example.

#### Compact Log V3 records

Cross-instance diagnostic IDs use unpadded Base64URL encoding: `clientInstanceId` is `c_` plus 22 characters, `monitoringSessionId` is `m_` plus 22, and the local-evidence page lifecycle ID is `p_` plus 22. The SDK first encodes all 16 bytes of `crypto.randomUUID()`; if unavailable or failing, it obtains 16 bytes from `crypto.getRandomValues()`. It neither truncates nor hashes the UUID, retaining its random bits and collision resistance. If both cryptographic sources are unavailable, a type prefix, base-36 time, module-local counter and random string provide a compatibility fallback; that path has no fixed 24-character length or cryptographic uniqueness guarantee. Random IDs do not promise mathematically impossible collisions. Treat all these IDs as opaque strings rather than parsing their prefix or length.

Short cross-instance IDs apply across log, performance and diagnostic versions, so derived IDs containing the Client ID also become shorter. Existing fields, composition rules and uniqueness scopes remain, and historical records are not rewritten. Caller, backend, browser and LiveKit IDs, and the automatic application `sessionId` rules prefixed with `sdk-`, are unchanged. `failureId` and `alertId` retain their full Client-qualified identity for independent consumers; do not derive them from short operation IDs.

Log V3 retains `schema: 'sdk.log'`, `schemaVersion: 3`, `SDK_OPERATIONAL_LOGS_V3` major 3. This format was introduced in SDK 2.7.0; the current SDK is 2.8.0; legacy V1/V2 fields and meanings stay intact.

Since 2.8.0, browser failures classified as `permission` from structured `NotAllowedError` / `SecurityError` evidence use `warn` instead of `error` in V1 `sdk.error` / `operation.failed` and V2/V3 `sdk.failure`; existing lower levels remain. Permission unavailability does not establish user refusal. HTTP 401/403, LiveKit `NotAllowed`, other backend/device failures and unknown errors retain their rules, as do precondition and confirmed-cancellation classification precedence. Failure records, codes, operation outcomes, public `sdk:error` events and Promise rejections remain; error-only consumers must accept warn or count failure events.

Each record contains `clientInstanceId`, `stream: 'operational' | 'avsync'`, `sequence`, Unix wall-clock `occurredAtMs`, `level`, `module`, and `event`; optional fields are `sessionId`, `sessionEpoch`, `avsyncSessionId`, `observedAtMs`, `trigger`, `operation`, top-level `failureId`, `error`, `correlation`, `snapshot`, and `data`. V3 omits top-level `eventId`, `streamId`, `clockId`, `serverSessionId`, and `failureRef`, and does not repeat `failureId` inside `error`. Internal Auth server identity and the Auth getter retain their behavior.

- Deduplicate operational records by `[clientInstanceId, 'operational', sequence]`; the sequence spans the Client lifetime, including separate `sdk.failure` records. Deduplicate A/V evidence by `[clientInstanceId, 'avsync', avsyncSessionId ?? null, sequence]`, retaining each media diagnostic source's sequence. Order within one space, never across spaces. Application `sessionId`/`sessionEpoch` are excluded from record identity, so Auth binding does not relabel records. Client plus epoch still isolates lifecycle ownership.
- `occurredAtMs` is the fact's Unix wall-clock time in milliseconds. When available, `observedAtMs` is milliseconds from the main Client's monotonic clock: operational records read it at publication, including failures rather than reusing failure capture time; A/V records retain the source observation. The clock origin is implicit in the contract. Do not subtract different clock domains or interpret publication intervals as business duration; use existing measurement fields for durations.
- One source failure has one `sdk.failure` containing complete `error` details. Its top-level `failureId` uses the same source identity as V2. Related operation/flow results carry that same top-level `failureId`, without eventId joins or repeated error details. Optional `error.operation`, `error.phase`, and `error.protocolEvent` retain failure-site information that operation does not express.
- `snapshot` contains connection state only: SDK state, HTTP/RTC flags, Room presence/state, whether it connected, and last disconnect. It does not repeat source, time, identity, or cause; error evidence belongs in `error.evidence`.
- V3 projects SDK-generated operations to decimal ordinals such as `o1`, `o2` and `o10`, consistently across `operationId`, `rootOperationId`, `parentOperationId` and both `relatedOperationId` locations. The complete operation join key is `[clientInstanceId, operationId]`; separate Clients can each emit `o1`. Source ordinals span the Client lifetime and do not reset on reconnect, configuration changes, Auth binding or plugin uninstall/reinstall. Internal source IDs and V1/V2 operation composition remain unchanged. Only SDK operation IDs matching the record's Client exactly are converted; other values are preserved. Operation joins do not replace the record deduplication keys above.
- `operation.phase` is optional: omit started/finished or an event-name duplicate, while retaining real request/response/init phases. A root operation omits its own `rootOperationId`; children retain the root ID. If operation and correlation have the same `relatedOperationId`, keep only correlation; preserve both when they differ.
- `trigger` is optional: omit unknown/default SDK event entries; retain real API, backend, browser, and independently meaningful SDK entries. `correlation.mediaStreamId` identifies a browser MediaStream, replacing the old V2 correlation name `streamId`.
- Auth `session.id-bound` carries the new top-level `sessionId` and only `previousSessionId` in `data`. Earlier facts are not rewritten. Evidence pages remain `sdk.log-evidence`, version 3, with compact V3 `records`; windows, cursors, first-query behavior, measurement names, units, and 0/false/null retain their meanings.

These compaction rules apply only to Log V3 and its evidence records. They do not remove fields from performance V2, AV Sync alerts V2, or raw telemetry. Legacy Log V2 retains its eventId, streamId, clockId, serverSessionId, and failureRef semantics. SkyWalking 1.1.0 still uses legacy contracts and does not automatically upload V3.

## 8. Core API Methods

The following are public `SDKClient` methods. Most media and session APIs require a **successful `connect()`**; authentication and connection configuration can be set before connecting.

### 8.1 Connection & Lifecycle

| Method | Description |
| :--- | :--- |
| `preConnect(): Promise<boolean>` | Pre-fetches and caches connection config (TTL is ~60s, see `PRECONNECT_CACHE_DURATION_MS`). Throws on failure. |
| `connect(): Promise<void>` | Obtains connection configuration when needed and connects to the LiveKit room. |
| `disconnect(): Promise<void>` | Stops capture, disconnects LiveKit, and cleans up HTTP sessions. The instance can still be used for `connect()` or `reconnect()`. |
| `reconnect(): Promise<ConnectionSnapshot>` | Re-establishes the connection when allowed and returns the snapshot captured at invocation. When disallowed, skips reconnection and returns the current snapshot. Read the current `connectionSnapshot` or connection events afterward. |
| `installPlugin(plugin): SDKPluginInstallResult` | Atomically installs a Plugin API v1 plugin. Rejection is returned as data and does not affect the SDK lifecycle. |
| `uninstallPlugin(pluginId: string): boolean` | Atomically removes contributions, invalidates old handles, and calls plugin dispose. |
| `dispose(): void` | Releases all resources. The instance **must not** be used after this call. |

**Constraint**: Due to browser policies, **audio playback** and **microphone capture** should ideally be triggered by `connect()` / `startAudioCapture()` within a user gesture (e.g., click) callback.

### 8.2 Authentication & Connection Config

| Method | Description |
| :--- | :--- |
| `setAuthToken(token: string): void` | Sets the token used for subsequent Auth requests; does not invalidate cached room configuration. |
| `updateConnectionConfig(config: DirectConnectionConfig): void` | **Direct Mode only**. Validates and stages `sfuUrl` / `userToken`; since 2.7.0 also accepts optional `sessionId` (see 5.2). **Does not affect the current session**; takes effect during the next `preConnect()` / `connect()` or `reconnect()`. |

### 8.3 Media

| Method | Description |
| :--- | :--- |
| `setRenderFitMode(mode: RenderFitMode): void` | Updates the video object-fit mode. |
| `updateGreenScreenOptions(options: Partial<GreenScreenOptions>): void` | Updates green-screen options in the active connected session; changes are not retained across reconnects. |
| `startAudioCapture(): Promise<void>` | Enables and publishes the microphone through LiveKit; does not automatically enable RMS, PCM, or local visualization. |
| `stopAudioCapture(): Promise<void>` | Stops microphone publishing. |
| `setVolume(volume: number): void` | Sets playback volume (`0` to `1`). |
| `getVolume(): number` | Gets playback volume; in the current 2.8.0 implementation, a zero underlying volume returns `1`. Do not use it to determine real zero volume or mute state. |
| `mute()` / `unmute()` | Controls playback muting without affecting remote audio analysis. |
| `get isMuted` | Returns whether playback is muted. |
| `get isAudioCapturing` | Returns whether the microphone is actively publishing. |
| `startMicLevel(): Promise<void>` | Explicitly enables RMS monitoring. May wait for a later microphone track; resolves with a real sampling window and never starts capture. |
| `getMicLevel(): number \| null` | Synchronously reads linear RMS (0..1). Disabled monitoring, no active microphone, preparation, or unavailability returns `null`; real silence is `0`. |
| `stopMicLevel(): void` | Clears RMS demand and cancels pending level startup without affecting publishing, PCM, or visualization. |
| `getMicrophoneAudioLevel(): number \| null` | **Deprecated; removed in 3.x.** Compatible forwarding alias of `getMicLevel()`, including return and error behavior. |
| `getMicrophoneStats(): Promise<MicrophoneStats \| null>` | Gets microphone transmission statistics and actual track settings. Each call re-samples. `availability` distinguishes missing values from real zero; use `roundTripTimeMs` for milliseconds. Returns `null` without an active microphone, a sender report, or on sender-stat failure. |
| `isMicrophoneSilent(): Promise<boolean \| null>` | Detects if the microphone is sending silence. |
| `startCamera(options?: { publishToLiveKit?: boolean }): Promise<void>` | Opens the camera; optionally publishes to LiveKit room. Defaults to local preview only. |
| `stopCamera(): void` | Stops the camera. |
| `getCameraStream(): MediaStream \| null` | Returns the local stream for previewing. |
| `getCameraTrack(): MediaStreamTrack \| null` | Returns the local media track. |
| `attachCameraTo(videoElement: HTMLVideoElement): void` | Binds the local camera stream to a `<video>` element. |
| `detachCameraFrom(videoElement: HTMLVideoElement): void` | Detaches the camera track from a video element. |
| `startAnalyserEmission(): void` | Starts emitting remote audio analyser data (~60fps) via `media:audio:analyserData`. If no remote audio track has arrived yet, the analyser chain is automatically deferred and will begin emitting once the track is subscribed. Playback muting does not affect analyser data; see "Remote Playback Muting and Audio Visualization" below. |
| `stopAnalyserEmission(): void` | Stops remote audio analyser data emission. |
| `restartAnalyserEmission(): void` | Restarts the remote audio analyser chain. |
| `startLocalAnalyserEmission(): void` | Starts emitting local microphone analyser data via `media:audio:localAnalyserData`. Calls before capture take effect after microphone startup; stopping or restarting visualization does not affect level queries or audio-frame subscriptions. |
| `stopLocalAnalyserEmission(): void` | Stops local microphone analyser emission. |
| `restartLocalAnalyserEmission(): void` | Restarts the local microphone analyser chain. |

### Remote Playback Muting and Audio Visualization

Calling `client.mute()` only mutes remote audio playback; it does not stop remote audio analysis. While the remote audio track is being received normally and `startAnalyserEmission()` is enabled, `media:audio:analyserData` continues to provide analysis data. Remote audio levels, frequency spectra, and waveforms can therefore continue to display and change with the audio signal actually received.

These visualizations reflect the received audio signal, not whether the speakers are producing sound. To stop emitting analysis data, call `stopAnalyserEmission()`.

### Local Microphone Level and Startup Results

Call `startAudioCapture()` from a user gesture. It waits only for microphone enablement and LiveKit publication, so the enabled icon can update immediately after success. Capture and level getters do not automatically enable an SDK-owned analysis graph. RMS, PCM, and local visualization are independent consumers enabled by `startMicLevel()`, actual `media:audio:frameData` subscriptions, and `startLocalAnalyserEmission()`, respectively.

```ts
await client.startAudioCapture();
console.log('Microphone enabled', client.isAudioCapturing);

// Enable only when the application needs levels; the first read after success is real.
try {
  await client.startMicLevel();
  console.log(client.getMicLevel()); // 0 means real silence, never a placeholder.
} catch (error) {
  // AbortError means cancelled waiting; sampling failure must not stop publication.
  console.warn('Level monitoring did not become ready', error);
}

client.stopMicLevel(); // Removes only RMS demand; microphone publishing continues.
```

`startMicLevel()` may register demand before capture and wait for a later track, but it never requests microphone permission or starts publishing. Without a track it creates no sampling resources or readiness timer. Start capture independently; do not await monitoring before triggering capture. Once a track is available, the SDK prepares a real window within approximately 2 seconds without requiring speech, warm-up queries, sleeps, or repeated startup. Repeated calls join the current preparation; an already ready monitor completes immediately.

`getMicLevel()` is synchronous and read-only; it never initializes or recovers sampling. Disabled monitoring, no active microphone, preparation, or unavailability returns `null`; preparation alone does not report an error. The first read after successful `startMicLevel()` returns real RMS, including real silence (`0`). The forwarding `getMicrophoneAudioLevel()` alias is deprecated and will be removed in 3.x. Migration requires explicit monitoring startup as well as the shorter getter name. Existing connected/disposed checks remain.

Within each SDK Client, the three local microphone consumers share one demand-created `AudioContext` and input source, with separate branches. This context is not shared with other Clients, remote analysis, or LiveKit-owned contexts. Adding a consumer ensures shared resources exist; removing it releases only its branch. Removing the last consumer releases the graph, timers, and recovery listeners. Pending monitoring is also a consumer. PCM and visualization do not depend on RMS being enabled or ready, and analysis cannot determine publication success. With no consumers the SDK creates no local analysis graph. LiveKit's native one-time silence detection when creating a track remains; this is not a guarantee of zero temporary AudioContexts throughout the dependency.

A level preparation timeout or fault rejects `startMicLevel()` with `AUDIO_INPUT_MONITORING_UNAVAILABLE` and uses existing nonfatal diagnostic reporting, deduplicated per continuous fault. An RMS branch fault does not stop other healthy branches; a shared context fault requires shared recovery. Disabling analysis does not affect successful microphone publishing. Permission/publication failures still reject `startAudioCapture()`. Drive the icon from startup and `isAudioCapturing`, never from `null` or monitoring errors.

### Microphone Stop, Cancellation, and Sampling Recovery

- While permission acquisition or publication is pending, `stopAudioCapture()`, `disconnect()`, and `dispose()` immediately revoke the old startup operation's authority. A cancelled startup Promise resolves normally, without a microphone-started event or an enablement guarantee. When startup races with stop, use the current `isAudioCapturing` state.
- The browser permission dialog may remain open. The SDK does not wait for that request; a track returned later is stopped without publication. A track whose publication is pending is stopped locally first, and any late publication is cleaned up against that old track without affecting a replacement microphone.
- Normal stop of an already published microphone still waits for unpublication. Unpublication failures still reject the Promise, while local track stopping is attempted independently. One connection-cleanup failure does not skip the remaining release steps. `dispose(): void` synchronously revokes authority and stops known capture resources; remaining asynchronous cleanup runs in the background. It does not guarantee completed network, browser, or operating-system device release.
- Analysis preparation/recovery requires explicit consumer demand and an available track. Suspension first resumes the existing Context; shared graph faults receive at most one automatic rebuild per continuous fault, with an approximately 2-second attempt deadline. Persistent failure stops retrying; a later trusted interaction or hidden-to-visible transition can grant a new bounded attempt, at most once per second. Branch faults do not rebuild healthy branches, and getters never trigger recovery.
- `stopMicLevel()` clears RMS intent and rejects pending startup with `AbortError`. Capture stop, disconnect, track end, or replacement also cancels current waiting and releases the old graph, but retains enabled RMS intent for a later application-triggered capture; disposal clears intent. Rejected Promises do not revive. PCM subscriptions and visualization retain their own lifecycles. Diagnostic recovery never starts capture or switches devices. Publication and enablement do not prove effective audio transmission.

If macOS continues showing active microphone use after Chrome exits, record browser/system versions, the actually loaded SDK/LiveKit versions, device, remaining processes, and indicator duration. Check browser AudioContext device/rendering errors, SDK sampling, RTC publication, and operating-system audio sessions separately. The indicator alone does not establish an SDK leak, and unit tests alone do not establish a device fix.

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
| `get sessionId(): string \| undefined` | Auth keeps its server-ID contract. Since 2.7.0, Direct returns the caller ID or automatic ID prefixed with `sdk-` after the first successful configuration resolution/application; undefined beforehand. Staging does not change it (see 5.2). |
| `setPerformanceMetricReporter(reporter?: PerformanceMetricReporter): void` | **Inert; removed in 3.0.0.** Ignores the callback, creates no detector, and preserves the disposal check. |
| `getPlaybackStatus(): Promise<PlaybackStatus>` | Current connection, media-receiving ICE/age-qualified RTT, per-track receiver metrics, completed output FPS and SDK-defined stalls. Supports single tracks; core metrics do not require debug. |
| `captureRemoteAvSyncDiagnostics(): Promise<AvSyncDiagnosticsSnapshot>` | Actively captures remote audio/video receiver statistics and this connection's track lifecycle nodes; available without debug configuration. |
| `getAvSyncRawTelemetry(query?): Promise<RemoteAvSyncRawTelemetry>` | Reads current bounded context plus one `avSyncLogs` page. The default is the latest five minutes and 200 logs; use `range`, `pageSize`, and `page.nextCursor` to paginate. A fact-only window summary is enabled by default and can be disabled with `includeSummary: false`; log queries and aggregation run in a Dedicated Worker when available, or use the memory fallback when no Worker exists at creation. |
| `getAvSyncAnalysis(query?): Promise<AvSyncAnalysisReport>` | Asynchronously analyzes the complete retained window in the telemetry Worker. The caller controls polling cadence; the report exposes stable `status`, `finding`, `features`, `layers`, and bounded limitations, and ordinary evidence gaps do not reject the promise. Automatic video-stall notification does not depend on calling this method. |
| `get events(): PublicEmitterAPI` | Accesses the event emitter (supports `on`, `off`, `once` only). |

All `getAvSyncAnalysis()` counters are **deltas within the requested analysis window**. Browser WebRTC cumulative receiver counters are never treated as a current anomaly. Important `features` fields are:

- `audioObservationDurationMs`: total observation time across valid remote-audio delta intervals.
- `audioConcealedSamplesDelta` / `audioConcealmentEventsDelta`: audio concealment increments in the window.
- `audioConcealedSamplesPerSecond` / `audioConcealmentEventsPerSecond`: the corresponding increments divided by the observation duration; `null` when no valid audio delta interval exists.
- `videoPacketsLostDelta`, `videoFramesReceivedDelta`, `videoFramesDroppedDelta`, and `videoFramesDecodedDelta`: remote-video increments in the window; received/decoded progression helps distinguish receive/assembly from decode stalls.

For multiple remote participants, the global audio-rate denominator is the sum of valid remote-audio channel observation durations. `participants` and alert participant IDs contain only identities with remote-media evidence in the requested window; local and connection-only identities are excluded.

### 8.6 Playback Status and Diagnostics Selection

Available in **2.5.0**. Use `getPlaybackStatus()` for a connection/playback panel; use the other APIs for diagnostic evidence and analysis.

| API                                | Main use                                                                                              | Fresh stats capture | Debug requirements                                                                                               |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------- | ------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `getPlaybackStatus()`              | Current connection, ICE/RTT, per-track loss/bitrate/resolution/FPS, SDK output FPS/stalls and quality | Yes                 | None for core metrics; `presentationFps` requires `debug.avSync`                                                 |
| `captureRemoteAvSyncDiagnostics()` | One immediate paired A/V receiver snapshot, lifecycle and TimeSync context                            | Yes                 | Callable without debug; continuous TimeSync evidence needs A/V diagnostics                                       |
| `getAvSyncRawTelemetry(query?)`    | Export retained evidence and retrieve a page of diagnostic logs                                       | No                  | Enable `debug.avSync` in advance for continuous receiver/frame evidence; frame metadata requires explicit opt-in |
| `getAvSyncAnalysis(query?)`        | Analyze a retained time window for receive/decode/render/binding/playback issues                      | No                  | Needs sufficient retained evidence and the analysis Worker; otherwise reports `insufficient_evidence`            |

For retained direct APIs and legacy AV Sync contracts (new plugin contracts: see 5.2), when a monitoring session exists, diagnostic snapshots, raw telemetry, analysis reports, alerts, session-ended events, and retained diagnostic logs include `clientInstanceId` and `monitoringSessionId`. Diagnostics `sessionId` identifies one media diagnostics session; old records and delayed analysis retain their original correlation. The `avsync.session.started` log links monitoring and diagnostics sessions through `correlation.avsyncSessionId`.

The APIs share some receiver metrics but have separate roles. Playback sampling supports single audio/video tracks and keeps its interval baseline independent of A/V diagnostic sampling. Immediate diagnostic snapshots organize complete audio/video pairs. Raw telemetry and analysis cannot reconstruct facts that were never collected. Only `avSyncLogs` is paginated; other raw telemetry arrays are bounded context, not necessarily filtered to the same query window. Disconnect preserves frozen diagnostic evidence for querying; reconnect replaces the session, and dispose clears it.

#### Current Playback Status

```ts
// Reuse an initialized client. Run this from a serialized polling loop, roughly once per second.
const status = await client.getPlaybackStatus();
console.log(status.connection.overall.state, status.quality, status.qualityReasons);
console.log(status.ice.connectionType, status.ice.protocol, status.ice.relayProtocol);
console.log(status.ice.rttMs, status.ice.rttAgeMs, status.ice.rttValidForMs);
console.log(status.output?.outputFps, status.output?.stallCount);
console.log(status.output?.stallState, status.output?.stallDurationMs, status.output?.currentStallDurationMs);

for (const track of status.tracks) {
  console.log({
    kind: track.kind,
    lossPercent: track.packetLossRate === null ? null : track.packetLossRate * 100,
    receiveKbps: track.bitrateBps === null ? null : track.bitrateBps / 1000,
    width: track.width,
    height: track.height,
    decodedFps: track.decodedFps,
  });
}
```

- Interval loss and bitrate require two compatible samples; new/replaced receivers start a fresh baseline. `null`/`unknown` means unavailable, not zero or connection failure.
- ICE stats follow the media-receiving PeerConnection: shared publisher in single-PC mode, subscriber in both dual-PC modes. TURN and UDP are not mutually exclusive; `relayProtocol` describes the local client-to-TURN hop.
- RTT is client-to-SFU STUN RTT. The first sample establishes a response-counter baseline; an increase supplies an age-qualified RTT, which expires after 5 seconds. `rttMeasuredAtMs` is the SDK observation time, not exact STUN completion time. Polling does not refresh unchanged evidence.
- `outputFps` measures distinct successful Canvas output/Worker acknowledgements (raw mode: supported video callbacks), not physical screen FPS. SDK stalls mean output gaps >=1 second after first output, excluding the first second of each gap, pauses and hidden-page periods. Browser freeze counters are separate.
- `quality` is a fixed receiver/output heuristic, not a measured end-to-end quality score. It can be `disconnected`, `no-media`, `unknown`, `good` or `degraded`.
- Only `presentationFps` needs `debug: { avSync: true }` or `debug: { avSync: { enabled: true } }`; `debug: true` alone is insufficient. ICE details, RTT, browser freezes and presentation FPS can remain unavailable even with debug. Accurate one-way/end-to-end latency, physical screen FPS and all perceived stalls are not provided.
- Before connection/after teardown, tracks are empty; `output` can be null without a supported active renderer. Stop polling before disposing the client; calls after disposal reject. Connection duration is not collected.

#### Returned Fields and Units

| Field                                                                                    | Meaning and unit                                                                                         |
| ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `sampledAtMs`, `ice.sampledAtMs`, `ice.rttMeasuredAtMs`                                  | Monotonic `performance.now()` milliseconds, not Unix timestamps                                          |
| `ice.rttMs`, `ice.rttAgeMs`, `ice.rttValidForMs`                                         | RTT, observed age and validity in milliseconds; validity is fixed at 5000                                |
| `tracks[].packetLossRate`, `tracks[].bitrateBps`                                         | Interval loss ratio 0–1 and receive bitrate in bit/s; excludes total ICE/TURN bandwidth overhead         |
| `tracks[].width`, `tracks[].height`, `tracks[].decodedFps`                               | Decoded video dimensions in pixels and decoded FPS; not Canvas dimensions or final display FPS           |
| `tracks[].jitterMs`, `tracks[].networkQuality`                                           | RTP jitter in milliseconds and latest remote-participant LiveKit quality; unknown before a quality event |
| `tracks[].freezeCount`, `tracks[].freezeDurationSeconds`, `tracks[].intervalFreezeCount` | Browser-defined cumulative freezes, cumulative seconds and interval freeze count; optional fields        |
| `output.stallCount`                                                                      | Includes the ongoing stall once; resets on track replacement/render-mode change                          |
| `output.stallDurationMs`, `output.currentStallDurationMs`                                | Completed cumulative stall duration and current stall duration, separately in milliseconds               |
| `output.stallState`, `output.stallThresholdMs`                                           | `suspended`/`warming-up`/`flowing`/`stalled`; threshold fixed at 1000 milliseconds                       |

`outputFps` is null without sufficient valid output samples, and zero during a confirmed output stall. Raw output statistics are unavailable without `requestVideoFrameCallback`. `presentationFps` is video-element presentation FPS and needs sufficient valid presentation evidence no older than 3 seconds; it differs from decoded FPS and SDK output FPS.

Quality rules: disconnected RTC means `disconnected`; no remote receivers means `no-media`. Any track with poor/lost network quality, interval loss >=5%, a new freeze, zero receive bitrate or zero interval decoded FPS, or an ongoing SDK output stall, means `degraded`. `good` requires valid interval loss and bitrate for every track with no degradation; otherwise the result is `unknown`. `qualityReasons` provides track-qualified reasons, with `sdk:output-stall` for an ongoing output stall.

Maintainers can find the full metric contract, quality thresholds and availability details in `docs/architecture/playback-status.md` in the SDK source repository.

#### Diagnosis and Historical Evidence

```ts
// Capture a current receiver snapshot; does not require debug.
const snapshot = await client.captureRemoteAvSyncDiagnostics();

// Analyze evidence already collected for the most recent 30 seconds.
const analysis = await client.getAvSyncAnalysis({
  range: { kind: 'recent', durationMs: 30_000 },
});
console.log(analysis.status, analysis.finding, analysis.confidence, analysis.limitations);

// Export one evidence page. Follow page.nextCursor for additional pages.
const evidence = await client.getAvSyncRawTelemetry({
  range: { kind: 'recent', durationMs: 30_000 },
  pageSize: 200,
});
```

Do not poll all four methods for a status panel. Poll playback status, invoke immediate capture when needed, and query analysis/raw evidence on demand. Analysis can publish analysis results and update diagnostic alerts; merely querying it does not enable background sampling.

### 8.7 Version Information

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
- **Description**: `allConnected === false` means the mode's overall connection condition is not met; it does not prove all channels are disconnected or manual reconnect is permitted. Check `connectionSnapshot` and current state after reconnect.

### `sdk:error`

- **Trigger**: When the SDK reports an error or an unavailable capability. Handle it by `code`; `AUDIO_INPUT_MONITORING_UNAVAILABLE` only means local sampling is unavailable, not that microphone publishing failed.
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

- **Trigger**: When a remote participant whose identity starts with `renderer_` disconnects from the LiveKit room.
- **Payload**: `{ participantId: string }`.

**Video**

| Event | Trigger | Payload |
| :--- | :--- | :--- |
| `media:video:available` | Remote video playback availability signal; does not guarantee a physical first frame has been displayed. | `undefined` |
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
| `media:audio:frameData` | Raw audio frame data with sample rate; processing is active only while public subscribers exist. | `{ data: Float32Array; sampleRate: number }` |

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

`representations` contains JSON-safe projections ordered by successful plugin installation and setup registration. Equal `mediaType` values do not overwrite each other. New code should use only `rawDelta` / `rawText`. Legacy fields remain in 2.8.0 without runtime warnings and can be removed only in a future major release.

**Session**

| Event | Trigger | Payload |
| :--- | :--- | :--- |
| `session:closing` | Server-initiated session closure. | `Record<string, unknown>` |

---

## 10. Full Usage Example

```ts
import { createClient, SDK_PERFORMANCE_METRIC_V1, VERSION } from '@sanseng/liveavatar-js-sdk';

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
    plugins: [{
      id: 'performance-collector',
      apiVersion: 1,
      setup(registrar) {
        registrar.observe(SDK_PERFORMANCE_METRIC_V1, {
          id: 'individual-metrics',
          handler: (metric) => console.log('[measurement]', metric.metric, metric.durationMs),
        });
      },
    }],
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

When enabling client Chroma Key for a plain or legacy stream, ensure the following settings are applied. The SDK never uses these settings to infer a packed transparent layout. The render mode can be selected when constructing the SDK or through Auth connection configuration; after connection, `updateGreenScreenOptions({ enabled })` can switch between the raw and processed paths:

- `video.renderMode = 'processed'`
- `greenScreen.enabled = true`
- `greenScreen.isBackgroundKeying = true` (optional: key only when at least three frame edges are screen)

### 11.1 Tuning Recommendations

1. **Background Color Selection (`chromaKey`)**
   - High-chroma green (hue `80°..160°`) and blue (hue `200°..260°`) are supported. Prefer green screens; choose blue when the subject's clothing or props contain substantial green.
   - Each RGB component must be a finite value in `0..255`; saturation must be `≥ 55%`, value `≥ 30%`, and the YCbCr chroma radius `≥ 0.35`.
   - Red, cyan, purple, gray/white/black, low-saturation, dark, and out-of-range complete tuples fall back to `[0, 255, 0]` instead of being used for keying.
   - Pick a representative well-lit screen color from a real video screenshot. This version accepts one key only; it does not yet automatically sample spatial screen variations caused by shadows or reflections.

2. **Similarity (`similarity`)**
   - Default: `0.4`
   - Start from `0.3` and adjust incrementally.
   - Values too high may erroneously remove person details (e.g., hair or clothing).
   - The SDK may reduce its effective value together with `smoothness` to keep neutral foreground pixels outside the transparent feather.

3. **Edge Color Spill Suppression (`despillStrength`)**
   - Default: `1.15`
   - Removes green reflections for green keys and blue reflections for blue keys.

4. **Edge Smoothness (`smoothness`)**
   - Default: `0.25`
   - Improves blending for hair and semi-transparent areas.

5. **Background Gate (`isBackgroundKeying`)**
   - Default: `false`. When set to `true`, the SDK downsizes each frame to a `32 × 18` sample and checks the top, right, bottom, and left edges independently with the current `chromaKey` and effective `similarity`.
   - Existing chroma key and despill run only when at least three of the four edges each have at least `75%` key-color samples. If fewer than three edges qualify or the browser cannot read the sample frame, the SDK renders that frame unchanged instead of applying blind keying.
   - This protects a central green object in a scene whose edges are not a screen. When a real green screen surrounds the frame, green clothing or props remain indistinguishable from the background to a color-only algorithm; that case requires person/foreground segmentation.

### 11.2 Runtime Adjustment

After the client connects, options can be patched without rebuilding the SDK or processor. Ordinary tuning reuses the current CPU/WebGL backend. Passing `enabled: true` immediately switches to the processed Canvas path; passing `enabled: false` immediately restores the raw video path. All other green-screen options also take effect immediately.

```ts
client.updateGreenScreenOptions({
  enabled: true,
  similarity: 0.34,
  smoothness: 0.18,
  despillStrength: 1.25,
  isBackgroundKeying: true,
});
```

This method is available only while `client.isConnected === true`. Its patch applies only to the active connection; after `reconnect()`, green-screen settings are restored from the constructor options or Auth configuration resolved for that connection.

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

The following are string values from the internal `ErrorCode` enum, consistent with `sdk:error` and thrown `SDKError` objects. `ErrorCode` is not exported from the package root; check the public error object's string `code` instead.

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
| `LIVEKIT_DISCONNECTED` | An established Room ended because of an unrecoverable runtime transport, media, protocol, agent, or SIP-trunk fault. Inspect the paired `sdk:disconnected` reason. |
| `LIVEKIT_SEND_VIDEO_AVAILABLE_STATE_FAILED` | Failed to report video available state. |
| `LIVEKIT_SEND_TEXT_DATA_FAILED` | Failed to send text data via LiveKit Data Channel. |
| `LIVEKIT_DATA_MESSAGE_PARSE_ERROR` | Failed to parse data message. Check protocol version compatibility. |
| `LIVEKIT_UNPUBLISH_MICROPHONE_FAILED` | Failed to unpublish the microphone track. |

### 13.4 Audio

| Code | Description & Recommendation |
| :--- | :--- |
| `AUDIO_CAPTURE_START_FAILED` | Mic start failed. Check user gestures, HTTPS, and permissions. |
| `AUDIO_INPUT_MONITORING_UNAVAILABLE` | Local sampling or level measurement is unavailable; microphone publishing continues. Do not disconnect solely because of it. |
| `AUDIO_CAPTURE_FAILED` | Capture interrupted. |
| `AUDIO_INVALID_SAMPLE_RATE` / `AUDIO_INVALID_CHANNEL` / `AUDIO_INVALID_CODEC` | Parameter mismatch with device or protocol. |
| `AUDIO_CONTROLLER_NOT_AVAILABLE` | Audio controller not created or already released. |
| `AUDIO_OUTPUT_DISABLED` | The internal output pipeline can report this when output is disabled. `getAudioElement()` is not a public `SDKClient` method. |

### 13.5 Video / Camera

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
| `connect_to_first_frame_ms` | Time from `connect()` to `loadeddata` on the SDK-owned internal video element. |
| `text_send_to_text_response_ms` | Time from sending text query to receiving the first text response chunk. |
| `text_send_to_audio_response_ms` | Time from sending text query to receiving the first audio response chunk. |
| `no_speech_report_to_audio_response_ms` | Time from "no-speech" report to receiving the audio response. |

Data structure: `PerformanceMetricRecord` contains `metric`, `durationMs`, `startedAt`, `endedAt`, optional `questionId`, and optional `attemptId` for first-frame metrics. When a monitoring session exists, it also contains `clientInstanceId` and `monitoringSessionId`; delayed results retain the snapshot captured when measurement started.

For connection stages, connection failure/first-frame timeout, text responses, and text-to-audio responses, `durationMs` uses elapsed `performance.now()` time. `startedAt` is the Unix timestamp sampled at measurement start, and `endedAt = startedAt + durationMs`. System clock jumps cannot change these durations or reverse timestamps within a record. Fractional milliseconds are retained; use `durationMs` for elapsed time.

These timestamps describe a measurement timeline anchored at its start. After a system clock adjustment, `endedAt` may differ from `Date.now()` at completion; it is not an independently sampled completion wall-clock fact. Separate measurements take fresh anchors and need not remain globally ordered. A successful first-frame batch reuses the first-frame metric endpoint. Operational Log occurrence timestamps retain their existing wall-clock semantics.

The legacy `no_speech_report_to_audio_response_ms` still uses the event-provided wall-clock start and wall-clock subtraction, so it remains affected by system clock adjustment. No production emitter or monotonic origin has been identified; this fix excludes that metric and does not guarantee its availability in normal sessions.

### 14.1a Segmented First-Frame Timeline

Every effective connect creates an increasing attemptId after duplicate-connect protection. All stages are cumulative from that connect start. The established connect_to_first_frame_ms name remains compatible and ends exactly at loadeddata on the SDK-owned persistent video element. It does not mean application pixels are already visible, playing occurred, or requestVideoFrameCallback ran.

| Metric | First fact |
| --- | --- |
| connect_to_config_ready_ms | connection configuration applied |
| connect_to_rtc_connected_ms | internal RTC connected |
| connect_to_video_track_added_ms | remote video track added |
| connect_to_video_metadata_ms | internal video loadedmetadata |
| connect_to_first_frame_ms | internal video loadeddata |
| connect_to_first_frame_timeout_ms | no loadeddata within 10 seconds after RTC connected |
| connect_to_first_frame_failed_ms | either connect failure path ended before first frame |

A stage is recorded once. A stage that did not occur is never written as 0 and is omitted from the terminal report. First-frame PerformanceMetricRecord values have an optional attemptId; existing conversation metrics continue to use their optional questionId.

After RTC connected, the SDK starts a 10-second first-frame timer. loadeddata, connect failure, explicit disconnect, and dispose clear it. Explicit disconnect, dispose, and a superseded attempt are cancellations only and do not emit failure or timeout metrics. A passive RTC disconnect that does not enter a connect failure path remains bounded by this timer.

### 14.1b Terminal Batch and Plugin Inputs

Every successful, failed, or timed-out attempt produces one immutable SDKPerformanceMetricsReport. Its schema is sdk.performance-metrics, schemaVersion is 1, and it contains attemptId, optional clientInstanceId/monitoringSessionId, outcome, startedAt, endedAt, and metrics for present stages only. The SDK package root exports the detached SDK_PERFORMANCE_METRICS_V1 Observer token. Observers receive a deep-cloned frozen snapshot; the SDK does not await handlers, and a plugin throw, rejection, or timeout cannot change connection or media behavior.

The SDK also emits a sanitized Operational Log with category media and event performance.first-frame. Its top-level envelope carries the monitoring correlation; its correlation contains only attemptId. Attributes contain only outcome, renderMode, timeoutAfterRtcMs for a timeout, and sparse numeric nodes. It excludes URLs, tokens, connection configuration, room/track/participant identifiers, text, raw DataChannel data, media frames, DOM/LiveKit objects, Error, message, and stack. This SDK release does not upload automatically. A plugin must register the observer or match the log and owns setPerformance, reportFrameErrors, authorization, retries, and delivery; SkyWalking 1.1.0 remains on legacy contracts.

### 14.2 Structured Measurements and Plugin Integration

The SDK provides performance measurements; plugins receive and upload them. `setPerformanceMetricReporter()`, `PerformanceMonitorOptions.reporter`, and `PerformanceMetricReporter` retain their public signatures only, now have no effect, and will be **removed in 3.0.0**. Callbacks are never invoked and default `[performance-metric]` Console output is removed. This explicitly changes the old reporter behavior. `performanceMonitor.enabled` still controls performance measurement and defaults to enabled.

Plugins consume individual `PerformanceMetricRecord` measurements through `SDK_PERFORMANCE_METRIC_V1` (identity `performance.metric`, major 1), including stage and conversation response timings without an additional envelope. The current implementation validates metric names, finite nonnegative durations, ordered finite timestamps, and optional `questionId` / `attemptId`. Consumers of the same dispatch share one deep-cloned frozen snapshot. Detached delivery uses microtasks and does not enforce the configured queue limit or overflow policy; handler timeout is 30 seconds and faults disable the contribution. Earlier documentation promised independent snapshots and a drop-oldest queue of 100; the implementation differs, creating a compatibility risk for plugins relying on those guarantees. This clarification changes neither the plugin contract nor the SDK implementation. Plugins own upload, retries, and storage.

`SDK_PERFORMANCE_METRICS_V1` retains the unchanged terminal first-frame batch protocol. A first-frame measurement can occur in both streams; choose the appropriate stream to avoid double counting. Existing output-stall detection remains available through `getPlaybackStatus()`, with unchanged thresholds, and is not added to the performance metric streams. Disconnect, disposal, or a new connection attempt clears pending conversation measurements to prevent cross-connection attribution.

Audio-response timings do not represent the exact time sound becomes audible or guarantee per-question audio attribution. `no_speech_report_to_audio_response_ms` is not guaranteed in normal sessions; do not fill a missing measurement with `0`.

```ts
import { createClient, SDK_PERFORMANCE_METRIC_V1 } from '@sanseng/liveavatar-js-sdk';

const client = createClient({
  connectConfig: { type: 'auth', config: { avatarId: 'demo' } },
  plugins: [{
    id: 'performance-collector',
    apiVersion: 1,
    setup(registrar) {
      registrar.observe(SDK_PERFORMANCE_METRIC_V1, {
        id: 'individual-metrics',
        handler(metric) {
          console.log('[measurement]', metric.metric, metric.durationMs, metric.attemptId ?? metric.questionId);
        },
      });
    },
  }],
});
```

### 14.3 Troubleshooting Recommendations

- **Slow First Frame**: Check region latency, TURN server status, and whether repeated `connect()` retries are occurring.
- **Slow Text Response**: Correlate with backend LLM/business processing time and check for main-thread blocking.
- **Audio Significantly Slower Than Text**: Inspect the TTS pipeline and remote audio track subscription states.

---

_Document version consistent with package version: 2.8.0._
