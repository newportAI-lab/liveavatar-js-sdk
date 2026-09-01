# Facemarket Live Avatar SDK Plugin Integration Guide (v2.2.0)

This guide is for **plugin developers**. It explains how to build, package, and maintain a plugin using the first public `@sanseng/liveavatar-js-sdk` Plugin API v1.

If you only need to install an existing plugin in an application, see the “Plugin Import and Configuration” section of the [SDK User Manual](./SDK_User_Manual.md). No standalone plugin package is currently published; see “Currently Supported Plugin Packages.”

## 1. Scope and boundaries

Plugin API v1 is the first public contract for trusted, in-process TypeScript extensions. A plugin can register contributions only against versioned extension points defined by the SDK and exported from the package root, or consume a read-only CoreService.

Plugins are not a security sandbox and must not:

- define extension points or consume unexported internal tokens;
- replace RTC, transport, the Conversation FSM, MessageAssembler, media ownership, or AV Sync collection;
- obtain SDKClient, EventBus, LiveKit Room, tokens, HTTP controllers, or internal controllers;
- rely on inter-plugin communication or plugin-provided Providers; v2.2.0 exports no business Provider token.

The SDK retains ownership of protocol parsing, connection state, conversation source text, and media pipelines. Plugins own their HTTP, authorization, queues, retries, persistence, sampling, deduplication, and drain policy.

## 2. Package layout and dependencies

Declare the SDK as a peer dependency so a plugin does not embed a second copy of the SDK:

```json
{
  "peerDependencies": {
    "@sanseng/liveavatar-js-sdk": "^2.2.0"
  }
}
```

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
| `setup(registrar)` | Must complete synchronously. Returning a Promise/thenable, throwing, or registering an invalid contribution rolls the whole install back. |
| `dispose()`        | Optional; called on uninstall and SDK disposal. A throw does not interrupt the SDK but emits a sanitized `sdk:pluginFault`.               |

The `registrar` is valid only while `setup()` runs. Do not retain it for later registration or service consumption; calls after setup throw `SDK_PLUGIN_CONTEXT_INACTIVE`.

Every contribution needs an ID unique **within the plugin** and a handler. Installation validates point kind, handler, contribution ID, and Provider conflicts before committing everything atomically.

## 4. Available extension points

All tokens are exported from the `@sanseng/liveavatar-js-sdk` package root. A token’s `{ id, major, kind }` is stable identity: compatible additions keep the same major and breaking changes receive a new token major.

| Token                              | Type                                                | Purpose and execution semantics                                                                                                                                                   |
| ---------------------------------- | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SDK_OPERATIONAL_LOGS_V1`          | `ObserverPoint<SDKOperationalLogRecord>`            | Observes sanitized SDK operational logs. Detached and non-blocking; records exclude tokens, Authorization, text bodies, raw DataChannel payloads, media frames, and Error stacks. |
| `SDK_CONVERSATION_INBOUND_TEXT_V1` | `TransformPoint<SDKInboundText, SDKTextProjection>` | Adds representations for inbound answers, system prompts, and ASR. Synchronous, parallel, fail-open; cannot change source text, protocol data, or the FSM.                        |
| `SDK_AVSYNC_ALERTS_V1`             | `ObserverPoint<AvSyncAlertEvent>`                   | Observes the AV Sync alert lifecycle. Detached; installing an observer does not start analysis.                                                                                   |
| `SDK_AVSYNC_ANALYSES_V1`           | `ObserverPoint<AvSyncAnalysisReport>`               | Observes one result for each real single-flight analysis.                                                                                                                         |
| `SDK_AVSYNC_SESSION_ENDED_V1`      | `ObserverPoint<AvSyncSessionEndedEvent>`            | Observes session end only after local segment sealing and worker segment close.                                                                                                   |
| `SDK_AVSYNC_DIAGNOSTICS_V1`        | `CoreServicePoint<SDKAvSyncDiagnosticsService>`     | Consumes diagnostics snapshots, raw telemetry, and analysis queries; results are defensive copies and an old proxy becomes inactive after uninstall.                              |

The generic `ProviderPoint` runtime exists, but v2.2.0 exposes no business Provider token. Plugins therefore cannot currently call `registrar.provide()` to supply SDK-used business behavior.

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

### 4.2 Conversation text Transform

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

### 4.3 AV Sync diagnostics and alerts

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

- `captureRemoteAvSyncDiagnostics()`
- `getAvSyncRawTelemetry(query?)`
- `getAvSyncAnalysis(query?)`

Plugin uninstall aborts active asynchronous Transform/Provider work, and handlers should observe every execution context `signal`. Calling a retained CoreService proxy after uninstall throws `SDK_PLUGIN_CONTEXT_INACTIVE`.

## 5. Installation, hot swap, and faults

Applications can install a plugin at construction through `ClientOptions.plugins` or later through `client.installPlugin(plugin)`. Both paths use the same transaction.

- Execution order is successful plugin installation order, then setup registration order; there is no priority.
- A current observe, transform, or provider invocation retains its Registry snapshot. Install or uninstall affects only the next operation and never replays past events.
- Uninstall atomically removes all contributions, drops callbacks that have not started, aborts active asynchronous Transform/Provider work, invalidates retained registrars/CoreService proxies, and then calls `dispose()`.
- `SDKPluginInstallResult` rejection reasons include duplicate IDs, unsupported API version, invalid plugin, setup failure, invalid contribution, and Provider conflict. Rejecting one plugin does not affect SDK construction or installed plugins.
- `sdk:pluginFault` includes only plugin/contribution identity, optional point descriptor, phase, reason, and timestamp. It never includes Error, stacks, tokens, business text, raw protocol payloads, or private plugin data, and it is excluded from Operational Log observation.

## 6. Currently supported plugin packages

| Plugin package | Purpose                                                                                                                                                          | Plugin API | Status               |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | -------------------- |
| None           | No standalone plugin package is published or registered for v2.2.0. Exported SDK extension points are development capabilities, not installable plugin packages. | —          | No available package |

When an actual package is released, add its name, purpose, compatible SDK/Plugin API versions, installation command, and its own documentation link here.

## 7. Release checklist

- Import from the SDK package root and declare the SDK as a peer dependency.
- Keep `setup()` synchronous and use unique contribution IDs inside the plugin.
- Handle `AbortSignal` in Observer/async handlers so upload failures never affect core SDK paths.
- Return JSON-safe Conversation projections and never mutate source input.
- Do not collect or upload tokens, text bodies, raw DataChannel payloads, media frames, or Error stacks that the SDK deliberately excludes.
- Test installation, rejection results, uninstall, reinstall, disposal, and `sdk:pluginFault` against the target SDK version.
