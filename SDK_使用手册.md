# Facemarket 实时数字人 SDK 使用手册（v2.3.0）

本手册对应 npm 包 `@sanseng/liveavatar-js-sdk` **2.3.0** 版本。SDK 基于 **LiveKit Client**，封装数字人音视频下行、麦克风/摄像头上行、会话文本 与 HTTP 控制面（鉴权模式下获取连接配置）。

---

## 1. 概述

SDK 对外提供统一入口 **`createClient` → `SDKClient`**，负责：

- 音视频轨道的订阅与路由、本地采集与发布
- 会话侧文本与协议消息的编解码与分发
- 连接生命周期与快照（便于重连与排障）
- 可选绿幕与性能指标上报

使用方通过构造参数、连接 API、事件订阅即可完成集成，无需依赖 SDK 内部模块（如 `transport` 实现细节）。

---

## 2. 从 v1 升级到 v2

从 **v1.3.2 或 v1.3.3** 升级至当前 **v2.3.0** 时，请参阅独立的 [SDK v1 → v2 升级指南](./SDK_v1到v2升级指南.md)。该指南列出后端 DataChannel 兼容验证、对话事件规范字段、断连错误处理、麦克风 RMS 阈值，以及可选 A/V Sync 和插件能力的迁移工作。

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

- `preConnect()` / `connect()`：从当前 Direct 配置（或 `updateConnectionConfig` / `reconnect` 阶段暂存的替换配置）解析出 `livekitUrl` 与 `token`，写入上下文后建立 LiveKit 连接。
- `reconnect()`：在断开当前连接后调用 `refreshConfig()`；Direct 模式下会再次读取 ConfigManager 中的 Direct 配置（含已通过 `replaceDirectConfig` 应用的 `updateConnectionConfig` 结果），不会调用鉴权 HTTP 拉取房间。
- 不会自动「刷新」服务端签发的 token；令牌过期须由业务侧获取新令牌并调用 `updateConnectionConfig` 后再 `preConnect()` / `connect()` 或 `reconnect()`。

**使用限制**

- 不支持在 Auth 模式下调用 `updateConnectionConfig`（将抛出 `SDK_INVALID_STATE_TRANSITION`）。
- 不支持在 Direct 模式下依赖 HTTP 接口返回的 `videoOptions` 覆盖策略；绿幕等视频参数以构造时的 `video` 与运行时 `setRenderFitMode` 等为准（除非业务自行在服务端约定后写入本地配置）。

**适用场景**
已有稳定签发 LiveKit token 的后端、私有化部署、或调试阶段快速接入。

---

### 2.2 Auth Mode（鉴权模式）

**描述**
调用方提供 `avatarId`（及可选的 `authToken`、`avatarVoice`）。SDK 通过 `HttpController` 拉取鉴权与房间配置（`ConnectionConfig`），并可将服务端返回的绿幕等 `videoOptions` 合并进上下文。

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

- `preConnect()` / `connect()`：通过 HTTP 获取 token 与连接负载，解析为 `livekitUrl`、`token`、`roomId` 及可选 `videoOptions`，再建立 LiveKit。
- `reconnect()`：断开后调用 `refreshConfig()`，**重新走 HTTP** 拉取最新 `ConnectionConfig`（缓存会被清空后重新拉取）。
- `setAuthToken`：更新上下文中的令牌；若已存在 `HttpController`，会同步刷新其鉴权读取逻辑。

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
| `reconnect()` 配置刷新   | 使用 `refreshConfig()` 读取 Direct 路径配置（含已 `replaceDirectConfig` 的更新）          | `refreshConfig()` 重新 HTTP 拉取                       |

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

**约定**：`questionId` 为较短字符串（实现中常见为 **4 字符**），仅在当前会话内保证唯一；**不适合**作为长期持久化主键。

**强烈建议**：在收到 `conversation:answer:completed` 后，清理该 `questionId` 对应的本地缓冲（文本拼接、UI 状态等），避免内存增长与 ID 复用带来的展示错误。

### 5.2 控制面与媒体面

- **媒体面**：LiveKit 房间连接、音视频轨道的订阅与发布、会话文本与协议消息的 **Data Channel** 传输，均由 `LiveKitService` 统一承载。
- **控制面（Auth）**：`HttpController` + `ConfigManager` 负责在鉴权模式下获取 `livekitUrl`、`token`、`roomId` 及服务端可选的 `videoOptions`（如绿幕开关）。**不存在**与 LiveKit 并行的独立「业务 WebSocket」通道对外 API。

### 5.3 视频渲染模式

- `video.renderMode === 'raw'`：直接在 `<video>` 上渲染远端轨道（默认类行为以选项为准）。
- `video.renderMode === 'processed'`：经 Canvas 处理管线（如绿幕）。
- **Auth 模式**下，服务端返回的绿幕配置会映射为 `renderMode` / `greenScreen`（见 `ConfigManager._fromAuthPayload`）。若本地与远端均未显式配置，由 SDK 内部策略判定。

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
| `plugins`            | `readonly SDKPlugin[]`      | 否   | 构造期安装的插件列表。传入已从插件包导入的 `SDKPlugin`；单个安装失败不影响客户端构造。详见[插件接入文档](./SDK_插件接入文档.md)。 |

本章仅列出构造参数。Debug 日志、A/V 诊断、自动视频停滞告警和实验性 frame metadata worker 的完整配置及行为说明见第 7 章。

### 6.2 `connectConfig`

**Direct**

| 字段               | 类型       | 必填 |
| ------------------ | ---------- | ---- |
| `type`             | `'direct'` | 是   |
| `config.sfuUrl`    | `string`   | 是   |
| `config.userToken` | `string`   | 是   |

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

**注意**：`channelCount` 为可选，默认为 `1`。`noiseSuppression`、`echoCancellation`、`autoGainControl` 未设置时默认值为 `true`。

### 6.5 `performanceMonitor`

| 字段       | 说明                                                   |
| ---------- | ------------------------------------------------------ |
| `enabled`  | 默认 `true`；`false` 关闭内置性能采集                  |
| `reporter` | `(metric: PerformanceMetricRecord) => void` 自定义上报 |

### 6.6 `MicrophoneStats`

| 字段            | 说明             |
| --------------- | ---------------- |
| `bytesSent`     | 已发送字节数     |
| `packetsSent`   | 已发送数据包数   |
| `packetsLost`   | 丢失的数据包数   |
| `roundTripTime` | 往返时间（毫秒） |

---

## 7. Debug 与 A/V 诊断

### 7.1 基础 Debug 与 LiveKit verbose

`debug: true`、`debug: {}` 仅开启 SDK Debug 日志；只有 `debug: { verbose: true }` 才会开启 LiveKit JS SDK 的内部 Debug 日志。SDK Logger 与 LiveKit Logger 都是同一页面的 runtime 全局设置：每次创建客户端都会应用当前配置；`debug` 为 `false` 或省略时 SDK Logger 恢复为 `ERROR`，非 `verbose: true` 时 LiveKit Logger 恢复为 `info`，客户端 `dispose()` 也会恢复这两个默认级别。因此最后创建或释放的 SDK 客户端会影响同页其他 SDK 或 LiveKit 客户端的日志级别。

### 7.2 周期 A/V 诊断（`debug.avSync`）

通过以下配置在 LiveKit Room 连接成功后按周期采样远端音频/视频接收端统计；默认不轮询：

```ts
debug: {
  avSync: { enabled: true, intervalMs: 2000, historyLimit: 60 },
}
```

后台采样最多等待 10 秒以接收通过 renderer 参与者过滤的首个远端音频或视频轨道；超时后停止当前连接会话的后台采样，并在上传日志中记录 `remote-track-wait-timeout`。该超时不影响手动 `captureRemoteAvSyncDiagnostics()`。

离线证据经 `getAvSyncRawTelemetry(query?)` 按页返回，不再输出 `[RTC remote-media]` Console 行；无参只返回最近 5 分钟、最多 200 条，使用 `range`、`pageSize` 和 `page.nextCursor` 读取更多记录。日志查询和事实摘要在 Dedicated Worker 执行，主动 capture 仍在浏览器主线程读取当前 WebRTC stats。`avSyncLogRetention` 返回保留量、存储错误和丢弃量；Room 断开会冻结最后会话供分页查询，重连替换它，`dispose()` 和连接建立失败会清除它。

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

所有告警阶段发送给稳定公开 `sdk:avSyncAlert`，并可由已安装插件接收；插件接入方式见[插件接入文档](./SDK_插件接入文档.md)。只有首次 `raised` 会写入留存的 `video-pipeline-stalled` 事实证据。SDK 不会自动上传告警或日志。只监听公开事件、不轮询 `getAvSyncAnalysis()`，即可覆盖上述自动周期观测 transition；但 `getAvSyncAnalysis()` 仍负责轨道绑定、传输退化、音频 concealment、播放和相对漂移等完整保留窗口的手动分析。此类 finding 只有业务实际调用分析 API 后才可能变为告警。

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

插件开发、扩展点和运行语义见[插件接入文档](./SDK_插件接入文档.md)。使用插件时，从其自身包导入 `SDKPlugin` 对象；将下面占位符替换为“当前已支持插件包”目录中实际插件提供的包名和导出名。

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

#### 当前已支持插件包

| 插件包 | 用途 | 状态 |
| --- | --- | --- |
| 暂无 | v2.3.0 当前没有已发布或已登记的独立插件包。SDK 扩展点不是可直接安装的插件包。 | 暂无可用包 |

有实际插件包发布后，本表会补充包名、安装命令和链接。

---

## 8. 核心 API 方法

以下方法均定义于 `SDKClient`（`src/client/SDKClient.ts`）。除构造与 `setAuthToken` / `updateConnectionConfig` 外，多数媒体与会话 API 要求 **已成功 `connect()`**（内部通过 `sessionState.isConnected` 校验）。

### 8.1 连接与生命周期

| 方法                                       | 说明                                                                                                      |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| `preConnect(): Promise<boolean>`           | 预拉取并缓存连接配置（带 TTL，默认 60s 量级，见 `PRECONNECT_CACHE_DURATION_MS`）。失败抛出错误。          |
| `connect(): Promise<void>`                 | 建立协调器、各域控制器与 LiveKit；内部可在无有效缓存时等价调用预连接路径。                                |
| `disconnect(): Promise<void>`              | 停止采集、断开 LiveKit、结束 HTTP 会话相关清理；实例可再次 `connect()` / `reconnect()`。                  |
| `reconnect(): Promise<ConnectionSnapshot>` | 手动重连：先 `disconnect()`，再按模式刷新配置后 `connect()`。若当前状态不允许重连，返回当前快照并打日志。 |
| `installPlugin(plugin): SDKPluginInstallResult` | 原子安装 Plugin API v1 插件；拒绝以结果数据返回，不影响 SDK 生命周期。 |
| `uninstallPlugin(pluginId: string): boolean` | 原子卸载全部 contribution、失效旧 handle 并调用插件 dispose。 |
| `dispose(): void`                          | 释放全部资源；之后不得再使用该实例。                                                                      |

**约束**：浏览器策略下，**带声音的播放**与 **麦克风采集** 建议在用户手势（点击等）回调内触发 `connect()` / `startAudioCapture()`。

### 8.2 鉴权与连接配置

| 方法                                                           | 说明                                                                                                                                                                          |
| -------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `setAuthToken(token: string): void`                            | 写入鉴权令牌；**Auth 模式**下供 HTTP 与 Config 拉取使用。                                                                                                                     |
| `updateConnectionConfig(config: DirectConnectionConfig): void` | **仅 Direct**。校验 `sfuUrl` / `userToken` 非空后暂存，**不影响当前已连接会话**；在下次 `preConnect()` / `connect()` 或 `reconnect()` 流程中通过 `replaceDirectConfig` 生效。 |

### 8.3 媒体

| 方法                                                     | 说明                               |
| -------------------------------------------------------- | ---------------------------------- |
| `setRenderFitMode(mode: RenderFitMode): void`            | 设置画面适配模式。                 |
| `updateGreenScreenOptions(options: Partial<GreenScreenOptions>): void` | 仅在已连接会话中实时更新绿幕参数。`enabled: true` / `false` 会立即在 processed Canvas 与 raw video 路径间切换；其他参数也即时生效。运行期改动仅作用于当前连接，`reconnect()` 后恢复构造参数或本次 Auth 配置。 |
| `startAudioCapture(): Promise<void>`                     | 打开麦克风并通过 LiveKit 发布。    |
| `stopAudioCapture(): Promise<void>`                      | 停止麦克风发布。                   |
| `setVolume(volume: number): void`                        | 播放音量 `0..1`。                  |
| `getVolume(): number`                                    | 读取播放音量。                     |
| `mute()` / `unmute()`                                    | 播放静音控制。                     |
| `get isMuted`                                            | 是否静音。                         |
| `get isAudioCapturing`                                   | 麦克风是否处于活跃发布状态。       |
| `getMicrophoneAudioLevel(): number \| null`              | 获取最近一个 4096-sample 本地 PCM 帧的线性 RMS (0.0-1.0)；首帧前为 `0`，无活动麦克风时为 `null`。 |
| `getMicrophoneStats(): Promise<MicrophoneStats \| null>` | 获取麦克风发送统计信息。           |
| `isMicrophoneSilent(): Promise<boolean \| null>`         | 检测麦克风是否在发送静音。         |
| `startCamera(options?: { publishToLiveKit?: boolean }): Promise<void>` | 开启摄像头；可选择发布到 LiveKit 房间（默认仅本地预览）。 |
| `stopCamera(): void`                                     | 停止摄像头。                       |
| `getCameraStream(): MediaStream \| null`                 | 本地预览用流。                     |
| `getCameraTrack(): MediaStreamTrack \| null`             | 本地轨道。                         |
| `attachCameraTo(videoElement: HTMLVideoElement): void`   | 将本地摄像头画面绑定到 `<video>`。 |
| `detachCameraFrom(videoElement: HTMLVideoElement): void`  | 将摄像头轨道从视频元素解绑。       |
| `startAnalyserEmission(): void`                         | 开始发送远端音频分析器数据（~60fps）。若远端音频轨道尚未到达，将自动延迟等待轨道订阅成功后开始发送。 |
| `stopAnalyserEmission(): void`                          | 停止远端音频分析器并释放 AudioContext。 |
| `restartAnalyserEmission(): void`                        | 重启远端音频分析器链路。           |
| `startLocalAnalyserEmission(): void`                     | 开始发送本地麦克风分析器数据（~60fps）。若麦克风尚未启动，将自动延迟等待 `startAudioCapture()` 成功后开始发送（轮询间隔100ms，最多等待10s）。 |
| `stopLocalAnalyserEmission(): void`                     | 停止本地麦克风分析器。             |
| `restartLocalAnalyserEmission(): void`                  | 重启本地麦克风分析器链路。         |

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
| `get sessionId(): string \| undefined`                                     | 服务器分配的会话 ID。**仅 Auth 模式**；Direct 模式下返回 `undefined` 并记录调试日志。   |
| `setPerformanceMetricReporter(reporter?: PerformanceMetricReporter): void` | 设置或更新性能指标上报回调。                                                            |
| `captureRemoteAvSyncDiagnostics(): Promise<AvSyncDiagnosticsSnapshot>`      | 主动采集远端音视频接收统计和本次连接的轨道生命周期节点；未配置 `debug` 也可调用。        |
| `getAvSyncRawTelemetry(query?): Promise<RemoteAvSyncRawTelemetry>`           | 读取当前有界上下文和一页 `avSyncLogs`。默认最近 5 分钟、200 条；通过 `range`、`pageSize` 和 `page.nextCursor` 分页。窗口事实摘要默认启用，可用 `includeSummary: false` 关闭；日志查询和摘要在 Dedicated Worker 执行。 |
| `getAvSyncAnalysis(query?): Promise<AvSyncAnalysisReport>` | 在 telemetry Worker 中异步分析完整保留窗口；调用方自行决定轮询周期。返回 `status`、`finding`、`features`、`layers` 和有界限制信息，不会因证据不足抛出异常；自动视频停滞通知不依赖调用此方法。 |
| `get events(): PublicEmitterAPI`                                           | 仅 `on` / `off` / `once`。                                                              |

`getAvSyncAnalysis()` 的计数均是**请求分析窗口内的增量**，不会把浏览器 WebRTC 的累计 receiver counter 当作当前异常量。`features` 中的关键字段如下：

- `audioObservationDurationMs`：有效远端音频 delta interval 的总观测时长。
- `audioConcealedSamplesDelta` / `audioConcealmentEventsDelta`：该窗口内的音频 concealment 增量。
- `audioConcealedSamplesPerSecond` / `audioConcealmentEventsPerSecond`：以上增量除以观测时长；没有有效音频 delta interval 时为 `null`。
- `videoPacketsLostDelta`、`videoFramesReceivedDelta`、`videoFramesDroppedDelta`、`videoFramesDecodedDelta`：该窗口内的远端视频增量；其中 received/decoded 用于区分接收组帧与解码推进情况。

多远端参与者时，全局音频速率的分母是所有有效远端 audio channel 的观测时长之和。`participants` 及告警 participant IDs 只包含本窗口内具有远端媒体证据的身份；本地或仅连接质量相关的参与者不会出现在分析结果中。

### 8.6 版本信息

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
- **说明**：可与 `connectionSnapshot` 交叉校验。判定"已断开 → 可重连"应使用 `payload.allConnected === false`。

### `sdk:error`

- **触发时机**：内部错误经映射为对外安全负载时。
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

- **触发时机**：远端参与者离开 LiveKit 房间时。
- **Payload**：`{ participantId: string }`。

**视频**

| 事件                       | 触发时机                                | Payload                            |
| -------------------------- | --------------------------------------- | ---------------------------------- |
| `media:video:available`    | 远端视频可用（首帧等语义由 RTC 层驱动） | `undefined`                        |
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
| `media:audio:frameData` | 原始音频帧数据 | `{ data: Float32Array; sampleRate: number }` |

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

`representations` 是按插件成功安装顺序、setup 注册顺序排列的 JSON-safe 投影；相同 `mediaType` 不会互相覆盖。新代码只使用 `rawDelta` / `rawText`。旧字段在 2.3.0 继续返回且不输出 warning，只能在未来 major 删除。

**会话控制**

| 事件              | 触发时机       | Payload                   |
| ----------------- | -------------- | ------------------------- |
| `session:closing` | 服务端发起关闭 | `Record<string, unknown>` |

---

## 10. 完整使用示例

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

**Direct 模式下更换房间/token**：

```ts
client.updateConnectionConfig({ sfuUrl: 'wss://new-host', userToken: 'new-token' });
await client.reconnect();
```

---

## 11. 视频绿幕参数调试指南

通过初始化配置启用绿幕时，请确保已应用以下设置，或不做配置（由 SDK 内部自动判断）。渲染模式可以在创建 SDK 或 Auth 服务端连接配置中确定；连接成功后也可以用 `updateGreenScreenOptions({ enabled })` 在 raw 与 processed 路径间切换。

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

以下为 `ErrorCode` 枚举（`src/errors/ErrorCodes.ts`）的字符串值，与 `sdk:error` 及抛出 `SDKError` 的 `code` 一致。

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
| `AUDIO_CAPTURE_FAILED`                                                        | 采集中断。                                                           |
| `AUDIO_INVALID_SAMPLE_RATE` / `AUDIO_INVALID_CHANNEL` / `AUDIO_INVALID_CODEC` | 参数与设备/协议不匹配。                                              |
| `AUDIO_CONTROLLER_NOT_AVAILABLE`                                              | 控制器未创建或已释放。                                               |
| `AUDIO_OUTPUT_DISABLED`                                                       | `output.enabled === false` 时访问 `getAudioElement()` 会抛出此错误。 |

### 13.5 摄像头

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
| `connect_to_first_frame_ms`             | `connect()` 至首帧渲染完成耗时 |
| `text_send_to_text_response_ms`         | 文本发送到首段文本响应         |
| `text_send_to_audio_response_ms`        | 文本发送到首段音频响应         |
| `no_speech_report_to_audio_response_ms` | 无人声上报到音频响应           |

记录结构：`PerformanceMetricRecord`（`metric`、`durationMs`、`startedAt`、`endedAt`、可选 `questionId`）。

### 14.2 自定义上报

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

或在运行时：

```ts
client.setPerformanceMetricReporter((metric) => {
  /* 上报 OTLP / 自建 */
});
```

### 14.3 排查建议

- **首帧慢**：检查地域、TURN、是否重复失败重试 `connect()`。
- **文本响应慢**：结合后端 LLM/业务耗时与主线程阻塞。
- **音频明显慢于文本**：检查 TTS 链路及远端音频轨道订阅状态。

---

_文档版本与包版本一致：2.3.0。_
