# Facemarket Live Avatar SDK v1 → v2 Upgrade Guide

> **Supported path**: upgrade `@sanseng/liveavatar-js-sdk` from **v1.3.2** to **v2.2.0**.
>
> **Audience**: Web application integrators, backend owners of LiveKit/DataChannel traffic, and teams that use conversation events, microphone levels, or A/V diagnostics.

This guide identifies the validation and recommended application changes for the upgrade. v2.2.0 retains the principal v1.3.2 connection configuration and compatible conversation-event aliases. Do not change a backend solely because of the historical, short-lived participant-filter rule from v2.0.0.

## 1. Upgrade summary

| Area | Code change required? | Migration result |
| --- | --- | --- |
| Direct / Auth connection configuration | No | The mode-specific contracts of `connectConfig`, `setAuthToken()`, and `updateConnectionConfig()` are unchanged. Regression-test the existing mode. |
| DataChannel sender identity | No, but integration verification is required | v2.2.0 accepts DataChannel messages from both `renderer_*` and `coordinator_*` identities. This upgrade does not require a backend prefix change. |
| Conversation event text fields | Recommended | Legacy fields continue to work, but new code should use `rawDelta` / `rawText` and `representations`. |
| Disconnect and error handling | Yes, if the application branches on error codes | Passive disconnect handling belongs to `sdk:disconnected`; established-session runtime faults use `LIVEKIT_DISCONNECTED`. |
| Microphone-level thresholds | Yes, if the application consumes the value | The level is now local PCM linear RMS, not a LiveKit private field or a synthetic fallback. |
| A/V Sync and plugins | No | These are optional v2 capabilities; adopt their APIs only when the capability is needed. |

## 2. Prepare the upgrade

1. Archive the production dependency, lockfile, and v1.3.2 integration-test result so the release can roll back by restoring the original lockfile and package version.
2. Upgrade the package and reinstall dependencies:

   ```bash
   npm install @sanseng/liveavatar-js-sdk@2.2.0
   ```

3. Run the validation matrix in section 7 using production-equivalent browsers, Direct/Auth configuration, LiveKit rooms, and backend participant identities.
4. Do not depend on SDK internals, private LiveKit fields, or SDK-created media elements; they are not upgrade-compatible contracts.

## 3. Connection and backend protocol

### 3.1 Keep the existing Direct or Auth configuration

Direct mode still receives `sfuUrl` and `userToken` from the caller. To replace an expiring token, call `updateConnectionConfig()` before the next `preConnect()` / `connect()` or `reconnect()`. Auth mode still receives room configuration from the backend; provide `authToken` before the first connection and use `reconnect()` to fetch refreshed configuration.

This upgrade does not require moving between modes. Keep the existing restriction that Auth mode cannot call `updateConnectionConfig()`.

### 3.2 Verify DataChannel traffic

The current SDK accepts `RoomEvent.DataReceived` traffic from both of these remote identities:

- `renderer_*`
- `coordinator_*`

An existing v1.3.2 backend may therefore keep its `coordinator_*` conversation/protocol sender. During integration testing, verify that the identity used in production reaches question/answer, ASR, and server-message events. Messages from all other identities continue to be ignored by the SDK.

## 4. Application-code migration

### 4.1 Move conversation handlers to canonical text fields

v2 retains the legacy fields for v1.3.2 compatibility, but TypeScript marks them as deprecated. Move new code to canonical fields now so a future major removal of aliases does not require another migration.

| Event | Compatible v1 field | Recommended v2 fields |
| --- | --- | --- |
| `conversation:answer:chunk` | `chunk` | `rawDelta`, `rawText`, `representations` |
| `conversation:answer:completed` | `fullAnswer` | `rawText`, `representations` |
| `conversation:server:message` | `message` | `rawText`, `representations` |
| `conversation:asr:chunk` / `conversation:asr:received` | `text` | `rawText`, `representations` |

```ts
// v1: still runs, but the field is deprecated.
client.events.on('conversation:answer:chunk', ({ chunk }) => {
  appendAnswer(chunk);
});

// v2: use the streaming delta, complete text, and optional plugin projections.
client.events.on('conversation:answer:chunk', ({ rawDelta, rawText, representations }) => {
  appendAnswer(rawDelta);
  cacheCompleteAnswer(rawText);
  renderRepresentations(representations);
});
```

In v2.2.0, `chunk === rawDelta`, `fullAnswer === rawText`, `message === rawText`, and ASR `text === rawText`. `representations` is a JSON-safe projection array ordered by plugin registration; it may be empty when no text-transform plugin is installed.

### 4.2 Update disconnect and error handling

`sdk:disconnected` remains the lifecycle notification for a passive disconnect, and its `reason` supports troubleshooting. Do not use `sdk:error` as the only disconnect trigger.

In v2.2.0:

- `LIVEKIT_CONNECT_FAILED` remains for Room connection and existing operation-failure paths.
- When a successfully connected Room ends because of an unrecoverable runtime fault (transport, media, protocol, Agent, or SIP trunk), the SDK additionally emits `sdk:error` with `LIVEKIT_DISCONNECTED`.
- Ordinary session endings such as server termination, duplicate identity, room close/deletion, migration, or user rejection emit `sdk:disconnected` without an additional `sdk:error`.

```ts
client.events.on('sdk:disconnected', ({ reason, allConnected }) => {
  if (!allConnected) scheduleReconnect(reason);
});

client.events.on('sdk:error', ({ code, message }) => {
  if (code === 'LIVEKIT_DISCONNECTED') reportRuntimeRtcFault(message);
});
```

### 4.3 Recalibrate microphone-level thresholds

`getMicrophoneAudioLevel()` still returns `number | null`, but its value is now the linear RMS of the latest 4096-sample local PCM frame:

- It returns `null` with no active local microphone track.
- It returns `0` before the first frame after capture starts, and after capture/frame emission stops.
- It returns a measured `0..1` RMS value when frames exist; it no longer reads LiveKit's private `_volume` or returns a fixed `0.5`.

If v1 code used `0.5` as a microphone-available or silence threshold, first handle `null`, then recalibrate its threshold against measured RMS values.

## 5. Optional v2 capabilities

### 5.1 A/V Sync diagnostics

A/V Sync does not turn on merely because the SDK is upgraded. Background sampling requires `debug.avSync: true` or `debug.avSync.enabled: true`.

```ts
const client = createClient({
  // ...existing connection, audio, and video options
  debug: { avSync: { enabled: true, intervalMs: 2000, historyLimit: 60 } },
});
```

`getAvSyncRawTelemetry(query?)` and `getAvSyncAnalysis(query?)` are asynchronous. Await raw telemetry; each call returns one page, then use `page.nextCursor` for another page:

```ts
const telemetry = await client.getAvSyncRawTelemetry({ pageSize: 200 });
const older = telemetry.page.nextCursor
  ? await client.getAvSyncRawTelemetry({ cursor: telemetry.page.nextCursor })
  : null;
```

For `sdk:avSyncAlert`, retain active incidents by opaque `alertId`: both `raised` and `updated` remain active, while only `recovered` removes the incident.

### 5.2 Plugin API v1

Plugin API v1 is the first public plugin contract in v2; there is no published v1 plugin format to convert. Only when extending logging, conversation text, or A/V Sync should an application add an `SDKPlugin` through `ClientOptions.plugins` or `installPlugin()`. See the full contract in the [Plugin Integration Guide](./SDK_Plugin_Integration_Guide.md).

### 5.3 Runtime green-screen updates

`updateGreenScreenOptions()` can update chroma key, thresholds, smoothing, despill, and `isBackgroundKeying` after connection. Do not pass `enabled` to switch between `raw` and `processed` at runtime: switching from raw to processed can stall shared remote A/V playback. Set `video.renderMode` and `video.greenScreen.enabled` when the instance is created, or have Auth provide them before connecting.

## 6. Pre-release validation checklist

| Scenario | Passing condition |
| --- | --- |
| Installation and type check | The application installs, builds, and starts with v2.2.0 and no longer references internal modules or private fields. |
| Direct connection | Existing URL/token connects, disconnects, and reconnects; a refreshed token reaches the target Room. |
| Auth connection | `setAuthToken()`, initial connection, and backend configuration refresh through `reconnect()` succeed. |
| DataChannel | The production `coordinator_*` or `renderer_*` sender reaches conversation, ASR, and server-message events. |
| Conversation UI | Canonical and compatible fields behave as expected; new code consumes `rawDelta` / `rawText`. |
| Remote media | Audio and video play together, and track replacement, disconnect, and reconnect do not stall or duplicate binding. |
| Microphone | Capture starts in a user gesture; `null`, initial `0`, and measured RMS thresholds match business expectations. |
| Disconnect handling | Ordinary session endings reach `sdk:disconnected`; serious runtime faults can be identified as `LIVEKIT_DISCONNECTED`. |
| Optional A/V Sync | When enabled, paged results are awaited and alert state is cleared by `alertId` only on `recovered`. |
| Green screen | Rendering mode is selected before creation/connection; runtime changes exclude `enabled`. |

## 7. Rollback strategy

Keep the v1.3.2 lockfile or explicit package version during the rollout. If connection, protocol, or media regression fails the checklist, stop expanding v2.2.0 traffic and restore the v1.3.2 dependency and matching build artifact. Do not work around the issue by calling private SDK APIs or forcing changes on internal media elements. Collect `sdk:disconnected.reason`, `sdk:error.code`, browser version, and backend participant identity before targeted investigation.

## 8. Related documentation

- [SDK User Manual](./SDK_User_Manual.md)
- [SDK 使用手册](./SDK_使用手册.md)
- [Plugin Integration Guide](./SDK_Plugin_Integration_Guide.md)
- [插件接入文档](./SDK_插件接入文档.md)
- [CHANGELOG](https://github.com/newportAI-lab/liveavatar-js-sdk/blob/main/CHANGELOG.md)
