# Facemarket Live Avatar SDK Plugin Integration Guide (v2.8.0)

This guide is for **plugin developers**. It explains how to build, package, and maintain a plugin using the first public `@sanseng/liveavatar-js-sdk` Plugin API v1.

If you only need to install an existing plugin in an application, see the “Plugin Import and Configuration” section of the [SDK User Manual](./SDK_User_Manual.md). For Markdown rendering and SkyWalking integration, see “Currently Supported Plugin Packages.”

## 1. Scope and boundaries

Plugin API v1 is the first public contract for trusted, in-process TypeScript extensions. A plugin can register contributions only against versioned extension points defined by the SDK and exported from the package root, or consume a read-only CoreService.

Plugins are not a security sandbox and must not:

- define extension points or consume unexported internal tokens;
- replace RTC, transport, the Conversation FSM, MessageAssembler, media ownership, or AV Sync collection;
- obtain SDKClient, EventBus, LiveKit Room, tokens, HTTP controllers, or internal controllers;
- rely on inter-plugin communication or plugin-provided Providers; v2.8.0 exports no business Provider token.

The SDK retains ownership of protocol parsing, connection state, conversation source text, and media pipelines. Plugins own their HTTP, authorization, queues, retries, persistence, sampling, deduplication, and drain policy.

## 2. Package layout and dependencies

Declare the SDK as a peer dependency so a plugin does not embed a second copy of the SDK:

```json
{
  "peerDependencies": {
    "@sanseng/liveavatar-js-sdk": "^2.6.0"
  }
}
```

The `^2.6.0` example supports Logging V2, A/V evidence V2, individual performance measurements, and pre-disposal Observer handoff. A plugin using only `getPlaybackStatus()` can retain a `2.5.0` minimum. Declare the minimum for the capabilities the plugin actually uses; this does not assert compatibility for any published plugin package. Plugin API remains `1`. Plugins using the unified-session points in Section 4.0 require **2.7.0** or later; the older minimums below remain valid for plugins using only older capabilities.

Export one or more `SDKPlugin` objects. Do not import types or tokens from `src/` or any other SDK internal path; import only from `@sanseng/liveavatar-js-sdk`.

## 3. Plugin API v1 contract

```ts
import type { SDKPlugin } from '@sanseng/liveavatar-js-sdk';

export const examplePlugin: SDKPlugin = {
  id: 'com.example.liveavatar-plugin',
  apiVersion: 1,
  version: '1.0.0',
  setup(registrar) {
    // Register observe / transform / provide contributions, or consume a CoreService synchronously.
  },
  dispose() {
    // Release plugin-owned resources; do not access SDK internals.
  },
};
```

| Field              | Requirement                                                                                                                               |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `id`               | Unique in the current SDK instance and matching `[A-Za-z0-9][A-Za-z0-9._-]*`. The ID can be installed again after uninstall.              |
| `apiVersion`       | Must be the literal `1`. It is independent of individual extension-point major versions.                                                  |
| `version`          | Optional plugin release version for your own publishing and diagnostics.                                                                  |
| `terminalObserverDelivery` | Optional literal `'sync-before-dispose'`; opts this installation into the Client-disposal handoff in §5.1. |
| `setup(registrar)` | Must complete synchronously. Returning a Promise/thenable, throwing, or registering an invalid contribution rolls the whole install back. |
| `dispose()`        | Optional; called on uninstall and SDK disposal. A throw does not interrupt the SDK but emits a sanitized `sdk:pluginFault`.               |

The `registrar` is valid only while `setup()` runs. Do not retain it for later registration or service consumption; calls after setup throw `SDK_PLUGIN_CONTEXT_INACTIVE`.

Every contribution needs an ID unique **within the plugin** and a handler. Installation validates point kind, handler, contribution ID, and Provider conflicts before committing everything atomically.

## 4. Available extension points

All tokens are exported from the `@sanseng/liveavatar-js-sdk` package root. A token’s `{ id, major, kind }` is stable identity: compatible additions keep the same major and breaking changes receive a new token major.

| Token                              | Type                                                | Purpose and execution semantics                                                                                                                                                   |
| ---------------------------------- | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SDK_OPERATIONAL_LOGS_V2` | `ObserverPoint<SDKLogRecordV2>` | Since 2.6.0: structured operation outcomes, failure details, and monitoring-session correlation. Choose one real-time log version per purpose. |
| `SDK_AVSYNC_DIAGNOSTICS_V2` | `CoreServicePoint<SDKAvSyncDiagnosticsServiceV2>` | Since 2.6.0: paginated retained evidence through `getAvSyncLogEvidence(query?)`; does not start sampling. |
| `SDK_OPERATIONAL_LOGS_V1`          | `ObserverPoint<SDKOperationalLogRecord>`            | Observes sanitized SDK operational logs. Detached and non-blocking; records exclude tokens, Authorization, text bodies, raw DataChannel payloads, media frames, and Error stacks. |
| `SDK_PERFORMANCE_METRIC_V1` | `ObserverPoint<PerformanceMetricRecord>` | Individual stage and conversation response measurements; detached, validated, cloned and frozen. See the documented implementation differences in §4.2. |
| `SDK_PERFORMANCE_METRICS_V1`      | `ObserverPoint<SDKPerformanceMetricsReport>`        | Observes immutable terminal first-frame metric batches. Detached, schema-validated, and deeply frozen. See §4.2. |
| `SDK_CONVERSATION_INBOUND_TEXT_V1` | `TransformPoint<SDKInboundText, SDKTextProjection>` | Adds representations for inbound answers, system prompts, and ASR. Synchronous, parallel, fail-open; cannot change source text, protocol data, or the FSM.                        |
| `SDK_AVSYNC_ALERTS_V1`             | `ObserverPoint<AvSyncAlertEvent>`                   | Observes the AV Sync alert lifecycle. Detached; installing an observer does not start analysis.                                                                                   |
| `SDK_AVSYNC_ANALYSES_V1`           | `ObserverPoint<AvSyncAnalysisReport>`               | Observes one result for each real single-flight analysis.                                                                                                                         |
| `SDK_AVSYNC_SESSION_ENDED_V1`      | `ObserverPoint<AvSyncSessionEndedEvent>`            | Observes session end only after local segment sealing and worker segment close.                                                                                                   |
| `SDK_AVSYNC_DIAGNOSTICS_V1`        | `CoreServicePoint<SDKAvSyncDiagnosticsService>`     | Consumes playback status, diagnostics snapshots, raw telemetry, and analysis queries; results are defensive copies and an old proxy becomes inactive after uninstall.                              |

The generic `ProviderPoint` runtime exists, but v2.8.0 exposes no business Provider token. Plugins therefore cannot currently call `registrar.provide()` to supply SDK-used business behavior.

### 4.0 Unified session contracts (Since 2.7.0)

The following additive points are available since SDK **2.7.0**. Plugins using any of them must declare a minimum SDK version of **2.7.0**, for example `^2.7.0`. Plugin API stays **1**. Existing points keep their identities, field meanings, schemas, availability, timing, snapshot behavior, timeout/fault isolation, and uninstall/reinstall invalidation. Select one version of a fact stream per destination to avoid duplicates.

| Token | Identity / payload | Correlation |
| --- | --- | --- |
| `SDK_OPERATIONAL_LOGS_V3` | `logging.operational`, major 3, observer; `SDKLogRecordV3`, `sdk.log` / 3 | Application `sessionId`, `sessionEpoch`; optional media `avsyncSessionId`; compact stream/sequence identity |
| `SDK_PERFORMANCE_METRIC_V2` | `performance.metric`, major 2, observer; `PerformanceMetricRecordV2`, no envelope | Captured application session at measurement start |
| `SDK_PERFORMANCE_METRICS_V2` | `performance.metrics`, major 2, observer; `SDKPerformanceMetricsReportV2`, `sdk.performance-metrics` / 2 | Same application ownership on batch and metrics; `attemptId` unchanged |
| `SDK_AVSYNC_ALERTS_V2` | `avsync.alerts`, major 2, observer; `AvSyncAlertEventV2` | Application `sessionId`/`sessionEpoch`; media `avsyncSessionId` |
| `SDK_AVSYNC_ANALYSES_V2` | `avsync.analyses`, major 2, observer; `AvSyncAnalysisReportV2`, schemaVersion 2 | Same captured application/media ownership |
| `SDK_AVSYNC_SESSION_ENDED_V2` | `avsync.session-ended`, major 2, observer; `AvSyncSessionEndedEventV2` | Same captured application/media ownership |
| `SDK_AVSYNC_DIAGNOSTICS_V3` | `avsync.diagnostics`, major 3, core-service; `SDKAvSyncDiagnosticsServiceV3` | Playback status unchanged; modern capture, raw, analysis and evidence projections |

New outputs omit `monitoringSessionId`. Raw telemetry and analysis use schemaVersion **2**; evidence pages use `schema: 'sdk.log-evidence'`, schemaVersion **3**, and contain Log V3 records. The diagnostics service has `getPlaybackStatus()`, `captureRemoteAvSyncDiagnostics()`, `getAvSyncRawTelemetry(query?)`, `getAvSyncAnalysis(query?)`, and `getAvSyncLogEvidence(query?)`. Queries, units, retention, cursor isolation and first-call availability remain unchanged; consuming the service does not start sampling. Legacy direct Client methods and public `sdk:avSyncAlert` keep their legacy payloads.

New points retain their predecessors' detached policies: declared drop-oldest queue limits of 1000 for logs and 100 for performance/AV Sync, a 30-second handler timeout, and disable-contribution faults. The existing runtime discrepancy described in §4.2 (one shared frozen dispatch snapshot and unenforced queue limits) also applies; these new majors do not fix or redefine that behavior. CoreService proxies remain scoped to one plugin installation and become inactive after uninstall.

`SDKSessionCorrelation` is `{ clientInstanceId, sessionId, sessionEpoch }`. Epochs start at 1 and never repeat within a Client. A Direct caller may repeat a business `sessionId`; use `clientInstanceId + sessionEpoch` for lifecycle isolation, retain `sessionId` for business joins, and use the Log V3 stream/sequence keys below for record deduplication. A staged Direct config does not change the active getter or ownership. Applied changes to normalized URL, token, or caller ID create a new epoch; unchanged configuration, cache/refresh and internal reconnect retain it. Explicit disconnect ends it; a later application creates a new epoch and a new automatic ID when no caller ID was supplied.

Automatically generated application IDs use an `sdk-` prefix, with `crypto.randomUUID()` when available and a timestamp plus random-string fallback otherwise. Treat them as opaque strings; do not parse or require a UUID suffix. Auth real fetches allocate provisional local IDs using these rules. A returned server ID binds within the same epoch. Only Log V3 emits `session.id-bound`, with the new top-level `sessionId` and only `previousSessionId` in `data`; absence of a server ID keeps the local ID in modern output. Log V3 does not output `serverSessionId`; internal Auth identity and other versioned contracts retain their fields. Direct never fabricates a server ID. Auth `client.sessionId` keeps its original getter contract. Resource and asynchronous work retain captured ownership; historical fields are not filled from the current session.

```ts
import {
  SDK_OPERATIONAL_LOGS_V3,
  SDK_PERFORMANCE_METRIC_V2,
  SDK_AVSYNC_DIAGNOSTICS_V3,
  type SDKPlugin,
} from '@sanseng/liveavatar-js-sdk';

const unifiedSessionPlugin: SDKPlugin = {
  id: 'business.session-observer',
  apiVersion: 1,
  terminalObserverDelivery: 'sync-before-dispose',
  setup(registrar) {
    const diagnostics = registrar.consume(SDK_AVSYNC_DIAGNOSTICS_V3);
    registrar.observe(SDK_OPERATIONAL_LOGS_V3, {
      id: 'facts',
      handler(record, { signal }) {
        if (!signal.aborted) {
          // Hand off synchronously to a plugin-owned queue in a real integration.
          console.log(record.clientInstanceId, record.sessionId, record.sessionEpoch, record.event);
        }
      },
    });
    registrar.observe(SDK_PERFORMANCE_METRIC_V2, {
      id: 'measurements',
      handler(metric, { signal }) {
        if (!signal.aborted) console.log(metric.sessionId, metric.sessionEpoch, metric.durationMs);
      },
    });
    // Retain diagnostics only for this plugin installation; query on explicit demand.
    // void diagnostics.getAvSyncLogEvidence().then(handleEvidence).catch(handleQueryError);
    void diagnostics;
  },
};
```

The currently reviewed SkyWalking plugin **1.1.0** consumes legacy contracts. Keep those registrations until a separate plugin release migrates them. Plan collector schema and query migration alongside plugin work before the future **3.0** removal of legacy `monitoringSessionId` outputs. This change adds the new contracts and preserves old outputs; it does not migrate the plugin or remove old outputs.

#### Compact Log V3 records

Cross-instance diagnostic IDs use unpadded Base64URL encoding: `clientInstanceId` is `c_` plus 22 characters, `monitoringSessionId` is `m_` plus 22, and the local-evidence page lifecycle ID is `p_` plus 22. The SDK first encodes all 16 bytes of `crypto.randomUUID()`; if unavailable or failing, it obtains 16 bytes from `crypto.getRandomValues()`. It neither truncates nor hashes the UUID, retaining its random bits and collision resistance. If both cryptographic sources are unavailable, a type prefix, base-36 time, module-local counter and random string provide a compatibility fallback; that path has no fixed 24-character length or cryptographic uniqueness guarantee. Random IDs do not promise mathematically impossible collisions. Treat all these IDs as opaque strings rather than parsing their prefix or length.

Short cross-instance IDs apply across log, performance and diagnostic versions, so derived IDs containing the Client ID also become shorter. Existing fields, composition rules and uniqueness scopes remain, and historical records are not rewritten. Caller, backend, browser and LiveKit IDs, and the automatic application `sessionId` rules prefixed with `sdk-`, are unchanged. `failureId` and `alertId` retain their full Client-qualified identity for independent consumers; do not derive them from short operation IDs.

Log V3 retains `schema: 'sdk.log'`, `schemaVersion: 3`, `SDK_OPERATIONAL_LOGS_V3` major 3. This format was introduced in SDK 2.7.0; the current SDK is 2.8.0; legacy V1/V2 fields and meanings stay intact.

Since 2.8.0, browser failures classified as `permission` from structured `NotAllowedError` / `SecurityError` evidence downgrade candidate `error` to `warn` in V1 `sdk.error` / `operation.failed` and V2/V3 `sdk.failure`. Existing lower severity, precondition precedence and confirmed-cancellation rules remain. HTTP 401/403, LiveKit `NotAllowed`, backend/device failures and unknown errors are not automatically downgraded. Retain failure records, IDs, codes, outcomes and references; warn does not mean success or confirmed user refusal. Plugins filtering only error must accept warn to retain these diagnostics, or count failure events independently of severity. Public errors and Promise rejections are unchanged.

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

##### Record examples and migration checks

These are shapes of four independent facts, rather than a complete connection timeline. Results reference top-level failureId; a root operation omits its own root ID and finished phase; binding keeps the new ID only at top level; browser media correlation uses mediaStreamId.

```json
[
  {"schema":"sdk.log","schemaVersion":3,"clientInstanceId":"c_AAECAwQFBgcICQoLDA0ODw","stream":"operational","sequence":10,"sessionId":"sdk-local","sessionEpoch":1,"occurredAtMs":1793836800000,"observedAtMs":1020,"level":"error","module":"rtc","event":"sdk.failure","failureId":"c_AAECAwQFBgcICQoLDA0ODw:failure:1","operation":{"operationId":"o2","rootOperationId":"o1","parentOperationId":"o1","name":"rtc.connect","phase":"connect"},"error":{"code":"LIVEKIT_CONNECT_FAILED","reasonCategory":"timeout","evidence":{"name":"ConnectionError","connectionErrorReason":5,"connectionErrorReasonName":"Timeout"}}},
  {"schema":"sdk.log","schemaVersion":3,"clientInstanceId":"c_AAECAwQFBgcICQoLDA0ODw","stream":"operational","sequence":11,"sessionId":"sdk-local","sessionEpoch":1,"occurredAtMs":1793836800001,"observedAtMs":1021,"level":"info","module":"client","event":"operation.finished","failureId":"c_AAECAwQFBgcICQoLDA0ODw:failure:1","operation":{"operationId":"o1","name":"connect","outcome":"failed"}},
  {"schema":"sdk.log","schemaVersion":3,"clientInstanceId":"c_AAECAwQFBgcICQoLDA0ODw","stream":"operational","sequence":12,"sessionId":"server-bound","sessionEpoch":1,"occurredAtMs":1793836801000,"observedAtMs":2020,"level":"info","module":"client","event":"session.id-bound","data":{"previousSessionId":"sdk-local"}},
  {"schema":"sdk.log","schemaVersion":3,"clientInstanceId":"c_AAECAwQFBgcICQoLDA0ODw","stream":"avsync","sequence":1,"sessionId":"server-bound","sessionEpoch":1,"avsyncSessionId":"av-1","occurredAtMs":1793836801100,"observedAtMs":2120,"level":"debug","module":"avsync","event":"avsync.evidence","correlation":{"mediaStreamId":"browser-stream-1"},"data":{"node":"rtc-receiver","phase":"rtc-sample","eventName":"receiver-stats-sampled","logType":"rtc-sample","sessionElapsedMs":100,"measurements":{"framesDecoded":0,"packetsLost":0}}}
]
```

Build storage keys from the structured tuples above, for example with `JSON.stringify([...])`; avoid concatenating business IDs that can contain separators. Do not parse old streamId values or replace record keys with application session ID/epoch. Migration checks should cover repeated evidence pages, media window replacement, stable record identity across Auth binding, lifecycle isolation when business IDs repeat, multiple result references to one failure, and real zero versus missing values. Keep old V2 eventId/failureRef queries for old consumers; migrate V3 plugins and collector queries separately. These examples do not establish SkyWalking V3 delivery.

The examples below intentionally document retained contracts for existing plugins.

### 4.1 Observer

```ts
import { SDK_OPERATIONAL_LOGS_V1, type SDKPlugin } from '@sanseng/liveavatar-js-sdk';

export const operationalLogPlugin: SDKPlugin = {
  id: 'com.example.operational-log',
  apiVersion: 1,
  setup(registrar) {
    registrar.observe(SDK_OPERATIONAL_LOGS_V1, {
      id: 'enqueue-operational-log',
      handler: (record, { signal }) => {
        if (signal.aborted) return;
        enqueuePluginOwnedRecord(record);
      },
    });
  },
};
```

Operational Log and AV Sync Observers are currently detached: the SDK never awaits a handler Promise. A throw, rejection, or timeout disables that contribution and emits at most one `sdk:pluginFault`; SDK production, disconnect, persistence, and disposal continue.

### 4.1a Monitoring correlation

For the legacy V1 contract, Operational Log records continue to expose `clientInstanceId` and, in Auth mode, the server `sessionId`. Additive `monitoringSessionId` identifies the SDK monitoring lifecycle; use `clientInstanceId + monitoringSessionId` when joining logs, performance observations, and AV Sync output.

Additional lifecycle events:

| Event | Correlation / attributes |
| --- | --- |
| `monitoring.session.started` | `attributes.startReason` |
| `monitoring.session.ended` | fixed `attributes.endReason`; optional `attributes.cleanupOutcome` = `completed`, `failed`, or `deferred` |
| `config.fetch.started/completed/failed` | `correlation.configRoundId`, `correlation.attempt`, source and retry status; no config content |
| `avsync.session.started` | `correlation.avsyncSessionId` maps the monitoring session to one diagnostics session |

The first output of a normal operation already contains correlation; plugins do not need warm-up calls. Explicit disconnect ends a monitoring session once. Dispose publishes `deferred` before the SDK releases the publisher and plugin host, but detached delivery remains best-effort and is not an upload acknowledgement. A delayed operation keeps its original monitoring snapshot after configuration changes.

### 4.1b Structured Logs and AV Sync Evidence V2

Use `SDK_OPERATIONAL_LOGS_V2` for live records and consume `SDK_AVSYNC_DIAGNOSTICS_V2` during `setup()` for later evidence queries. The example writes to Console; replace that with your plugin-owned upload queue.

```ts
import {
  SDK_OPERATIONAL_LOGS_V2,
  SDK_AVSYNC_DIAGNOSTICS_V2,
  type SDKLogEvidencePageV2,
  type SDKPlugin,
} from '@sanseng/liveavatar-js-sdk';

export function createDiagnosticPlugin(): SDKPlugin & {
  readEvidence(): Promise<SDKLogEvidencePageV2> | undefined;
} {
  let readEvidence: (() => Promise<SDKLogEvidencePageV2>) | undefined;
  return {
    id: 'com.example.diagnostics-v2',
    apiVersion: 1,
    setup(registrar) {
      const diagnostics = registrar.consume(SDK_AVSYNC_DIAGNOSTICS_V2);
      readEvidence = () => diagnostics.getAvSyncLogEvidence({
        range: { kind: 'recent', durationMs: 30_000 },
        pageSize: 200,
      });
      registrar.observe(SDK_OPERATIONAL_LOGS_V2, {
        id: 'operational',
        handler(record) {
          console.log(record.module, record.event, record.operation?.outcome);
          if (record.event === 'sdk.failure') {
            console.warn(record.error?.code, record.error?.reasonCategory);
          }
        },
      });
    },
    dispose() {
      readEvidence = undefined;
    },
    // An application may expose a separate plugin-owned control to invoke this.
    readEvidence() {
      return readEvidence?.();
    },
  };
}
```

`SDKLogRecordV2` uses `schema: 'sdk.log'` and `schemaVersion: 2`. Filter by `module` and `event`; V2 has no `category`. Correlate a monitoring session using `clientInstanceId + monitoringSessionId`. Keep `serverSessionId` (Auth session) and `avsyncSessionId` (diagnostics session) distinct.

Each source failure has one `sdk.failure` record containing `error`; operation/flow results reference it through `failureRef`. Count those failure records rather than every failed result or warning. Order by `sequence` only within the same `streamId`; do not infer a global sequence across live logs and retained evidence. A missing field means unavailable information, not a zero measurement or confirmed cause.

Evidence queries return `SDKLogEvidencePageV2`, including `records`, `page`, `summary`, `retention`, and available session identities. The first normal call returns retained evidence without warm-up. Enable `debug.avSync` in advance for continuous evidence; querying does not start collection. Empty or truncated evidence does not prove normal playback.

Query rules match `getAvSyncRawTelemetry()`: `pageSize` is an integer from 1 to 1000 (default 200). Continue with `page.nextCursor` and omit `range` on that request. Do not reuse cursors after the diagnostics session changes. Invalid queries reject with `RTC_INVALID_AV_SYNC_TELEMETRY_QUERY`; query failures must not be reported as an empty successful result. Retained services become inactive after uninstall or Client disposal, so stop plugin-owned polling in `dispose()`.

V1 remains supported. Observe one real-time log version per upload destination to avoid uploading the same facts twice. Moving to V2 also requires updating plugin field selection and the receiving platform; upgrading the SDK alone does not update external plugins.

### 4.2 Individual Metrics and First-frame Timeline

The SDK provides performance measurements. `setPerformanceMetricReporter()`, `PerformanceMonitorOptions.reporter`, and `PerformanceMetricReporter` now retain signatures only and have no effect; they will be removed in 3.0.0. Callbacks are ignored and default `[performance-metric]` Console output is removed. `performanceMonitor.enabled` still controls measurement. This is a behavior change from the old callback path.

Use `SDK_PERFORMANCE_METRIC_V1` (`performance.metric`, major 1) for individual `PerformanceMetricRecord` values without an envelope. It covers stage timings and conversation response metrics when source events occur. The implementation validates names, finite nonnegative durations, finite ordered timestamps, and optional `questionId` / `attemptId`; it does not validate optional monitoring IDs. Delivery is detached and never awaited by the SDK. A dispatch clones and freezes one snapshot shared by its observers; handler timeout is 30 seconds and faults disable the contribution. Plugins own upload, authentication, retry, persistence, and deduplication.

**Known contract/implementation discrepancy:** Previously documented guarantees of an independent snapshot per observer and a queue limit of 100 with drop-oldest overflow are not enforced by the current detached dispatch implementation: it schedules microtasks and shares the same frozen snapshot within a dispatch. This differs from the documented Plugin API v1 isolation and overflow semantics. Plugins relying on those guarantees face a compatibility risk; this clarification does not change the extension-point identity, contract version, or SDK behavior. Track and resolve compatibility separately before changing the implementation or contract.

```ts
import { SDK_PERFORMANCE_METRIC_V1, type SDKPlugin } from "@sanseng/liveavatar-js-sdk";

export const measurementPlugin: SDKPlugin = {
  id: "com.example.measurements",
  apiVersion: 1,
  setup(registrar) {
    registrar.observe(SDK_PERFORMANCE_METRIC_V1, {
      id: "individual-metrics",
      handler(metric, { signal }) {
        if (signal.aborted) return;
        console.log(metric.clientInstanceId, metric.monitoringSessionId, metric.metric, metric.durationMs);
      },
    });
  },
};
```

Connection stages, connection failure/first-frame timeout, text responses, and text-to-audio responses use monotonic elapsed time for `durationMs`. The Unix timestamp `startedAt` anchors the measurement, and `endedAt = startedAt + durationMs`, retaining fractional milliseconds. System clock jumps cannot inflate or negate these durations or reverse their timestamp ordering. A successful first-frame batch uses the same endpoint as its first-frame metric.

After system clock adjustment, `endedAt` may differ from `Date.now()` at completion. Use `durationMs` for elapsed time; the endpoint is not a separately sampled wall-clock fact, and independent measurements may have differently adjusted anchors. Operational Log occurrence timestamps retain their existing semantics. The legacy no-speech metric still subtracts the event-provided wall-clock start from wall-clock completion time, remains vulnerable to clock adjustment, and has no identified production emitter or monotonic origin.

Conversation audio timings do not confirm the exact time sound becomes audible. `no_speech_report_to_audio_response_ms` is not guaranteed in normal sessions; do not replace a missing value with zero. Existing output-stall detection remains in `getPlaybackStatus()` with unchanged rules. Pending conversation measurements are cleared on explicit teardown and new attempts.

Use `SDK_PERFORMANCE_METRICS_V1` for terminal first-frame batches; its identity and schema are unchanged. First-frame measurements also appear in the individual stream; avoid counting both streams as independent measurements. Each accepted payload has schema `sdk.performance-metrics`, schemaVersion `1`, an increasing `attemptId`, optional `clientInstanceId`/`monitoringSessionId`, outcome `first-frame`, `connect-failed`, or `first-frame-timeout`, and metrics only for stages that actually occurred. Records are cumulative from the effective `connect()` start: configuration ready, RTC connected, video track added, video metadata, and first-frame `loadeddata`; failure and timeout have their own terminal metric. Missing stages are omitted, never reported as zero.

```ts
import { SDK_PERFORMANCE_METRICS_V1, type SDKPlugin } from "@sanseng/liveavatar-js-sdk";

export const performancePlugin: SDKPlugin = {
  id: "com.example.performance",
  apiVersion: 1,
  setup(registrar) {
    registrar.observe(SDK_PERFORMANCE_METRICS_V1, {
      id: "write-performance",
      handler: (report, { signal }) => {
        if (signal.aborted) return;
        for (const metric of report.metrics) {
          setPerformance(metric.metric, metric.durationMs, report.attemptId);
        }
      },
    });
  },
};
```

The point is detached with a 30-second handler timeout and failure isolation. The current dispatch shares one deep-cloned frozen snapshot among handlers and does not enforce a bounded detached queue; see the compatibility discrepancy above. Do not mutate it, await it from SDK code, or treat it as an upload acknowledgement. The plugin owns transport, authorization, batching, retries, persistence, sampling, and deduplication.

Use the sanitized `SDK_OPERATIONAL_LOGS_V1` event `performance.first-frame` when a backend needs a timeline or `reportFrameErrors` input. Its top-level envelope carries `clientInstanceId`/`monitoringSessionId`; correlation is only `{ attemptId }`. Attributes are only outcome, render mode, timeoutAfterRtcMs for a timeout, and sparse numeric nodes. The attributes deliberately exclude URL, token, connection configuration, room/session/track/participant identifiers, text, raw DataChannel data, media frames, DOM/LiveKit objects, Error, message, and stack.

SkyWalking plugin 1.1.0 uses legacy extension points. A separate plugin migration is required for the unified-session contracts added in 2.7.0; an SDK upgrade alone does not change its registrations.

### 4.3 Conversation text Transform

`SDKInboundText` contains:

| Field                     | Meaning                                                                                |
| ------------------------- | -------------------------------------------------------------------------------------- |
| `kind`                    | `'answer'`, `'server-message'`, or `'asr'`.                                            |
| `phase`                   | `'partial'` or `'final'`.                                                              |
| `questionId` / `sequence` | The conversation and optional protocol sequence number.                                |
| `rawDelta`                | The actual current answer chunk; `''` for completed answers, server messages, and ASR. |
| `rawText`                 | The accumulated answer, or the complete server-message / ASR text.                     |

The handler must synchronously return `SDKTextProjection`:

```ts
import { SDK_CONVERSATION_INBOUND_TEXT_V1, type SDKPlugin } from '@sanseng/liveavatar-js-sdk';

export const plainTextProjectionPlugin: SDKPlugin = {
  id: 'com.example.plain-text-projection',
  apiVersion: 1,
  setup(registrar) {
    registrar.transform(SDK_CONVERSATION_INBOUND_TEXT_V1, {
      id: 'plain-preview',
      handler: ({ rawText }) => ({
        mediaType: 'text/plain',
        value: rawText.slice(0, 120),
        metadata: { truncated: rawText.length > 120 },
      }),
    });
  },
};
```

`mediaType` must be non-empty, and `value` plus optional `metadata` must be JSON-safe. Equal `mediaType` values from separate plugins or contributions are retained in successful install/setup order in the public Conversation event `representations` array.

This point is `sync + parallel + fail-open`: a thenable result, throw, or invalid projection disables that contribution, while source text, the Conversation FSM, other contributions, and public events continue.

### 4.4 AV Sync diagnostics and alerts

Consume the CoreService during `setup()` and perform asynchronous upload work in an Observer handler:

```ts
import { SDK_AVSYNC_ALERTS_V1, SDK_AVSYNC_DIAGNOSTICS_V1, type SDKPlugin } from '@sanseng/liveavatar-js-sdk';

export const avSyncUploadPlugin: SDKPlugin = {
  id: 'com.example.avsync-upload',
  apiVersion: 1,
  setup(registrar) {
    const diagnostics = registrar.consume(SDK_AVSYNC_DIAGNOSTICS_V1);

    registrar.observe(SDK_AVSYNC_ALERTS_V1, {
      id: 'upload-raised-alert',
      handler: async (alert, { signal }) => {
        if (alert.phase !== 'raised' || signal.aborted) return;
        const telemetry = await diagnostics.getAvSyncRawTelemetry({ pageSize: 200 });
        if (signal.aborted) return;
        await uploadFromPlugin({ alert, telemetry, signal });
      },
    });
  },
};
```

The diagnostics service offers:

- `getPlaybackStatus()` — live status; poll only when needed, roughly once per second without overlapping calls.
- `captureRemoteAvSyncDiagnostics()`
- `getAvSyncRawTelemetry(query?)`
- `getAvSyncAnalysis(query?)`

Use playback status for a live panel, immediate capture for a current paired A/V snapshot, raw telemetry for retained evidence export, and analysis for a historical window. The service has the same debug requirements as the client APIs and returns frozen defensive copies. See [API selection and metric availability](./SDK_User_Manual.md#86-playback-status-and-diagnostics-selection).

```ts
// Inside setup(registrar), consume once; invoke later from plugin-owned work.
const diagnostics = registrar.consume(SDK_AVSYNC_DIAGNOSTICS_V1);
const readPlaybackStatus = () => diagnostics.getPlaybackStatus();
```

Core playback metrics need no debug. Only `presentationFps` requires `debug.avSync`; RTT/ICE fields and browser freezes remain optional. `outputFps` measures completed SDK output; SDK stalls use a one-second output-gap rule. Raw/analysis queries do not collect missing history. Plugins must cancel their own polling in `dispose()`.

AV Sync snapshots, raw telemetry, analysis reports, alerts, session-ended events, and `avSyncLogs` include `clientInstanceId` and `monitoringSessionId` when monitoring is active. Diagnostics `sessionId` remains the RTC session key. Preserve both when uploading so a single monitoring session can be reconstructed across multiple diagnostics sessions and config rounds.

Operational `sdk.error` records include `attributes.code` and optional string `attributes.reason` copied from `error.message` (including empty strings). Missing/non-string messages omit `reason`; Error objects and stacks remain excluded.

Plugin uninstall aborts active asynchronous Transform/Provider work, and handlers should observe every execution context `signal`. Calling a retained CoreService proxy after uninstall throws `SDK_PLUGIN_CONTEXT_INACTIVE`.

## 5. Installation, hot swap, and faults

Applications can install a plugin at construction through `ClientOptions.plugins` or later through `client.installPlugin(plugin)`. Both paths use the same transaction.

- Execution order is successful plugin installation order, then setup registration order; there is no priority.
- A current observe, transform, or provider invocation retains its registry membership snapshot; new installs affect later operations and never replay past events. Uninstall can invalidate a contribution before it starts, even when it appears in the captured snapshot.
- Uninstall atomically removes all contributions, drops callbacks that have not started, aborts active asynchronous Transform/Provider work, invalidates retained registrars/CoreService proxies, and then calls `dispose()`.
- `SDKPluginInstallResult` rejection reasons include duplicate IDs, unsupported API version, invalid plugin, setup failure, invalid contribution, and Provider conflict. Rejecting one plugin does not affect SDK construction or installed plugins.
- `sdk:pluginFault` includes only plugin/contribution identity, optional point descriptor, phase, reason, and timestamp. It never includes Error, stacks, tokens, business text, raw protocol payloads, or private plugin data, and it is excluded from Operational Log observation.

### 5.1 Terminal Observer handoff

Since **2.6.0**, set `terminalObserverDelivery: 'sync-before-dispose'` on `SDKPlugin` to opt in. Before uninstalling the plugin during Client disposal, the SDK synchronously starts one enqueue-ordered batch of pending runnable detached Observer callbacks. It can include ordinary logs and terminal facts already produced. Each callback starts at most once; callbacks already running or disabled by a fault are not replayed. The option is captured at installation.

Normal delivery remains detached. Without this option, unstarted callbacks are discarded during disposal. Explicit uninstall never flushes. Serial queues, callbacks added during handoff, and later asynchronous facts are excluded. Existing snapshots, freezing, fault isolation, timeouts, and execution-signal cancellation still apply.

Client `dispose()` stays synchronous and does not await handler Promises, upload, or persistence. Receive or enqueue each record before the handler's first `await`. Use plugin-owned upload resources and cancellation, independent of `context.signal` and SDK services that may already be unavailable. Keep synchronous handlers short. Handoff does not guarantee network delivery, execution after page exit, or crash recovery.

Do not install/uninstall plugins from handoff callbacks: installation returns `setup-failed` and uninstall returns `false`. Fault notifications may no longer reach application listeners during disposal. Require SDK **2.6.0** or later when using handoff; an older SDK ignoring the option does not implement it. Plugin API remains `1`. Plugins using the unified-session points in Section 4.0 require **2.7.0** or later; the older minimums below remain valid for plugins using only older capabilities.

This example serializes uploads

This example serializes uploads and captures resources per installation. Its finalizer is attached even when there is no final batch, and an older drain cannot release a replacement installation. Supply a transport with plugin-owned authentication, bounded network timeouts, and a failure policy; replace the empty catch with your own handled retry/persistence policy if required.

```ts
import {
  SDK_OPERATIONAL_LOGS_V1,
  type SDKOperationalLogRecord,
  type SDKPlugin,
} from '@sanseng/liveavatar-js-sdk';

interface UploadTransport {
  upload(batch: readonly SDKOperationalLogRecord[]): Promise<void>;
  close(): void;
}

export function createUploadPlugin(createTransport: () => UploadTransport): SDKPlugin {
  let stopCurrent: (() => void) | undefined;
  return {
    id: 'com.example.terminal-upload',
    apiVersion: 1,
    terminalObserverDelivery: 'sync-before-dispose',
    setup(registrar) {
      const transport = createTransport();
      const queue: SDKOperationalLogRecord[] = [];
      let chain = Promise.resolve();
      let sealed = false;
      const appendBatch = () => {
        const batch = queue.splice(0);
        if (batch.length > 0) {
          chain = chain.then(() => transport.upload(batch)).catch(() => {
            // Handle this installation's upload failure here.
          });
        }
      };
      const timer = setInterval(appendBatch, 1000);
      const stop = () => {
        if (sealed) return;
        sealed = true;
        clearInterval(timer);
        appendBatch();
        void chain.finally(() => transport.close()).catch(() => undefined);
      };
      stopCurrent = stop;
      try {
        registrar.observe(SDK_OPERATIONAL_LOGS_V1, {
          id: 'enqueue',
          handler: (record) => {
            if (!sealed) queue.push(record);
          },
        });
      } catch (error) {
        // A failed setup must release resources it allocated itself.
        stop();
        throw error;
      }
    },
    dispose() {
      const stop = stopCurrent;
      stopCurrent = undefined;
      stop?.();
    },
  };
}
```

Create one plugin object per concurrently active Client. Sequential uninstall/reinstall may reuse it because each setup owns a distinct queue, chain, timer, and transport. Test an empty queue, an empty terminal batch with an upload in flight, upload rejection, and reinstall before the old drain settles.

## 6. Currently supported plugin packages

| Plugin package | Purpose                                                                                                                                                          | Plugin API | Status               |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | -------------------- |
| `@sanseng/liveavatar-js-sdk-markdownrender` | Markdown rendering. See the plugin package documentation for exports, configuration and compatibility. | — | Listed plugin package |
| `@sanseng/liveavatar-js-sdk-skywalking` | SkyWalking integration; reviewed 1.1.0 consumes legacy contracts. See its own documentation for upload coverage. | 1.1.0 | Separate migration required for unified-session contracts |

SDK upgrades do not change a plugin’s registrations or collector schema. The SDK does not own SkyWalking authorization, queueing, retries, or delivery; migrate those separately before retiring legacy contracts.

## 7. Release checklist

- Import from the SDK package root and declare the SDK as a peer dependency.
- Keep `setup()` synchronous and use unique contribution IDs inside the plugin.
- Handle `AbortSignal` in Observer/async handlers so upload failures never affect core SDK paths.
- Return JSON-safe Conversation projections and never mutate source input.
- Do not collect or upload tokens, text bodies, raw DataChannel payloads, media frames, or Error stacks that the SDK deliberately excludes.
- Test installation, rejection results, uninstall, reinstall, disposal, and `sdk:pluginFault` against the target SDK version.
