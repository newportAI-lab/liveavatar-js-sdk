# Facemarket 实时数字人 SDK 使用手册（v2.8.0）

本手册对应 npm 包 `@sanseng/liveavatar-js-sdk` **2.8.0** 版本。SDK 基于 **LiveKit Client**，封装数字人音视频下行、麦克风/摄像头上行、会话文本 与 HTTP 控制面（鉴权模式下获取连接配置）。

---

## 1. 概述

SDK 对外提供统一入口 **`createClient` → `SDKClient`**，负责：

- 音视频轨道的订阅与路由、本地采集与发布
- 会话侧文本与协议消息的编解码与分发
- 连接生命周期与快照（便于重连与排障）
- 可选绿幕与性能指标上报

使用方通过构造参数、连接 API、事件订阅即可完成集成，无需依赖 SDK 内部模块（如 `transport` 实现细节）。

---

## 2. 升级说明

从 1.x 升级到 **2.8.0**，请阅读 [SDK 1.x → 2.8.0 升级指南](./SDK_v1到v2升级指南.md)，按连接、事件、麦克风和性能上报完成改造。指南以 v1.3.2／v1.3.3 为主要基线，并列出更早 1.x 的额外检查项。历史验证范围仍为 v1.3.2／v1.3.3 → v2.3.0；文档覆盖至 2.8.0 不代表所有路径已完成真实集成验收。当前 2.8.0 为待发布版本。

### 从 2.7.x 升级到 2.8.0

- `startAudioCapture()` 只等待麦克风启用并通过 LiveKit 发布，不再自动创建本 SDK 的采样图。需要音量时显式调用并等待 `startMicLevel()`，随后用同步只读的 `getMicLevel()` 读取；不用时调用 `stopMicLevel()`。旧 `getMicrophoneAudioLevel()` 是兼容别名，也需要显式开启检测，已弃用并将在 3.x 移除。图标依据启动结果和 `isAudioCapturing`，不依据音量是否可用。
- `startMicLevel()` 成功后首次读取返回真实 RMS，静音为 `0`；检测未开启、无活动麦克风、准备中或不可用为 `null`。采样失败不停止推流；取消等待的 `AbortError` 和非致命 `AUDIO_INPUT_MONITORING_UNAVAILABLE` 应独立处理。检测可提前登记，但不要先等待检测再执行开麦。
- 停止采集、断连和销毁立即撤销旧启动操作的权限；取消的启动 Promise 可正常完成但不保证已开麦，竞争时检查当前状态。SDK 回收迟到音轨并隔离旧资源清理；采样恢复不会重新开麦或切换设备。详见本地麦克风章节。
- 浏览器权限失败经结构化证据分类为 `permission` 后，其操作日志从 `error` 降为 `warn`；公开错误和 Promise 拒绝保持原行为。仅接收 error 的消费者应纳入 warn，详见日志章节。

### 从 2.6.x 升级到 2.7.0

- Direct 新增可选 `sessionId`。首次配置成功解析/应用后，`client.sessionId` 从原先的 `undefined` 变为调用方 ID 或带 `sdk-` 前缀的自动标识；依赖该 getter 始终为空的业务应调整判断。仅调用 `updateConnectionConfig()` 暂存不会改变当前值，详见第 5.2 节。
- 日志 V3、性能 V2、AV Sync Observer V2 和诊断 CoreService V3 为可选新扩展点，最低要求 SDK **2.7.0**，Plugin API 仍为 **1**。旧扩展点保留；选择一个事实流版本，避免重复上传。SDK 升级不会自动迁移插件、采集 schema 或查询。

### 从 2.5.x 升级到 2.6.0

- `performanceMonitor.reporter` 和 `setPerformanceMetricReporter()` 仍可传入或调用，但不再执行回调；默认 `[performance-metric]` Console 输出也已移除。已有性能上传应迁移到第 14.2 节的 `SDK_PERFORMANCE_METRIC_V1` 插件，旧签名将在 3.0.0 移除。
- 日志级别按 Client 隔离，创建或销毁其他 Client 不会覆盖已有实例的 `debug` 设置。日志 V2 和销毁前回调交接为可选能力，使用方式见第 7.5–7.6 节及插件接入指南。

---

## 3. 初始化说明

SDK 通过 `ClientOptions.connectConfig` 区分两种互斥模式：**Direct（直连）** 与 **Auth（鉴权）**。二者在配置来源、`setAuthToken` / `updateConnectionConfig` 的可用性及 `reconnect` 刷新行为上不同。

### 2.1 Direct Mode（直连模式）

**描述**
由调用方在构造参数中直接提供 LiveKit 所需的 `sfuUrl` 与 `userToken`。SDK 不通过 HTTP 拉取房间配置。

**前置条件**

- 必须提供非空的 `sfuUrl`（LiveKit WebSocket URL）与 `userToken`（房间访问令牌）。
- 若业务在运行中更换房间或令牌，必须在下次重连前通过 `updateConnectionConfig` 注入新配置（见行为说明）。

**初始化方式**

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

**行为说明**

- `preConnect()`：解析并缓存当前 Direct 配置或暂存的替换配置，不建立 LiveKit 连接。`connect()` 使用有效缓存或解析配置后建立连接。
- `reconnect()`：断开当前连接后使用最新暂存的 Direct 配置重新连接，不通过鉴权 HTTP 获取房间配置。
- 不会自动「刷新」服务端签发的 token；令牌过期须由业务侧获取新令牌并调用 `updateConnectionConfig` 后再 `preConnect()` / `connect()` 或 `reconnect()`。

**使用限制**

- 不支持在 Auth 模式下调用 `updateConnectionConfig`（将抛出 `SDK_INVALID_STATE_TRANSITION`）。
- 不支持在 Direct 模式下依赖 HTTP 接口返回的 `videoOptions` 覆盖策略；绿幕等视频参数以构造时的 `video` 与运行时 `setRenderFitMode` 等为准（除非业务自行在服务端约定后写入本地配置）。

**适用场景**
已有稳定签发 LiveKit token 的后端、私有化部署、或调试阶段快速接入。

---

### 2.2 Auth Mode（鉴权模式）

**描述**
调用方提供 `avatarId`（及可选的 `authToken`、`avatarVoice`）。SDK 通过 HTTP 获取鉴权与房间配置，并应用服务端返回的绿幕等 `videoOptions`。

**前置条件**

- 必须提供非空的 `connectConfig.config.avatarId`。
- 必须在 `preConnect()` / `connect()` 能访问到有效鉴权令牌：通过 `connectConfig.config.authToken` 传入，或在实例化后、连接前调用 `setAuthToken(token)` 写入上下文。
- 必须提供可用的 HTTP 服务（默认或自定义 `http.baseURL`），用于获取鉴权与 LiveKit 房间配置。

**初始化方式**

```ts
import { createClient } from '@sanseng/liveavatar-js-sdk';

const client = createClient({
  connectConfig: {
    type: 'auth',
    config: {
      avatarId: 'your-avatar-id',
      avatarVoice: 'optional-voice-id',
      // authToken 也可省略，改为稍后 client.setAuthToken('...')
    },
  },
  http: {
    baseURL: 'https://your-api.example.com/...',
    headers: {
      /* 可选 */
    },
  },
  video: {
    containerElement: document.getElementById('avatar')!,
  },
});

client.setAuthToken('jwt-or-business-token');
```

**行为说明**

- `preConnect()`：通过 HTTP 获取并缓存连接配置，不建立 LiveKit 连接。`connect()` 使用有效缓存或重新获取配置，再建立连接。
- `reconnect()`：断开后**重新通过 HTTP** 获取最新房间配置。
- `setAuthToken()`：更新后续鉴权和配置请求使用的令牌，不清除已缓存的房间配置。若必须立即取得新配置而当前状态不允许 `reconnect()`，应销毁旧实例并用新令牌创建实例。

**使用限制**

- 不支持在 Auth 模式下使用 `updateConnectionConfig` 手动替换 LiveKit URL/Token（应通过服务端刷新会话与 `reconnect()` 重新拉取）。
- `connectConfig.config.authToken` 与 `setAuthToken` 为同一语义数据源；须保证在首次 `connect()` 前至少一侧有效。

**适用场景**
生产环境由业务后端统一鉴权、统一下发房间参数与绿幕等视频策略。

---

### 2.3 模式对比

| 项目                     | Direct Mode                                                                               | Auth Mode                                              |
| ------------------------ | ----------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| 配置来源                 | 构造参数中的 `sfuUrl`、`userToken`                                                        | HTTP 接口返回的 `ConnectionConfig`                     |
| 是否需要业务 HTTP        | 否（仅当同时使用其他 HTTP 能力时可选配 `http`）                                           | 是（获取鉴权与房间配置）                               |
| 必填字段                 | `sfuUrl` + `userToken`                                                                    | `avatarId` + 有效 `authToken`（构造或 `setAuthToken`） |
| `setAuthToken`           | 不用于解析 LiveKit 连接配置                                                               | 必须或建议在连接前注入                                 |
| `updateConnectionConfig` | 可用；作用于**下一次** `preConnect()` / `connect()` 或 `reconnect()` 所使用的 Direct 配置 | 不可用（抛出错误）                                     |
| `reconnect()` 配置刷新   | 使用最新暂存的 Direct 配置          | 重新通过 HTTP 获取                       |

---

## 4. 快速开始

### 4.1 安装

```bash
npm install @sanseng/liveavatar-js-sdk
```

### 4.2 最小流程（Auth 示例）

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

// 必须由业务后端签发后注入，此处省略具体请求实现
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

// 用户点击后再开麦（满足浏览器自动播放 / 采集策略）
document.getElementById('mic')?.addEventListener('click', async () => {
  await client.startAudioCapture();
});
```

### 4.3 最小流程（Direct 示例）

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

## 5. 核心概念

### 5.1 questionId（问答关联标识）

`questionId` 是**一次完整问答对**（用户提问 → 服务端处理 → 流式/完整回答）的关联键，用于串联：

- `conversation:question:sent`
- `conversation:answer:waiting`
- `conversation:server:message`
- `conversation:asr:received` / `conversation:asr:chunk`
- `conversation:answer:chunk`
- `conversation:answer:completed`

**注意**：SDK 生成的 `questionId` 是短随机字符串，未进行会话内查重，可能碰撞；不可作为唯一键或长期持久化主键，业务需使用自己的标识并考虑碰撞对关联的影响。

**强烈建议**：在收到 `conversation:answer:completed` 后，清理该 `questionId` 对应的本地缓冲（文本拼接、UI 状态等），避免内存增长与 ID 复用带来的展示错误。

### 5.2 会话关联（自 2.7.0 起）

本节可选 Direct `sessionId` 与统一会话扩展点自 SDK **2.7.0** 起提供；使用新扩展点的插件须将 SDK 最低版本设为 **2.7.0**。Plugin API 仍为 **1**。

Direct 的 `connectConfig.config` 与 `updateConnectionConfig()` 接受 `sessionId?: string`。字符串先去除首尾空白；未传、`undefined`、空串、全空白或运行时 `null` 时，SDK 自动生成带 `sdk-` 前缀的标识。后缀优先使用 `crypto.randomUUID()`，不可用时使用时间戳与随机串回退；调用方应将其视为不透明字符串，不解析后缀格式。其他非字符串值导致 `SDK_PRECONNECT_FAILED`。该标识用于业务关联，不是鉴权凭据，不会被写入仅属于 Auth 的 `serverSessionId`。

```ts
const client = createClient({
  connectConfig: {
    type: 'direct',
    config: { sfuUrl, userToken, sessionId: 'business-session-42' },
  },
});
// 配置真正成功解析/应用前为 undefined；无需先建立 RTC 连接。
await client.preConnect();
console.log(client.sessionId); // 'business-session-42'
client.updateConnectionConfig({ sfuUrl, userToken, sessionId: 'next-session' });
// 仅暂存，getter 与当前连接保持原值；下次配置应用后才切换。
```

新输出的 `SDKSessionCorrelation` 为 `{ clientInstanceId, sessionId, sessionEpoch }`。同一个 Client 的 epoch 从 **1** 开始，每个新生命周期递增且不复用。业务提供的 `sessionId` 可以重复；使用 `clientInstanceId + sessionEpoch` 隔离生命周期，并保留 `sessionId` 作业务关联，不能单凭它作为持久化唯一键。`attemptId` 继续只表示一次首帧测量。

- Direct 比较规范化后的 `sfuUrl`、`userToken` 及调用方 `sessionId`；三者相同、缓存命中、refresh 及内部 reconnect 延续生命周期。仅 ID 变化也会在配置应用时开启新 epoch；仅暂存不会切换。
- 显式 `disconnect()` 结束当前生命周期；再次准备配置开启新 epoch，自动 ID 随之新建，调用方 ID 可保持相同。被动掉线及内部重连拆卸本身不结束生命周期。
- Auth 实际配置获取开始时生成带 `sdk-` 前缀的临时标识，使用相同的 UUID 优先及回退规则；自动重试、并发复用请求不切换。成功返回服务端 ID 后在同一 epoch 绑定；仅新日志 V3 发布 `session.id-bound`，顶层 `sessionId` 为新 ID，`data` 仅携带 `previousSessionId`。缺少服务端 ID 时新输出保留本地 ID。Auth 的 `client.sessionId` 仍遵循原服务端 ID getter 契约。
- 异步工作和媒体资源保存所属 epoch 的快照；迟到结果、旧遥测和历史字段不会从当前会话补填，避免跨生命周期串联。

分阶段迁移：日志选择 `SDK_OPERATIONAL_LOGS_V3`（`sdk.log` / schemaVersion 3），性能选择 `SDK_PERFORMANCE_METRIC_V2` 或 `SDK_PERFORMANCE_METRICS_V2`（批次为 `sdk.performance-metrics` / 2），AV Sync 选择 `SDK_AVSYNC_ALERTS_V2`、`SDK_AVSYNC_ANALYSES_V2`、`SDK_AVSYNC_SESSION_ENDED_V2` 和 `SDK_AVSYNC_DIAGNOSTICS_V3`。新契约使用应用 `sessionId + sessionEpoch`，媒体诊断 ID 单独命名 `avsyncSessionId`，不输出 `monitoringSessionId`。详见插件指南。

旧日志 V1/V2、性能 V1、AV Sync 旧扩展点、Client 直接诊断 API 和公开事件保留原字段、schema、时序与含义：`monitoringSessionId` 仍关联旧输出，旧 AV Sync `sessionId` 仍是媒体诊断 ID。日志 V3 的顶层应用 `sessionId` 表示当前本地/绑定 ID，媒体 `avsyncSessionId` 独立；不重复输出 Auth `serverSessionId`。Plugin API 仍为 **1**。

当前 SkyWalking 插件 **1.1.0** 只使用旧契约；继续使用匹配的旧扩展点，SDK 升级不代表插件或采集端已迁移。后续须分别迁移插件、采集 schema 和查询，再计划在 **3.0** 清理旧 `monitoringSessionId` 出口；本次不执行该清理。

### 5.3 控制面与媒体面

- **媒体面**：通过 LiveKit 连接房间、订阅与发布音视频，并通过 **Data Channel** 传输会话文本与协议消息。
- **控制面（Auth）**：通过 HTTP 获取 `livekitUrl`、`token`、`roomId` 及服务端可选的 `videoOptions`（如绿幕开关）。**不存在**与 LiveKit 并行的独立「业务 WebSocket」通道对外 API。

### 5.4 视频渲染模式

- `video.renderMode === 'raw'`：直接在 `<video>` 上渲染远端轨道（默认类行为以选项为准）。
- `video.renderMode === 'processed'`：经 Canvas 处理管线（如绿幕）。
- **Auth 模式**下，服务端绿幕配置可映射为 `renderMode` / `greenScreen`。

---

## 6. 配置说明（`createClient` 构造参数）

`createClient(options: ClientOptions)`，`ClientOptions` 定义见 `src/types/ClientOptions.d.ts`。

### 6.1 顶层字段

| 字段                 | 类型                        | 必填 | 说明                                                                                    |
| -------------------- | --------------------------- | ---- | --------------------------------------------------------------------------------------- |
| `connectConfig`      | `direct \| auth` 联合       | 是   | 连接模式与模式专属配置                                                                  |
| `audio`              | `AudioOptions`              | 否   | 输入/输出音频选项                                                                       |
| `video`              | `VideoOptions`              | 否   | 渲染容器、绿幕、`renderMode`、`fitMode` 等                                              |
| `reconnect`          | `{ maxAttempts?, delay? }`  | 否   | LiveKit 重连：`maxAttempts` 默认 `3`；`delay` 默认 `10`（**秒**），内部会换算为毫秒上限 |
| `http`               | `{ baseURL?, headers? }`    | 否   | Auth 与 HTTP 控制接口根路径及默认头                                                     |
| `performanceMonitor` | `PerformanceMonitorOptions` | 否   | 默认开启；`enabled: false` 可关闭                                                       |
| `sandbox`            | `boolean`                   | 否   | 沙箱开关（依业务后端约定）                                                              |
| `debug`              | `boolean \| DebugOptions`  | 否   | 可选 Debug、LiveKit verbose 与 A/V 诊断配置；详见第 7 章「Debug 与 A/V 诊断」。          |
| `plugins`            | `readonly SDKPlugin[]`      | 否   | 构造期安装的插件列表。传入已从插件包导入的 `SDKPlugin`；单个安装失败不影响客户端构造。详见[插件接入指南](./SDK_Plugin_Integration_Guide.md)。 |

本章仅列出构造参数。Debug 日志、A/V 诊断、自动视频停滞告警和实验性 frame metadata worker 的完整配置及行为说明见第 7 章。

### 6.2 `connectConfig`

**Direct**

| 字段               | 类型       | 必填 |
| ------------------ | ---------- | ---- |
| `type`             | `'direct'` | 是   |
| `config.sfuUrl`    | `string`   | 是   |
| `config.userToken` | `string`   | 是   |
| `config.sessionId` | `string`   | 否；自 2.7.0 起，去除首尾空白，省略则自动生成 |

**Auth**

| 字段                 | 类型     | 必填                                   |
| -------------------- | -------- | -------------------------------------- |
| `type`               | `'auth'` | 是                                     |
| `config.avatarId`    | `string` | 是                                     |
| `config.authToken`   | `string` | 否（可与 `setAuthToken` 二选一或组合） |
| `config.avatarVoice` | `string` | 否                                     |

### 6.3 `video`（`VideoOptions`）

| 字段               | 说明                                                                  |
| ------------------ | --------------------------------------------------------------------- |
| `containerElement` | 视频渲染挂载的容器（**业务侧应保证存在且已插入 DOM**）                |
| `renderMode`       | `'raw'` \| `'processed'`                                              |
| `greenScreen`      | `{ enabled, chromaKey?, similarity?, smoothness?, despillStrength?, isBackgroundKeying? }`；`isBackgroundKeying` 默认 `false` |
| `fitMode`          | `'contain' \| 'cover' \| 'fill' \| 'none'`                            |
| `camera`           | `{ publishToLiveKit?: boolean }`                                      | 默认 `false`；为 `true` 时 `startCamera()` 将轨道发布到 LiveKit 房间。 |
| `debug`            | 继承 `BaseOptions`                                                    |

### 6.4 `audio`（`AudioOptions`）

| 字段     | 说明                                                                                                                                                                                                                                                          |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `input`  | `{ deviceId?: string, sampleRate?: number, channelCount?: number, sampleSize?: number, noiseSuppression?: boolean, voiceIsolation?: boolean, bitDepth?: number, echoCancellation?: boolean, autoGainControl?: boolean, constraints?: MediaTrackConstraints }` |
| `output` | `{ enabled?: boolean, volume?: number, muted?: boolean }` 播放侧默认值                                                                                                                                                                                        |

**输入优先级**：顶层显式字段覆盖 `constraints` 中的同名成员；其他标准 constraint 成员原样保留并传给 `getUserMedia`。`sampleSize` 正式传入采集请求；`bitDepth` 是仅在未设置 `sampleSize` 时使用的已弃用别名，最后回退到 `constraints.sampleSize`。`noiseSuppression`、`echoCancellation`、`autoGainControl` 只有在顶层字段和 `constraints` 都未设置时才默认为 `true`，不会覆盖显式 `false`。本地 PCM 处理在没有可用数值时使用 `channelCount` 默认值 `1`。

采集请求成功不代表设备接受了全部约束；实际值以 `getMicrophoneStats().settings` 返回的浏览器设置为准。

### 6.5 `performanceMonitor`

| 字段       | 说明                                                   |
| ---------- | ------------------------------------------------------ |
| `enabled`  | 默认 `true`；`false` 关闭内置性能采集                  |
| `reporter` | `(metric: PerformanceMetricRecord) => void`；**已无作用，配置被忽略，回调不执行；3.0.0 移除** |

### 6.6 `MicrophoneStats`

| 字段            | 说明             |
| --------------- | ---------------- |
| `bytesSent`     | 已发送字节数；缺失时兼容填 `0`，需结合 `availability.bytesSent` 判断 |
| `packetsSent`   | 已发送数据包数；缺失时兼容填 `0`，需结合 `availability.packetsSent` 判断 |
| `packetsLost`   | 丢失的数据包数，可为负值；缺失时兼容填 `0`，需结合 `availability.packetsLost` 判断 |
| `roundTripTime` | **旧字段**：LiveKit outbound RTP 的原始 RTT 秒值，为兼容保留；缺失时为 `0`。新代码请使用 `roundTripTimeMs` |
| `roundTripTimeMs` | 由原始秒值换算的往返时间（毫秒）；不可用时为 `null` |
| `availability`  | 各统计字段的有效证据标记：`bytesSent`、`packetsSent`、`packetsLost`、`roundTripTimeMs`。旧字段为 `0` 且标记为 `false` 表示缺失；`0` 且为 `true` 表示真实零 |
| `settings`      | 浏览器实际 `MediaStreamTrack.getSettings()` 归一化后的 `MicrophoneTrackSettings`；无法读取时为 `null` |
| `sampledAtMs`   | 查询时间（`performance.now()`）。每次调用都重新读取发送统计与 track settings，不缓存 |

`MicrophoneTrackSettings` 包含可空的 `deviceId`、`groupId`、`sampleRate`、`channelCount`、`sampleSize`、`echoCancellation`、`noiseSuppression`、`autoGainControl`、`voiceIsolation`。无活动麦克风、无 sender report 或发送统计请求失败时，`getMicrophoneStats()` 仍返回 `null`；仅 settings 读取失败不会影响统计返回。

---

## 7. Debug 与 A/V 诊断

### 7.1 基础 Debug 与 LiveKit verbose

SDK 普通 Console 日志按 Client 隔离：`debug` 为 `false` 或省略时该实例默认使用 `ERROR`，`true` 或对象（包括 `{}`）默认使用 `DEBUG`。创建或销毁其他 Client 不改变该实例级别。

显式调用 `Logger.setLevel()` 会立即影响调用时已有的 Client；之后新建的 Client 按自身 `debug` 初始化。`Logger.setPrefix()` 统一设置 SDK Console 日志前缀。

`debug: { verbose: true }` 开启 LiveKit Debug 日志。同一 SDK 模块副本内，只要仍有未销毁的 Client 请求 verbose，LiveKit 就保持 `debug`，否则为 `info`；断开和重连不会取消该设置，`dispose()` 会释放。LiveKit 日志级别为全局设置，可能影响同页其他 LiveKit 实例；独立 SDK 副本之间不协调。

### 7.2 周期 A/V 诊断（`debug.avSync`）

通过以下配置在 LiveKit Room 连接成功后按周期采样远端音频/视频接收端统计；默认不轮询：

```ts
debug: {
  avSync: { enabled: true, intervalMs: 2000, historyLimit: 60 },
}
```

后台采样最多等待 10 秒以接收通过 renderer 参与者过滤的首个远端音频或视频轨道；超时后停止当前连接会话的后台采样，并在上传日志中记录 `remote-track-wait-timeout`。该超时不影响手动 `captureRemoteAvSyncDiagnostics()`。

离线证据经 `getAvSyncRawTelemetry(query?)` 按页返回，不再输出 `[RTC remote-media]` Console 行；无参只返回最近 5 分钟、最多 200 条，使用 `range`、`pageSize` 和 `page.nextCursor` 读取更多记录。telemetry Worker 可用时，日志查询和事实摘要在 Dedicated Worker 执行；创建时无 Worker 则使用内存回退。主动 capture 仍在浏览器主线程读取当前 WebRTC stats。`avSyncLogRetention` 返回保留量、存储错误和丢弃量；Room 断开会冻结最后会话供分页查询，重连替换它，`dispose()` 和连接建立失败会清除它。

分页契约：时间范围包含两端；`recent.durationMs` 必须为正有限数，wall-clock 端点必须有限且 `fromMs <= toMs`。`pageSize` 必须是 1–1000 的整数，默认 200。非法参数、空/损坏 cursor、跨会话 cursor、cursor 与 range 同传均拒绝为 `SDKError`，code 为 `RTC_INVALID_AV_SYNC_TELEMETRY_QUERY`；没有当前或冻结会话时仍校验参数。

第一页 cursor 固定会话、时间范围和 sequence 上界，续页允许调整 pageSize；`hasMore: true` 一定附带可用 `nextCursor`，末页为 false/null。`summary` 统计完整查询窗口内且不超过该上界的所有可用日志，不受当前页影响；`includeSummary: false` 返回 null 并跳过聚合。`currentDiagnosis` 是本次调用的当前诊断上下文。`availableFrom/ToWallClockMs` 描述当前会话上界内保留日志的可用边界；`truncated` 反映 retention 对窗口的淘汰，不代表正常分页。后续新增日志不会进入旧 cursor 的页或摘要，但 retention 淘汰可能减少后续可读证据，SDK 不保留服务器式窗口快照。

telemetry Worker 与创建时无 Worker 的回退共用上述分页算法。LocalStore 负责已封口分段的留存预算和淘汰；Worker 查询这些保留分段及未封口 tail 的增量内存镜像，不等待持久化，也不回读 IndexedDB。待写入或写入失败的证据仍可存在于镜像中；存储状态反映请求时已知的信息，不代表待完成的写入已成功。无 Worker 时仅查询回退实际拥有的内存证据，不具备同等持久化历史覆盖。运行中的原始遥测或证据查询失败会向上传递，不自动降级；分析 Worker 不可用或镜像证据不完整时，分析返回 `insufficient_evidence`。此 telemetry Worker 与第 7.4 节的实验性 frame metadata worker 是不同组件。

### 7.3 自动视频停滞告警

启用 `debug.avSync` 后，无需轮询 `getAvSyncAnalysis()` 即可收到自动视频流水线异常。共享 `<video>` 已触发 `playing` 后，SDK 会在连续 3 个兼容周期样本确认后，将接收、解码和浏览器呈现停滞分别归类为 `video_receive`、`video_decode` 和 `video_render`。`video_render` 仅在浏览器支持 `requestVideoFrameCallback`、已有呈现基线且页面可见时检测；SDK 不会自动重连、重订阅或重启播放。

自动告警是一个连续的问题生命周期：首次确认时发送 `phase: "raised"`；同一已确认分类持续时保持活跃，但刻意不重复派发事件或 `video-pipeline-stalled` 日志；同一视频轨道确认变为另一 receive/decode/render 分类时发送 `phase: "updated"`；只有受影响的接收、解码或呈现层出现直接正向进展时才发送 `phase: "recovered"`。因此，没有新的告警事件或留存日志**不表示已经恢复**。

每个生命周期 payload 都带有不透明 `alertId`。同一活跃问题的 `raised`、`updated` 和最终 `recovered` 保持相同 ID。`updated` 提供 `previousFinding`，并可能改变 `finding`；业务应保留该问题直至收到 `recovered`，不可用“未出现新告警”作为正常信号：

```ts
const activeAlerts = new Map<string, AvSyncAlertEvent>();

client.events.on(`sdk:avSyncAlert`, (alert) => {
  if (alert.phase === "recovered") {
    activeAlerts.delete(alert.alertId);
    return;
  }

  // raised 与 updated 都表示问题仍处于活跃状态。
  activeAlerts.set(alert.alertId, alert);
});
```

所有告警阶段发送给稳定公开 `sdk:avSyncAlert`，并可由已安装插件接收；插件接入方式见[插件接入指南](./SDK_Plugin_Integration_Guide.md)。只有首次 `raised` 会写入留存的 `video-pipeline-stalled` 事实证据。SDK 不会自动上传告警或日志。只监听公开事件、不轮询 `getAvSyncAnalysis()`，即可覆盖上述自动周期观测 transition；但 `getAvSyncAnalysis()` 仍负责轨道绑定、传输退化、音频 concealment、播放和相对漂移等完整保留窗口的手动分析。此类 finding 只有业务实际调用分析 API 后才可能变为告警。

### 7.4 实验性 Frame Metadata Worker

周期 A/V 统计和 TimeSync 默认不会启用实验性的 frame metadata worker。如需启用，必须同时开启周期诊断并设置 `frameMetadata: true`：

```ts
debug: {
  avSync: { enabled: true, frameMetadata: true },
}
```

开启它的作用是为**远端视频帧**补充可关联的 frame metadata：SDK 会请求 LiveKit frame metadata worker，并在上游已声明相应 packet-trailer feature、轨道支持 metadata lookup 且存在可关联的 TimeSync RTP timestamp 时，把 `frameId`、`userTimestamp` 和 `userData` 与接收端基线、视频呈现事件关联。这样可以在 `getAvSyncRawTelemetry()` 中获得有界的 `frameMetadata` 样本，以及带 metadata 的 TimeSync/接收侧证据，便于排查“同一发布端视频帧从接收到实际呈现”的链路。

如果 `userData` 是 ASCII 十进制的发布端 Unix 微秒级捕获时间戳，且 metadata 关联新鲜、没有重复帧，SDK 还会填充视频帧的 `videoCaptureToDisplayRawMs` 与相对变化值，用作 capture-to-display 诊断证据。`userData` 不符合该格式时会原样保留，相关时延字段保持 `null`；SDK 不会据此虚构时间戳或精确时延。

该选项**不是**常规 WebRTC receiver stats、周期采样、TimeSync、基于 RTP 的相对漂移时间线，或 `video_receive` / `video_decode` / `video_render` 自动告警的前置条件；关闭后这些能力仍可工作，只是不再收集逐帧 metadata 和 capture-to-display 证据。metadata 仅适用于远端视频，并依赖浏览器/worker 能力、上游 feature 声明、轨道 lookup 和 TimeSync RTP 关联；任一条件缺失时，遥测会保留不可用或未观察到的事实，相关 metadata 字段为 `null` 或为空。

`userData` 会以受限的 Base64 原样写入诊断遥测，启用前应确认其中不包含不应进入客户端诊断数据的业务信息。如 worker 初始化或运行失败，SDK 会以 `[RTC frame-metadata-worker]` 前缀在 Chrome DevTools Console 中提示，并将 worker 状态保留为诊断日志事实；普通 LiveKit 连接和基础 A/V 诊断仍会继续工作，诊断 API 不会因此抛出异常。

### 7.5 插件引入与配置

插件开发、扩展点和运行语义见[插件接入指南](./SDK_Plugin_Integration_Guide.md)。使用插件时，从其自身包导入 `SDKPlugin` 对象；将下面占位符替换为“当前已支持插件包”目录中实际插件提供的包名和导出名。

```ts
import { createClient } from '@sanseng/liveavatar-js-sdk';
import { plugin } from '<plugin-package>';

const options = {
  connectConfig: {
    type: 'direct',
    config: { sfuUrl: 'wss://example.test', userToken: 'token' },
  },
} as const;

// 二选一：构造期安装
const configuredClient = createClient({ ...options, plugins: [plugin] });

// 二选一：在未安装该 ID 的实例上运行期安装
const runtimeClient = createClient(options);
const result = runtimeClient.installPlugin(plugin);
if (result.status === 'rejected') {
  console.warn(result.pluginId, result.reason, result.message);
}

runtimeClient.uninstallPlugin(plugin.id);
```

同一插件 ID 不应同时出现在 `plugins` 与同一实例的 `installPlugin()` 中；运行期安装前请检查结果。`uninstallPlugin()` 返回 `true` 表示已卸载，`false` 表示当前实例中不存在该 ID。

#### 销毁期 Observer 交接

自 **2.6.0** 起，插件可设置 `terminalObserverDelivery: 'sync-before-dispose'`，在 Client 销毁前同步接收当时尚可执行的待处理 Observer 回调，包括已产生的会话结束日志。普通业务继续调用 `client.dispose()` 即可；不使用插件时无需增加处理。

SDK 不等待上传或持久化完成。插件应在回调首次 `await` 前接收或入队数据，并在自身 `dispose()` 中使用自有上传资源完成后续处理；不能依赖销毁后的 SDK 服务。此选项不改变正常回调和显式 `uninstallPlugin()` 的行为，也不保证页面退出后上传成功。

使用该能力的插件应将 SDK peer dependency 最低版本设为 **2.6.0**，Plugin API 仍为 `1`。完整约定与示例见[插件接入指南](./SDK_Plugin_Integration_Guide.md#51-terminal-observer-handoff)。

#### 当前已支持插件包

| 插件包 | 用途 | 状态 |
| --- | --- | --- |
| `@sanseng/liveavatar-js-sdk-markdownrender` | Markdown 渲染；导出、配置和兼容版本见插件包文档。 | 已列入插件目录 |
| `@sanseng/liveavatar-js-sdk-skywalking` | SkyWalking 集成；已核对的 1.1.0 使用旧 SDK 契约，上传覆盖范围以插件文档为准。 | 新会话契约需独立迁移插件 |

SkyWalking 1.1.0 不消费本次新增的会话契约；插件与采集端须分别迁移。SDK 不拥有插件的鉴权、队列、重试或上传。

---

### 7.6 运行日志与诊断证据

SDK 不自动上传日志。业务通过插件接收并发送到自己的平台；SDK 升级不会自动改变已有插件的上报内容。

| 插件能力 | 用途 | 最低 SDK 版本 |
| --- | --- | --- |
| `SDK_OPERATIONAL_LOGS_V1` | 继续接收原有运行日志 | 保持原插件兼容范围 |
| `SDK_OPERATIONAL_LOGS_V2` | 接收包含操作结果、错误原因和会话关联的 `SDKLogRecordV2` | `2.6.0` |
| `SDK_AVSYNC_DIAGNOSTICS_V2` | 通过 `getAvSyncLogEvidence(query?)` 按页读取已有音画诊断证据 | `2.6.0` |
| `SDK_OPERATIONAL_LOGS_V3` | 统一应用会话关联的日志 V3（第 5.2 节） | `2.7.0` |
| `SDK_PERFORMANCE_METRIC_V2` / `SDK_PERFORMANCE_METRICS_V2` | 统一应用会话归属的单条/批次性能指标 | `2.7.0` |
| `SDK_AVSYNC_ALERTS_V2` / `SDK_AVSYNC_ANALYSES_V2` / `SDK_AVSYNC_SESSION_ENDED_V2` | 分离应用会话与媒体诊断身份的音画 Observer | `2.7.0` |
| `SDK_AVSYNC_DIAGNOSTICS_V3` | 统一会话关联的音画快照、遥测、分析与日志证据 | `2.7.0` |

V1 继续可用；同一用途选择一个实时日志版本，避免重复上传。V2 通过 `clientInstanceId + monitoringSessionId` 关联监控会话；`serverSessionId` 为服务端会话，`avsyncSessionId` 为音画诊断会话，不应混用。

`getAvSyncLogEvidence()` 由插件通过 `registrar.consume(SDK_AVSYNC_DIAGNOSTICS_V2)` 获取的服务提供，不会启动额外采样；提前启用 `debug.avSync` 才能持续留存诊断证据。首次查询直接返回已有证据，无需预热或重复调用。查询条件和分页方式与 `getAvSyncRawTelemetry()` 一致。接入示例见[插件接入指南](./SDK_Plugin_Integration_Guide.md#41b-structured-logs-and-av-sync-evidence-v2)。

#### 日志 V3 的精简记录

跨实例诊断标识使用无填充 Base64URL 紧凑编码：`clientInstanceId` 为 `c_` 加 22 字符，`monitoringSessionId` 为 `m_` 加 22 字符，本地证据的页面生命周期标识为 `p_` 加 22 字符。SDK 优先完整编码 `crypto.randomUUID()` 的 16 字节；不可用或调用失败时使用 `crypto.getRandomValues()` 获取 16 字节，不截断或哈希 UUID，因此保留原有随机位与碰撞安全水平。两种加密随机源均不可用时，回退为类型前缀、36 进制时间、模块内递增序号和随机串；回退长度不固定为 24 字符，也不具有加密随机源的唯一性保证。随机 ID 不承诺数学上的绝不碰撞。所有这些标识都应视为不透明字符串，不解析其前缀或长度。

跨实例短标识用于所有日志、性能和诊断版本，包含 Client ID 的派生标识随之变短；旧字段、拼接规则与唯一性作用域保持不变，历史记录不改写。调用方、后端、浏览器和 LiveKit 提供的 ID，以及 SDK 自动应用 `sessionId` 的 `sdk-` 规则不变。`failureId`、`alertId` 保留包含所属 Client 的完整身份，支持独立消费；不要从操作短 ID 推导它们。

Log V3 保持 `schema: 'sdk.log'`、`schemaVersion: 3`、`SDK_OPERATIONAL_LOGS_V3` major 3。该格式自 SDK 2.7.0 起提供，当前 SDK 为 2.8.0；旧 V1/V2 的字段与语义不变。

自 2.8.0 起，结构化证据分类为 `permission` 的浏览器权限失败（`NotAllowedError` / `SecurityError`），在 V1 `sdk.error` / `operation.failed` 和 V2/V3 `sdk.failure` 中将原 `error` 级别降为 `warn`，已有较低级别保留。权限不可用不等于用户主动拒绝；HTTP 401/403、LiveKit `NotAllowed` 及其他后端、设备和未知异常沿用原规则，前置条件与已确认取消的分类优先级保持不变。失败记录、错误码、操作结果、公开 `sdk:error` 和 Promise 拒绝不因降级而改变；仅按 error 过滤的消费者应调整为接收 warn，或按失败事件统计。

每条记录包含 `clientInstanceId`、`stream: 'operational' | 'avsync'`、`sequence`、Unix 墙钟 `occurredAtMs`、`level`、`module`、`event`；可选 `sessionId`、`sessionEpoch`、`avsyncSessionId`、`observedAtMs`、`trigger`、`operation`、顶层 `failureId`、`error`、`correlation`、`snapshot` 和 `data`。V3 不输出顶层 `eventId`、`streamId`、`clockId`、`serverSessionId`、`failureRef`，也不在 `error` 内重复 `failureId`。内部 Auth 服务端 ID 和 Auth getter 继续保留原行为。

- Operational 记录去重键为 `[clientInstanceId, 'operational', sequence]`，序号覆盖 Client 生命周期，包括独立的 `sdk.failure`；AV Sync 为 `[clientInstanceId, 'avsync', avsyncSessionId ?? null, sequence]`，沿用每个媒体诊断源的序号。同一空间按序号排序，不跨空间混排。键不含应用 `sessionId`/`sessionEpoch`，Auth 绑定不会改变已有记录身份。生命周期隔离仍使用 Client 加 epoch。
- `occurredAtMs` 表示事实发生的 Unix 墙钟毫秒；可用的 `observedAtMs` 表示主 Client 单调时钟毫秒，Operational 在发布时读取（失败也不回用捕获时刻），AV Sync 保留源观测值。时钟基准由契约隐含，不另加 ID。两类时钟不能相减；发布间隔不是业务耗时，业务耗时使用已有测量字段。
- 一次源失败只有一个携带完整 `error` 的 `sdk.failure`，其顶层 `failureId` 沿用 V2 的同一源身份。相关操作/流程结果以相同顶层 `failureId` 引用它，不再按 eventId 关联或重复错误详情。`error.operation`、`error.phase`、`error.protocolEvent` 可补充 operation 未表达的失败现场。
- `snapshot` 仅保留连接状态：SDK 状态、HTTP/RTC 标志、Room 是否存在/状态、是否曾连接及上次断开。源、时间、身份和原因不再重复，错误证据位于 `error.evidence`。
- SDK 自生成操作在 V3 中使用 `o1`、`o2`、`o10` 等十进制短序号；`operationId`、`rootOperationId`、`parentOperationId` 和两处 `relatedOperationId` 统一投影。完整操作关联键为 `[clientInstanceId, operationId]`，不同 Client 可分别出现 `o1`。序号沿用源操作的 Client 生命周期，不因重连、配置切换、Auth 绑定或插件卸载重装重置；内部源 ID 和 V1/V2 操作拼接规则不变。只有与记录所属 Client 精确匹配的 SDK 操作 ID 会转换，其他值原样保留。操作键用于关联，不替代上述记录去重键。
- `operation.phase` 可省略：事件名已表达的 started/finished/事件名不重复，真实 request/response/init 等阶段保留。根操作不重复自身 `rootOperationId`，子操作仍保留根 ID。operation 与 correlation 的 `relatedOperationId` 相同时只保留 correlation；不同时两者都保留。
- `trigger` 可省略：unknown 和默认 SDK 事件入口省略，真实 API、backend、browser 及有独立含义的 SDK 入口保留。`correlation.mediaStreamId` 表示浏览器 MediaStream ID，区别于旧 V2 的 `correlation.streamId`。
- Auth 的 `session.id-bound` 使用顶层新 `sessionId`，`data` 仅含 `previousSessionId`。不回写早期事实。`sdk.log-evidence` 页面仍为版本 3，`records` 使用上述精简 V3；窗口、游标、首次查询和 measurements 字段名、单位、0/false/null 不变。

这些精简规则仅适用于日志 V3 与其证据记录，不删除性能 V2、AV Sync 告警 V2 或原始遥测中的字段。旧日志 V2 继续使用 eventId、streamId、clockId、serverSessionId 和 failureRef 的旧语义；SkyWalking 1.1.0 仍走旧契约，没有自动启用 V3 上传。

## 8. 核心 API 方法

以下为 `SDKClient` 的公共方法。多数媒体与会话 API 要求 **已成功 `connect()`**；鉴权与连接配置可在连接前设置。

### 8.1 连接与生命周期

| 方法                                       | 说明                                                                                                      |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| `preConnect(): Promise<boolean>`           | 预拉取并缓存连接配置（带 TTL，默认 60s 量级，见 `PRECONNECT_CACHE_DURATION_MS`）。失败抛出错误。          |
| `connect(): Promise<void>`                 | 建立协调器、各域控制器与 LiveKit；内部可在无有效缓存时等价调用预连接路径。                                |
| `disconnect(): Promise<void>`              | 停止采集、断开 LiveKit、结束 HTTP 会话相关清理；实例可再次 `connect()` / `reconnect()`。                  |
| `reconnect(): Promise<ConnectionSnapshot>` | 允许时重建连接，返回触发时的快照；状态不允许时跳过并返回当前快照。成功后应读取当前 `connectionSnapshot` 或连接事件判断状态。 |
| `installPlugin(plugin): SDKPluginInstallResult` | 原子安装 Plugin API v1 插件；拒绝以结果数据返回，不影响 SDK 生命周期。 |
| `uninstallPlugin(pluginId: string): boolean` | 原子卸载全部 contribution、失效旧 handle 并调用插件 dispose。 |
| `dispose(): void`                          | 释放全部资源；之后不得再使用该实例。                                                                      |

**约束**：浏览器策略下，**带声音的播放**与 **麦克风采集** 建议在用户手势（点击等）回调内触发 `connect()` / `startAudioCapture()`。

### 8.2 鉴权与连接配置

| 方法                                                           | 说明                                                                                                                                                                          |
| -------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `setAuthToken(token: string): void`                            | 写入后续 Auth 请求使用的鉴权令牌；不清除已缓存的房间配置。 |
| `updateConnectionConfig(config: DirectConnectionConfig): void` | **仅 Direct**。校验 `sfuUrl` / `userToken` 非空后暂存；自 2.7.0 起另支持可选 `sessionId`（规则见 5.2），**不影响当前已连接会话**；在下次 `preConnect()` / `connect()` 或 `reconnect()` 时生效。 |

### 8.3 媒体

| 方法                                                     | 说明                               |
| -------------------------------------------------------- | ---------------------------------- |
| `setRenderFitMode(mode: RenderFitMode): void`            | 设置画面适配模式。                 |
| `updateGreenScreenOptions(options: Partial<GreenScreenOptions>): void` | 在已连接会话中实时更新绿幕参数；修改仅作用于当前连接。 |
| `startAudioCapture(): Promise<void>` | 只负责启用麦克风并通过 LiveKit 发布，不自动开启 RMS、PCM 或本地可视化。 |
| `stopAudioCapture(): Promise<void>`                      | 停止麦克风发布。                   |
| `setVolume(volume: number): void`                        | 播放音量 `0..1`。                  |
| `getVolume(): number` | 读取播放音量；当前 2.8.0 实现中底层音量为 `0` 时会返回 `1`，不能用此返回值判断静音或真实零音量。 |
| `mute()` / `unmute()`                                    | 播放静音控制，不影响远端音频分析。 |
| `get isMuted`                                            | 是否静音。                         |
| `get isAudioCapturing`                                   | 麦克风是否处于活跃发布状态。       |
| `startMicLevel(): Promise<void>` | 显式开启 RMS 检测；可在开麦前调用并等待后续音轨，真实窗口就绪后完成，不代为开麦。 |
| `getMicLevel(): number \| null` | 同步只读线性 RMS（0..1）；检测未开启、无活动麦克风、准备中或不可用为 `null`，真实静音为 `0`。 |
| `stopMicLevel(): void` | 清除 RMS 需求并取消未完成的开启等待，不影响推流、PCM 或可视化。 |
| `getMicrophoneAudioLevel(): number \| null` | **已弃用，3.x 移除**；兼容转调 `getMicLevel()`，返回值及错误行为相同。 |
| `getMicrophoneStats(): Promise<MicrophoneStats \| null>` | 获取麦克风发送统计和实际 track settings；每次调用重新采样。`availability` 区分真实零与缺失，RTT 毫秒值使用 `roundTripTimeMs`。无活动麦克风、无 sender report 或发送统计失败时返回 `null`。 |
| `isMicrophoneSilent(): Promise<boolean \| null>`         | 检测麦克风是否在发送静音。         |
| `startCamera(options?: { publishToLiveKit?: boolean }): Promise<void>` | 开启摄像头；可选择发布到 LiveKit 房间（默认仅本地预览）。 |
| `stopCamera(): void`                                     | 停止摄像头。                       |
| `getCameraStream(): MediaStream \| null`                 | 本地预览用流。                     |
| `getCameraTrack(): MediaStreamTrack \| null`             | 本地轨道。                         |
| `attachCameraTo(videoElement: HTMLVideoElement): void`   | 将本地摄像头画面绑定到 `<video>`。 |
| `detachCameraFrom(videoElement: HTMLVideoElement): void`  | 将摄像头轨道从视频元素解绑。       |
| `startAnalyserEmission(): void`                         | 开始发送远端音频分析器数据（~60fps）。若远端音频轨道尚未到达，将自动延迟等待轨道订阅成功后开始发送。播放静音不影响分析数据，见下方“远端播放静音与音频可视化”。 |
| `stopAnalyserEmission(): void`                          | 停止发送远端音频分析器数据。 |
| `restartAnalyserEmission(): void`                        | 重启远端音频分析器链路。           |
| `startLocalAnalyserEmission(): void`                     | 开始发送本地麦克风分析器数据（~60fps）。采集前调用会在麦克风启动后生效；停止或重启可视化不影响音量查询与音频帧订阅。 |
| `stopLocalAnalyserEmission(): void`                     | 停止本地麦克风分析器。             |
| `restartLocalAnalyserEmission(): void`                  | 重启本地麦克风分析器链路。         |

### 远端播放静音与音频可视化

调用 `client.mute()` 仅将远端音频播放静音，不会停止远端音频分析。远端音轨正常接收且已开启 `startAnalyserEmission()` 时，`media:audio:analyserData` 仍会提供分析数据，因此远端音频电平、频谱和波形图仍可正常展示，并随实际接收到的音频信号变化。

这些可视化反映接收到的音频信号，不代表扬声器正在发声。需要停止分析数据发送时，调用 `stopAnalyserEmission()`。

### 本地麦克风音量与启动结果

在点击等用户手势内调用 `startAudioCapture()`。方法只等待麦克风启用及 LiveKit 发布；成功后即可更新启用图标。开麦和音量读取不会自动开启本 SDK 的分析图。音量、PCM 和本地可视化是独立消费者：分别由 `startMicLevel()`、真实 `media:audio:frameData` 订阅和 `startLocalAnalyserEmission()` 开启。

```ts
await client.startAudioCapture();
console.log('麦克风已启用', client.isAudioCapturing);

// 只在业务明确需要音量时开启；成功后首次读取即为真实测量。
try {
  await client.startMicLevel();
  console.log(client.getMicLevel()); // 0 是真实静音；不会用固定值掩盖不可用。
} catch (error) {
  // AbortError 表示等待被取消；采样故障不应触发推流清理。
  console.warn('音量检测未就绪', error);
}

client.stopMicLevel(); // 只移除 RMS 消费者；麦克风推流继续。
```

`startMicLevel()` 可在开麦前登记需求并等待后续音轨，但不会申请权限或启动推流。没有音轨期间不创建采样资源、不启动就绪超时；请独立触发开麦，不要先等待检测再执行开麦。音轨可用后，SDK 负责最多约 2 秒的窗口准备，不要求用户说话，也无需预热、固定延时或重复开启。重复开启合并当前准备；检测已就绪时直接成功。

`getMicLevel()` 同步只读，不初始化或恢复采样。检测未开启、无活动麦克风、准备中或不可用时返回 `null`；准备中本身不报告错误。`startMicLevel()` 成功后首次读取为真实 RMS，静音为 `0`。旧 `getMicrophoneAudioLevel()` 仅兼容转调新 getter，已弃用并将在 3.x 移除；迁移名称之外还需显式开启检测。getter 继续保留连接及销毁状态检查。

每个 SDK Client 的三项本地麦克风消费者共用一个按需创建的 `AudioContext` 和输入 source，不与其他 Client、远端分析或 LiveKit 自有 context 共用；各自拥有独立分支。每次添加消费者确保共享资源存在，移除时只释放对应分支；最后一个消费者退出后释放共享图、定时任务及恢复监听。准备中的检测也是消费者。PCM 和可视化不依赖 RMS 开启或就绪，故障及启停不应反过来影响推流。无消费者时，本 SDK 不创建本地分析图；LiveKit 创建音轨时原生的一次性静音检测仍保留，不能据此宣称底层完全没有临时 AudioContext。

音量准备超时或故障使 `startMicLevel()` 以 `AUDIO_INPUT_MONITORING_UNAVAILABLE` 拒绝，并通过既有非致命诊断报告；同一连续故障不重复通知。RMS 分支故障不停止其他健康分支，共享 context 故障才恢复共享图。关闭任意分析功能都不影响已成功发布的麦克风。权限或发布失败仍拒绝 `startAudioCapture()`；用启动结果与 `isAudioCapturing` 更新图标，不能把 `null` 或检测错误当作麦克风已关闭。

### 麦克风停止、取消与采样恢复

- 权限申请或发布尚未结束时，`stopAudioCapture()`、`disconnect()` 和 `dispose()` 立即撤销旧启动操作的使用权。被取消的启动 Promise 正常完成，但不发送麦克风启动事件，也不代表麦克风已启用；与停止操作并发时，以当前 `isAudioCapturing` 为准。
- 浏览器权限弹窗不一定能被关闭。SDK 不等待未结束的权限请求；迟到返回的轨道立即停止且不再发布。发布中的轨道先在本地停止，迟到发布完成后由 SDK 针对旧轨道清理，不影响重新开启的麦克风。
- 已发布麦克风的正常停止仍等待取消发布；取消发布失败继续拒绝 Promise，但 SDK 仍尝试停止本地轨道。连接清理的某一步失败不会跳过其余释放步骤。`dispose(): void` 同步撤销使用权并停止已知采集资源，其余异步释放在后台处理；它不保证网络断开、浏览器或系统设备释放已经完成。
- 仅有明确消费需求且音轨可用时才准备或恢复分析图。暂停时优先恢复原 Context；共享图损坏时，每个连续故障最多自动重建一次，单次准备/恢复最多约 2 秒。仍失败后停止持续重试，后续可信交互或隐藏到可见可触发新的有界尝试，每秒最多一次。分支故障不以重建健康分支为代价；getter 不启动恢复。
- `stopMicLevel()` 清除 RMS 意图并以 `AbortError` 取消尚未完成的开启等待。关麦、断连、音轨结束或替换同样取消当前等待并释放旧图，但保留 RMS 启用意图，后续业务再次开麦时自动恢复；销毁清除意图。已经拒绝的 Promise 不会复活。PCM 订阅和可视化各按自身生命周期处理；诊断恢复不代为开麦或切换设备。成功发布和启用状态不能代替有效音频发送验证。

若 macOS 在退出 Chrome 后仍持续显示麦克风正在使用，记录浏览器/系统版本、实际加载的 SDK/LiveKit 版本、设备、相关进程和指示持续时间。浏览器的 AudioContext 设备/渲染错误、SDK 采样状态、RTC 发布状态及系统音频会话需要分别核对，不能仅凭图标认定 SDK 泄漏或仅凭单元测试认定设备问题已修复。

### 8.4 会话

| 方法                                              | 说明                                                                                  |
| ------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `sendTextQuestion(text: string): Promise<string>` | 发送文本问题；返回 **消息 UID**（实现上用作问答关联，与事件中的 `questionId` 对应）。 |
| `interrupt(): Promise<void>`                      | 发送打断控制事件（`control.interrupt`）。                                             |

### 8.5 状态与可观测性

| 成员                                                                       | 说明                                                                                    |
| -------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `get isConnected(): boolean`                                               | 会话是否处于已连接态。                                                                  |
| `get connectionSnapshot(): ConnectionSnapshot`                             | 同步只读快照：`http.connected`、`rtc.connected`、`rtc.hasVideoTrack`、`overall.state`。 |
| `get sessionId(): string \| undefined`                                     | Auth 保持服务端 ID 契约。自 2.7.0 起，Direct 在首次配置成功解析/应用后返回调用方 ID 或带 `sdk-` 前缀的自动标识，此前为 `undefined`；暂存不改变当前值（见 5.2）。   |
| `setPerformanceMetricReporter(reporter?: PerformanceMetricReporter): void` | **已无作用，3.0.0 移除**；忽略回调、不创建检测器，保留销毁检查。                                                            |
| `getPlaybackStatus(): Promise<PlaybackStatus>` | 当前连接、接收媒体 ICE/带有效期 RTT、各轨道接收指标、SDK 输出 FPS 和停顿；支持单轨，基础指标无需 debug。 |
| `captureRemoteAvSyncDiagnostics(): Promise<AvSyncDiagnosticsSnapshot>`      | 主动采集远端音视频接收统计和本次连接的轨道生命周期节点；未配置 `debug` 也可调用。        |
| `getAvSyncRawTelemetry(query?): Promise<RemoteAvSyncRawTelemetry>`           | 读取当前有界上下文和一页 `avSyncLogs`。默认最近 5 分钟、200 条；通过 `range`、`pageSize` 和 `page.nextCursor` 分页。窗口事实摘要默认启用，可用 `includeSummary: false` 关闭；Worker 可用时日志查询和摘要在 Dedicated Worker 执行，否则使用创建时无 Worker 的内存回退。 |
| `getAvSyncAnalysis(query?): Promise<AvSyncAnalysisReport>` | 在 telemetry Worker 中异步分析完整保留窗口；调用方自行决定轮询周期。返回 `status`、`finding`、`features`、`layers` 和有界限制信息，不会因证据不足抛出异常；自动视频停滞通知不依赖调用此方法。 |
| `get events(): PublicEmitterAPI`                                           | 仅 `on` / `off` / `once`。                                                              |

以下描述适用于保留的直接 API 与旧 AV Sync 契约（新插件契约见 5.2）：诊断快照、原始遥测、分析报告、告警、session-ended 事件和留存诊断日志在有监控会话时都带 `clientInstanceId`、`monitoringSessionId`。诊断 `sessionId` 标识一次媒体诊断会话；旧日志和迟到分析保留原会话关联。`avsync.session.started` 日志通过 `correlation.avsyncSessionId` 关联监控会话与诊断会话。

`getAvSyncAnalysis()` 的计数均是**请求分析窗口内的增量**，不会把浏览器 WebRTC 的累计 receiver counter 当作当前异常量。`features` 中的关键字段如下：

- `audioObservationDurationMs`：有效远端音频 delta interval 的总观测时长。
- `audioConcealedSamplesDelta` / `audioConcealmentEventsDelta`：该窗口内的音频 concealment 增量。
- `audioConcealedSamplesPerSecond` / `audioConcealmentEventsPerSecond`：以上增量除以观测时长；没有有效音频 delta interval 时为 `null`。
- `videoPacketsLostDelta`、`videoFramesReceivedDelta`、`videoFramesDroppedDelta`、`videoFramesDecodedDelta`：该窗口内的远端视频增量；其中 received/decoded 用于区分接收组帧与解码推进情况。

多远端参与者时，全局音频速率的分母是所有有效远端 audio channel 的观测时长之和。`participants` 及告警 participant IDs 只包含本窗口内具有远端媒体证据的身份；本地或仅连接质量相关的参与者不会出现在分析结果中。

### 8.6 播放状态与诊断 API 选择

以下播放状态能力自 **2.5.0** 提供。业务连接与播放面板使用 `getPlaybackStatus()`，其他 API 用于诊断采样、历史证据导出与分析。

| API                                | 主要用途                                                                 | 立即采集统计 | debug 要求                                                       |
| ---------------------------------- | ------------------------------------------------------------------------ | ------------ | ---------------------------------------------------------------- |
| `getPlaybackStatus()`              | 当前连接、ICE/RTT、各轨道丢包率/码率/分辨率/FPS、SDK 输出 FPS/停顿和质量 | 是           | 基础指标不需要；`presentationFps` 需要 `debug.avSync`            |
| `captureRemoteAvSyncDiagnostics()` | 一次音视频配对接收快照、生命周期与 TimeSync 上下文                       | 是           | 无 debug 也可调用；持续 TimeSync 证据需要 A/V 诊断               |
| `getAvSyncRawTelemetry(query?)`    | 导出已保留证据和一页诊断日志                                             | 否           | 连续接收/帧证据需提前开启 `debug.avSync`；帧 metadata 需显式启用 |
| `getAvSyncAnalysis(query?)`        | 分析保留窗口内的接收、解码、渲染、绑定与播放异常                         | 否           | 需要充分证据和分析 Worker，否则返回 `insufficient_evidence`      |

这些 API 共享部分接收统计，但用途不同。播放状态支持单音频、单视频和音视频，区间基线独立于 A/V 诊断采样；即时诊断快照按完整音视频配对组织。原始遥测和分析无法补回未曾采集的历史事实。仅 `avSyncLogs` 分页；其他遥测数组是有界上下文，不一定按相同请求窗口裁剪。断连后可查询冻结保留的证据；重连替换会话，dispose 清除证据。

#### 查询当前播放状态

```ts
// 复用已初始化的 client；建议约每秒串行调用一次，避免重叠请求。
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

- 丢包率和码率需要两个兼容样本；新建或替换 receiver 后重新建立基线。`null`/`unknown` 表示不可用，不等于零或连接失败。
- ICE 统计来自接收媒体所使用的 PeerConnection：单 PC 使用共享 publisher，双 PC 的两种模式均使用 subscriber。TURN 与 UDP 不是互斥选项；`relayProtocol` 表示本地客户端到 TURN 的协议。
- RTT 是客户端到 SFU 的 STUN 往返时间。首次采样建立响应计数基线，计数增加后才提供带年龄的 RTT，5 秒后过期。`rttMeasuredAtMs` 是 SDK 观察时间，不是准确的 STUN 完成时间；重复读取不会刷新未变化的证据。
- `outputFps` 统计不同源帧成功完成 Canvas 绘制/Worker 确认的速率（raw 模式使用受支持的视频帧回调），不是物理屏幕 FPS。SDK 停顿指首帧输出后连续至少 1 秒无新输出，时长不包含每次间隔的首个 1 秒，也不计主动暂停和后台页面；浏览器冻结计数单独保留。
- `quality` 是固定的接收/输出启发式判断，不是端到端质量实测分数；取值为 `disconnected`、`no-media`、`unknown`、`good` 或 `degraded`。
- 仅 `presentationFps` 需要 `debug: { avSync: true }` 或 `debug: { avSync: { enabled: true } }`；单独 `debug: true` 不够。即使开启 debug，ICE 详情、RTT、浏览器冻结统计和展示 FPS 也可能不可用。当前不提供准确单向/端到端延迟、物理屏幕 FPS 或所有用户感知卡顿。
- 连接建立前/拆除后，轨道列表为空；没有支持的活动渲染器时 `output` 可为 null。dispose 前停止轮询，dispose 后调用会拒绝。不采集连接持续时长。

#### 返回字段与单位

| 字段                                                                                     | 含义与单位                                                                    |
| ---------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `sampledAtMs`、`ice.sampledAtMs`、`ice.rttMeasuredAtMs`                                  | `performance.now()` 单调时钟毫秒，不是 Unix 时间                              |
| `ice.rttMs`、`ice.rttAgeMs`、`ice.rttValidForMs`                                         | RTT、观察年龄和有效期，均为毫秒；有效期固定 5000                              |
| `tracks[].packetLossRate`、`tracks[].bitrateBps`                                         | 区间丢包比例 0–1、接收码率 bit/s；不包含 ICE/TURN 总带宽开销                  |
| `tracks[].width`、`tracks[].height`、`tracks[].decodedFps`                               | 解码视频尺寸（像素）、解码帧率；不是 Canvas 尺寸或最终显示帧率                |
| `tracks[].jitterMs`、`tracks[].networkQuality`                                           | RTP jitter（毫秒）、最新远端参与者 LiveKit 质量等级；没有质量事件时为 unknown |
| `tracks[].freezeCount`、`tracks[].freezeDurationSeconds`、`tracks[].intervalFreezeCount` | 浏览器定义的累计冻结次数、累计秒数、区间次数；字段可缺失                      |
| `output.stallCount`                                                                      | 包含正在持续的一次停顿，每次只计一次；轨道替换/渲染模式切换重置               |
| `output.stallDurationMs`、`output.currentStallDurationMs`                                | 已结束停顿的累计时长、当前停顿时长（毫秒），分别返回                          |
| `output.stallState`、`output.stallThresholdMs`                                           | `suspended`/`warming-up`/`flowing`/`stalled`；阈值固定 1000 毫秒              |

没有有效输出样本时 `outputFps` 为 null；确认输出停顿时为 0。raw 模式没有 `requestVideoFrameCallback` 时输出统计不可用。`presentationFps` 是视频元素展示帧率，依赖足够且未超过 3 秒的有效展示证据；它与解码 FPS、SDK 输出 FPS 不同。

质量规则：RTC 未连接为 `disconnected`，没有远端接收轨道为 `no-media`；任一轨道出现 poor/lost 网络质量、区间丢包率 >=5%、新冻结、零接收码率或区间零解码 FPS，或当前 SDK 输出停顿时为 `degraded`。所有轨道都有有效区间丢包率和码率且未触发降级才为 `good`，否则为 `unknown`。`qualityReasons` 提供轨道限定原因，输出停顿使用 `sdk:output-stall`。

维护者可在 SDK 源码仓库的 `docs/architecture/playback-status.md` 查阅完整指标契约、质量阈值和可用性。

#### 按需诊断与历史证据

```ts
// 主动采集当前接收快照；不要求 debug。
const snapshot = await client.captureRemoteAvSyncDiagnostics();

// 分析此前已采集的最近 30 秒证据。
const analysis = await client.getAvSyncAnalysis({
  range: { kind: 'recent', durationMs: 30_000 },
});
console.log(analysis.status, analysis.finding, analysis.confidence, analysis.limitations);

// 导出一页证据；使用 page.nextCursor 获取后续页。
const evidence = await client.getAvSyncRawTelemetry({
  range: { kind: 'recent', durationMs: 30_000 },
  pageSize: 200,
});
```

状态面板不需要同时轮询四个方法：轮询播放状态，按需即时采样，历史分析和证据导出由用户操作或故障处理触发。分析会发布结果并可能更新诊断告警；查询本身不会开启后台采样。

### 8.7 版本信息

| 成员                        | 说明                   |
| --------------------------- | ---------------------- |
| `SDKClient.version: string` | SDK 版本号字符串。     |
| `VERSION: string`           | 直接导出的版本号常量。 |

---

## 9. 公共事件介绍

事件通过 `client.events.on(eventName, listener)` 订阅。事件名**仅**下列白名单，其余名称会抛错。

**连接**

### `sdk:connected`

- **触发时机**：内部 `livekit` 与 `http` 连接事实变化后，聚合状态与上次不同时下发。
- **Payload**：`{ livekit: boolean; http: boolean; all: boolean; livekitConnected: boolean; httpConnected: boolean; allConnected: boolean }`。
  - `livekitConnected` / `httpConnected` / `allConnected` 始终反映**当前**真实连接状态，与 `sdk:disconnected` 完全对称。`auth` 模式：`allConnected === livekitConnected && httpConnected`；`direct` 模式：`allConnected === livekitConnected`。
  - 旧字段 `livekit` / `http` / `all` **已废弃**（保留兼容），下个主版本中将删除。
- **说明**：用于总览两路通道是否同时就绪。推荐新代码使用新字段。

### `sdk:disconnected`

- **触发时机**：任一路径断开事实导致聚合状态变化时。
- **Payload**：`{ livekit, http, all, livekitConnected, httpConnected, allConnected, reason? }`。
  - **新字段** `livekitConnected` / `httpConnected` / `allConnected` 始终反映**当前**真实连接状态，与 `sdk:connected` 完全对称，全断开时三者均为 `false`。推荐新代码使用。
  - **旧字段** `livekit` / `http` / `all` **已废弃**，保留以兼容历史消费方；其在 `sdk:disconnected` 路径上语义反转（断连后报 `true`），将在下个主版本中删除。
  - `allConnected`（与旧 `all`）的模式语义：`auth` 模式为 `livekitConnected && httpConnected`；`direct` 模式为 `livekitConnected`。
  - `reason`：透传自内层 `inner:sdk:disconnected.reason`。
- **说明**：`allConnected === false` 仅表示当前模式的整体连接条件未满足，不表示全部通道均已断开或手动重连一定允许；结合 `connectionSnapshot` 和重连后的当前状态判断。

### `sdk:error`

- **触发时机**：SDK 报告错误或部分能力不可用时。应按 `code` 判断处理方式；`AUDIO_INPUT_MONITORING_UNAVAILABLE` 仅表示本地采样不可用，不代表麦克风推流失败。
- **Payload**：`{ message: string; code: string }`（`code` 为 `ErrorCode` 字符串值）。

### `sdk:pluginFault`

- **触发时机**：插件 Observer/Transform/Provider/CoreService/dispose 发生隔离故障时。
- **Payload**：仅包含 `pluginId`、可选 `contributionId` / `{ id, major, kind }` 扩展点描述、`phase`、`reason` 与 `occurredAtMs`。不会暴露 Error、stack、token、业务正文、原始协议 payload 或插件私有数据；该事件不会进入 Operational Log Observer。

### `sdk:connectionStateChanged`

- **触发时机**：LiveKit 连接状态变更时（断开/连接中/已连接/重连中）。
- **Payload**：`{ state: ConnectionState }`，其中 `ConnectionState` 为枚举值：`disconnected` | `connecting` | `connected` | `reconnecting` | `signalReconnecting`。

### `sdk:avSyncAlert`

- **触发时机**：这是单个 A/V 问题的生命周期：首次确认 attention 证据时发送 `raised`；同一活跃自动视频流水线确认进入另一 receive/decode/render 分类时发送 `updated`；仅受影响层出现直接正向进展时发送 `recovered`。相同故障持续会被刻意去重；没有新事件或日志不等于恢复。`debug.avSync` 下，播放中的视频若连续 3 个兼容周期 RTP 包、接收帧、解码帧均无进展，会自动报告 `video_receive`；确认的解码和呈现停滞还会报告 `video_decode` / `video_render`。仅 `insufficient_evidence` 不会触发告警。
- **Payload**：有界 `AvSyncAlertEvent`，包含 `phase`、稳定不透明 `alertId`、`finding`、`previousFinding`、`status`、`confidence`、`sessionId`、`capturedAtMs`、参与者 ID、英文 message 和紧凑布尔特征。raised 的 `previousFinding` 为 null，updated/recovered 的该字段表示前一异常分类。参与者 ID 仅表示产生 finding 的远端媒体参与者，不会暴露原始日志、媒体对象、本地身份或 connection-only 身份。三个阶段只发送给公开事件与插件 handler；只有首次 raised 新增 retained stall log，SDK 不自动向外上传。

### `sdk:connectionQualityChanged`

- **触发时机**：参与者连接质量变化时。
- **Payload**：`{ quality: ConnectionQuality; participantId: string; isLocal: boolean }`，其中 `ConnectionQuality` 为枚举值：`excellent` | `good` | `poor` | `lost` | `unknown`。

### `sdk:participant:disconnected`

- **触发时机**：身份以 `renderer_` 开头的远端参与者离开 LiveKit 房间时。
- **Payload**：`{ participantId: string }`。

**视频**

| 事件                       | 触发时机                                | Payload                            |
| -------------------------- | --------------------------------------- | ---------------------------------- |
| `media:video:available`    | 远端视频播放可用信号，不保证物理屏幕已呈现首帧 | `undefined`                        |
| `media:video:unavailable`  | 远端视频不可用                          | `undefined`                        |
| `media:video:trackAdded`   | 视频轨道添加                            | `undefined`（内部已做 track 去重） |
| `media:video:trackRemoved` | 视频轨道移除                            | `undefined`                        |

**远端音频**

| 事件                          | 触发时机                 | Payload |
| ----------------------------- | ------------------------ | ------- |
| `media:audio:trackAdded`      | 远端参与者音频轨道订阅   | `undefined` |
| `media:audio:trackRemoved`    | 远端参与者音频轨道取消订阅 | `undefined` |
| `media:audio:speakingChanged` | 远端参与者说话状态变化   | `{ participantId: string; isSpeaking: boolean }` |
| `media:audio:analyserData` | 远端音频分析器数据（~60fps） | `{ frequencyData: Uint8Array; timeDomainData: Uint8Array }` |

**本地麦克风**

| 事件                         | 触发时机     | Payload     |
| ---------------------------- | ------------ | ----------- |
| `media:audio:captureStarted` | 本地输入开始 | `undefined` |
| `media:audio:captureStopped` | 本地输入停止 | `undefined` |
| `media:audio:frameData` | 原始音频帧数据；仅存在真实公开订阅时处理 PCM | `{ data: Float32Array; sampleRate: number }` |

**本地麦克风分析器**

| 事件                                  | 触发时机           | Payload                |
| ------------------------------------- | ------------------ | ---------------------- |
| `media:audio:localAnalyserData` | 本地麦克风分析器数据（~60fps） | `{ frequencyData: Uint8Array; timeDomainData: Uint8Array }` |

**本地摄像头**

| 事件                   | 触发时机       | Payload     |
| ---------------------- | -------------- | ----------- |
| `media:camera:started` | 摄像头采集开始 | `undefined` |
| `media:camera:stopped` | 摄像头采集停止 | `undefined` |

**播放音量**

| 事件                          | 触发时机               | Payload                |
| ----------------------------- | ---------------------- | ---------------------- |
| `media:audio:volumeChanged`   | 音量变化               | `{ volume: number }`   |
| `media:audio:muted`          | 输出静音               | `undefined`            |
| `media:audio:unmuted`        | 解除静音               | `undefined`            |

**会话**

| 事件                            | 触发时机     | Payload                                                                                                          |
| ------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------- |
| `conversation:question:sent`    | 问题已发送   | `{ questionId: string; text: string }`                                                                           |
| `conversation:answer:waiting`   | 等待回答     | `{ questionId: string }`                                                                                         |
| `conversation:server:message`   | 服务端消息   | `{ questionId; rawText; representations; type }`；deprecated `message === rawText` |
| `conversation:asr:received`     | ASR 最终结果 | `{ questionId; rawText; representations }`；deprecated `text === rawText` |
| `conversation:asr:chunk`        | ASR 文本分片 | `{ questionId; rawText; representations; isComplete }`；deprecated `text === rawText` |
| `conversation:answer:chunk`     | 回答文本分片 | `{ questionId; rawDelta; rawText; representations; isComplete }`；deprecated `chunk === rawDelta` |
| `conversation:answer:completed` | 单次回答结束 | `{ questionId; rawText; representations }`；deprecated `fullAnswer === rawText` |

`representations` 是按插件成功安装顺序、setup 注册顺序排列的 JSON-safe 投影；相同 `mediaType` 不会互相覆盖。新代码只使用 `rawDelta` / `rawText`。旧字段在 2.8.0 继续返回且不输出 warning，只能在未来 major 删除。

**会话控制**

| 事件              | 触发时机       | Payload                   |
| ----------------- | -------------- | ------------------------- |
| `session:closing` | 服务端发起关闭 | `Record<string, unknown>` |

---

## 10. 完整使用示例

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

**Direct 模式下更换房间/token**：

```ts
client.updateConnectionConfig({ sfuUrl: 'wss://new-host', userToken: 'new-token' });
await client.reconnect();
```

---

## 11. 视频绿幕参数调试指南

对 plain 或 legacy 流通过初始化配置启用客户端绿幕时，请确保已应用以下设置。SDK 不会用这些设置推断 packed 透明布局。渲染模式可以在创建 SDK 或 Auth 服务端连接配置中确定；连接成功后也可以用 `updateGreenScreenOptions({ enabled })` 在 raw 与 processed 路径间切换。

- `video.renderMode = 'processed'`
- `greenScreen.enabled = true`
- `greenScreen.isBackgroundKeying = true`（可选：至少三条画面边缘为幕布时才执行抠图）

### 11.1 参数调试建议

1. **背景取色（`chromaKey`）**
   - 支持高色度绿色（色相 `80°..160°`）与高色度蓝色（色相 `200°..260°`）；优先使用绿幕，人物服装或道具含大量绿色时可选择蓝幕。
   - RGB 三个分量均须为有限值 `0..255`，并满足饱和度 `≥ 55%`、明度 `≥ 30%`、YCbCr 色度半径 `≥ 0.35`。
   - 红、青、紫、灰白黑、低饱和、过暗或越界的完整元组都会回退默认值 `[0, 255, 0]`，不会继续用于抠图。
   - 建议从真实视频截图中选择幕布具有代表性的亮区颜色。当前版本只支持一个代表性 key，不包含幕布阴影、反光等空间不均匀颜色的自动采样。

2. **相似度（`similarity`）**
   - 默认值：`0.4`
   - 推荐从 `0.3` 开始逐步调整
   - 数值过大可能误抠除人物细节
   - SDK 会与 `smoothness` 一起根据 key 的色度预算收紧有效值，避免中性前景进入透明羽化区。

3. **边缘颜色溢出抑制（`despillStrength`）**
   - 默认值：`1.15`
   - 绿色 key 去除绿色反光；蓝色 key 去除蓝色反光。

4. **边缘平滑（`smoothness`）**
   - 默认值：`0.25`
   - 可改善头发、半透明区域的融合效果

5. **背景门控（`isBackgroundKeying`）**
   - 默认值：`false`。设为 `true` 后，SDK 会将每帧缩小到 `32 × 18` 采样，并使用当前 `chromaKey` 与有效 `similarity` 分别检查顶、右、底、左四条边。
   - 四条边中至少三条的采样像素各有 `75%` 命中 key 色，才执行现有抠图与去色溢；不足三条或浏览器无法读取采样帧时，SDK 原样显示该帧，不会盲目抠图。
   - 它适用于“画面中央有绿色物品、但四周并非绿幕”的场景。若真实绿幕覆盖画面四周，绿色衣物或道具与背景仍无法由纯颜色算法区分；该场景需要人像/前景分割模型。

### 11.2 实时调整

连接成功后可以直接增量调整参数，无需重建 SDK 或处理器；普通调参会复用当前 CPU/WebGL 后端。传入 `enabled: true` 会立即切换至 processed Canvas 路径，传入 `enabled: false` 会立即切换回 raw video 路径；其他绿幕参数也会即时生效。

```ts
client.updateGreenScreenOptions({
  enabled: true,
  similarity: 0.34,
  smoothness: 0.18,
  despillStrength: 1.25,
  isBackgroundKeying: true,
});
```

此方法仅可在 `client.isConnected === true` 时调用，且修改仅作用于当前连接会话；`reconnect()` 后会恢复该次连接从构造参数或 Auth 服务端配置解析出的绿幕配置。

---

## 12. 常见问题

**视频无画面或黑屏**

1. 是否已为 `video.containerElement` 传入有效 DOM 节点。
2. 是否已成功 `connect()` 且 `client.isConnected === true`。
3. 是否收到 `media:video:available` 或至少 `media:video:trackAdded`（业务可据此展示加载态）。

**Auth 模式 preConnect / connect 失败**

1. `http.baseURL` 是否指向正确环境；`setAuthToken` 是否在连接前注入。
2. 打开 `debug: true` 查看控制台与 `sdk:error` 的 `code`。

**Direct 模式 reconnect 仍使用旧 token**

1. 是否在 `preConnect()` / `connect()` 或 `reconnect()` 前调用了 `updateConnectionConfig`；该方法**不会**影响当前连接，仅作用于后续连接路径。

**绿幕卡顿**

1. 适当缩小展示区域分辨率或改用 `renderMode: 'raw'` 对比验证是否为 CPU/GPU 瓶颈。

**无音频输出**

1. 音频播放前用户未进行过主动交互操作

---

## 13. 错误代码

以下为内部 `ErrorCode` 的字符串值，与 `sdk:error` 及抛出 `SDKError` 的 `code` 一致。`ErrorCode` 未从包根入口导出；业务应检查公开错误对象中的字符串 `code`。

### 13.1 HTTP 与控制面

| 代码                                | 说明与处理建议                                                          |
| ----------------------------------- | ----------------------------------------------------------------------- |
| `SDK_AUTH_TOKEN_FAILED`             | 鉴权 HTTP 失败；检查 token 与网关地址。                                 |
| `SDK_GET_LIVEKIT_CONFIG_FAILED`     | 获取房间配置失败；检查后端返回字段是否包含 `livekitUrl` / `roomToken`。 |
| `SDK_SWITCH_VIDEO_FAILED`           | 切换视频相关控制失败；结合日志排查。                                    |
| `SDK_INTERRUPT_CONVERSATION_FAILED` | 打断指令发送失败。                                                      |
| `HTTP_CONTROLLER_NOT_AVAILABLE`     | HTTP 控制器未就绪；避免在错误生命周期调用依赖 HTTP 的操作。             |

### 13.2 SDK 生命周期

| 代码                           | 说明与处理建议                                                        |
| ------------------------------ | --------------------------------------------------------------------- |
| `SDK_PRECONNECT_FAILED`        | 预连接失败；检查网络、Direct 参数是否为空、Auth token。               |
| `SDK_CONNECT_FAILED`           | 连接失败；查看前置错误。                                              |
| `SDK_INITIALIZATION_FAILED`    | 初始化阶段失败；查看控制台堆栈。                                      |
| `SDK_DISCONNECT_FAILED`        | 断开异常；可记录日志后重试新建实例。                                  |
| `SDK_RECONNECT_FAILED`         | 手动重连失败。                                                        |
| `SDK_NOT_CONNECTED`            | 未连接时调用了需要连接态的 API；先 `connect()` 或检查 `isConnected`。 |
| `SDK_INVALID_STATE_TRANSITION` | 非法状态迁移（如 Auth 模式调用 `updateConnectionConfig`）。           |
| `SDK_PLUGIN_CONTEXT_INACTIVE` | 插件卸载或 SDK dispose 后继续使用旧 registrar/CoreService proxy；停止使用该 handle。 |
| `SDK_ERROR`                    | 通用回退错误码。                                                      |

### 13.3 LiveKit / RTC

| 代码                                        | 说明与处理建议                           |
| ------------------------------------------- | ---------------------------------------- |
| `LIVEKIT_CONNECT_FAILED`                    | 房间连接失败；校验 URL/token/TURN 网络。 |
| `LIVEKIT_DISCONNECTED`                      | 已建立的房间因不可恢复的传输、媒体、协议、Agent 或 SIP trunk 运行期故障终止；结合 `sdk:disconnected` 的 reason 排查。 |
| `LIVEKIT_SEND_VIDEO_AVAILABLE_STATE_FAILED` | 上报视频可用状态失败。                   |
| `LIVEKIT_SEND_TEXT_DATA_FAILED`             | 文本数据经 LiveKit 发送失败。            |
| `LIVEKIT_DATA_MESSAGE_PARSE_ERROR`          | 数据消息解析失败；核对协议版本。         |
| `LIVEKIT_UNPUBLISH_MICROPHONE_FAILED`       | 取消麦克风发布失败。                     |

### 13.4 音频

| 代码                                                                          | 说明与处理建议                                                       |
| ----------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `AUDIO_CAPTURE_START_FAILED`                                                  | 麦克风启动失败；用户手势、HTTPS、权限。                              |
| `AUDIO_INPUT_MONITORING_UNAVAILABLE` | 本地采样或音量检测不可用，麦克风推流继续；不要直接断开连接。 |
| `AUDIO_CAPTURE_FAILED`                                                        | 采集中断。                                                           |
| `AUDIO_INVALID_SAMPLE_RATE` / `AUDIO_INVALID_CHANNEL` / `AUDIO_INVALID_CODEC` | 参数与设备/协议不匹配。                                              |
| `AUDIO_CONTROLLER_NOT_AVAILABLE`                                              | 控制器未创建或已释放。                                               |
| `AUDIO_OUTPUT_DISABLED` | 音频输出被禁用时内部输出管线可能报告此错误；`getAudioElement()` 不是 `SDKClient` 公共方法。 |

### 13.5 视频 / 摄像头

| 代码                              | 说明与处理建议       |
| --------------------------------- | -------------------- |
| `CAMERA_CONTROLLER_NOT_AVAILABLE` | 摄像头控制器不可用。 |

### 13.6 会话与状态机

| 代码                                     | 说明与处理建议                         |
| ---------------------------------------- | -------------------------------------- |
| `CONVERSATION_CONTROLLER_NOT_AVAILABLE`  | 会话控制器不可用。                     |
| `STATE_MACHINE_INVALID_STATE_TRANSITION` | 内部状态机收到非法迁移；收集日志反馈。 |

### 13.7 工具与其它

| 代码                     | 说明与处理建议                        |
| ------------------------ | ------------------------------------- |
| `OBJECT_DISPOSED`        | 实例已 `dispose()`。                  |
| `NO_AVATARID`            | Auth 配置缺少 `avatarId`。            |
| `NO_AUTH_TOKEN`          | 缺少鉴权 token（依运行时校验）。      |
| `INVALID_CONNECT_CONFIG` | `createClient` 时 Direct 配置不合法。 |

---

## 14. 性能监控与排查

### 14.1 内置指标

默认开启（`performanceMonitor.enabled !== false`）。指标名 `PerformanceMetricName`：

| `metric`                                | 含义                           |
| --------------------------------------- | ------------------------------ |
| `connect_to_first_frame_ms`             | `connect()` 至 SDK 内部 video 的 `loadeddata` 耗时 |
| `text_send_to_text_response_ms`         | 文本发送到首段文本响应         |
| `text_send_to_audio_response_ms`        | 文本发送到首段音频响应         |
| `no_speech_report_to_audio_response_ms` | 无人声上报到音频响应           |

记录结构：`PerformanceMetricRecord`（`metric`、`durationMs`、`startedAt`、`endedAt`、可选 `questionId`；首帧相关记录另有可选 `attemptId`）。已有监控会话时同时提供 `clientInstanceId` 和 `monitoringSessionId`，迟到结果沿用测量开始时的快照。

连接阶段、连接失败/首帧超时、文本响应和文本到音频响应的 `durationMs` 使用 `performance.now()` 的单调差值。`startedAt` 是测量开始时读取的 Unix 毫秒时间戳，`endedAt = startedAt + durationMs`；系统时间前跳或回拨不会改变这些测量的耗时，也不会使同条记录的时间顺序倒置。保留小数毫秒，耗时以 `durationMs` 为准。

这些时间戳表示以起点为锚的测量时间线。系统校时后，`endedAt` 可能与完成时的 `Date.now()` 不同；不要将其当作完成时独立采样的墙钟事实。不同测量各自读取新起点，跨测量不保证全局时间顺序。成功首帧终态批次复用首帧指标的结束时间；运行日志的发生时间仍遵循原有墙钟语义。

`no_speech_report_to_audio_response_ms` 保留事件携带的墙钟起点及墙钟差值，仍可能受系统校时影响。当前未发现生产发送方或单调起点，本轮修复不覆盖该指标，也不保证其在正常会话中产生。

### 14.1a 分段首帧时间线

每次有效的 connect 都有递增的 attemptId。只有在重复连接防护之后才会创建 attempt；所有阶段均相对该次 connect 起点累计。已有的 connect_to_first_frame_ms 名称保持兼容，其终点精确定义为 SDK 内部持久 video 元素触发 loadeddata，而不是业务 DOM 已显示像素、playing 或 requestVideoFrameCallback。

| 指标 | 首次事实 |
| --- | --- |
| connect_to_config_ready_ms | 连接配置应用完成 |
| connect_to_rtc_connected_ms | 内部 RTC 已连接 |
| connect_to_video_track_added_ms | 远端视频轨道已加入 |
| connect_to_video_metadata_ms | 内部 video 触发 loadedmetadata |
| connect_to_first_frame_ms | 内部 video 触发 loadeddata |
| connect_to_first_frame_timeout_ms | RTC 连接后 10 秒仍没有 loadeddata |
| connect_to_first_frame_failed_ms | 任一 connect 失败路径在首帧前结束 |

阶段只记录一次；没有发生的阶段不会被写成 0，也不会出现在该 attempt 的终态报告。首帧相关 PerformanceMetricRecord 会带可选 attemptId，其余问答指标继续使用可选 questionId。

RTC 连接后，SDK 启动 10 秒首帧等待计时器。loadeddata、connect 失败、显式 disconnect 和 dispose 都会清理计时器。显式 disconnect、dispose 或被下一次连接替换的 attempt 仅取消，不产生失败或超时指标。被动 RTC 断开若不走 connect 失败路径，仍由该 10 秒计时器收口。

### 14.1b 终态批次与插件输入

每个成功、connect 失败或首帧超时的 attempt 会生成一次只读 SDKPerformanceMetricsReport：schema 为 sdk.performance-metrics，schemaVersion 为 1，包含 attemptId、可选 clientInstanceId/monitoringSessionId、outcome、startedAt、endedAt 与只包含已观察指标的 metrics。SDK 根入口同时导出 detached Observer token SDK_PERFORMANCE_METRICS_V1。Observer 获得深拷贝且冻结的快照；SDK 不等待 handler，也不因插件 handler 的异常、拒绝或超时改变连接或媒体流程。

SDK 还会发送一条脱敏 Operational Log：category 为 media，event 为 performance.first-frame，顶层带监控会话关联，correlation 只有 attemptId；attributes 只有 outcome、renderMode、timeout 场景下的 timeoutAfterRtcMs 和稀疏数值 nodes。该日志不包含 URL、token、连接配置、房间/轨道/参与者标识、文本、DataChannel 原文、媒体帧、DOM/LiveKit 对象、Error、message 或 stack。当前 SDK 不自动上传；插件需注册 observer 或匹配该日志，自行负责 setPerformance、reportFrameErrors、鉴权、重试和投递；SkyWalking 1.1.0 仍使用旧契约。

### 14.2 结构化测量结果与插件接入

SDK 提供性能指标，插件负责接收并上传。`setPerformanceMetricReporter()`、`PerformanceMonitorOptions.reporter` 和 `PerformanceMetricReporter` 仅保留对外签名，现已无作用，将于 **3.0.0 移除**。回调不会执行，也不再输出默认 `[performance-metric]` Console 日志。这是旧 reporter 行为的明确变更。`performanceMonitor.enabled` 继续控制性能测量，默认开启。

插件通过 `SDK_PERFORMANCE_METRIC_V1`（identity：`performance.metric`，major 1）消费逐条 `PerformanceMetricRecord`，包括阶段耗时和源事件发生时的会话响应耗时，无额外包装。当前实现校验指标名称、有限非负耗时、时间顺序及可选 `questionId` / `attemptId`；同一次派发的消费者共享一个深拷贝冻结快照。detached 分发使用微任务，不执行配置中的队列上限和溢出策略；handler 超时 30 秒并隔离故障贡献。此前文档承诺每消费者独立快照及上限 100 的丢弃队列，与现有实现不一致，依赖这些保证的插件存在兼容风险；本次仅记录差异，未更改插件契约或 SDK 实现。插件负责上传、重试和存储。

`SDK_PERFORMANCE_METRICS_V1` 仍提供首帧终态批次，协议不变；同一首帧指标可能出现在两个出口，按用途选择以避免重复统计。现有输出卡顿检测仍由 `getPlaybackStatus()` 返回，不改变阈值，也不并入性能指标流。断开、销毁或新连接尝试清理待完成的会话测量，防止跨连接关联。

音频响应耗时不代表声音实际播放的精确时刻，也不保证逐问题的音频归属。`no_speech_report_to_audio_response_ms` 不保证在正常会话中产生；缺失时不要填充为 `0`。

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

### 14.3 排查建议

- **首帧慢**：检查地域、TURN、是否重复失败重试 `connect()`。
- **文本响应慢**：结合后端 LLM/业务耗时与主线程阻塞。
- **音频明显慢于文本**：检查 TTS 链路及远端音频轨道订阅状态。

---

_文档版本与包版本一致：2.8.0。_
