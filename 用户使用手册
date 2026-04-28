
# 实时音视频交互 SDK 使用手册（v1.0.0）

本手册对应 npm 包 `@sanseng/livekit-ws-sdk` **1.0.0** 版本。SDK 基于 **LiveKit Client**，封装数字人音视频下行、麦克风/摄像头上行、会话文本 与 HTTP 控制面（鉴权模式下获取连接配置）。

---

## 1. 概述

SDK 对外提供统一入口 **`createClient` → `SDKClient`**，负责：

- 音视频轨道的订阅与路由、本地采集与发布
- 会话侧文本与协议消息的编解码与分发
- 连接生命周期与快照（便于重连与排障）
- 可选绿幕与性能指标上报

使用方通过构造参数、连接 API、事件订阅即可完成集成，无需依赖 SDK 内部模块（如 `transport` 实现细节）。

---

## 2. 初始化说明

SDK 通过 `ClientOptions.connectConfig` 区分两种互斥模式：**Direct（直连）** 与 **Auth（鉴权）**。二者在配置来源、`setAuthToken` / `updateConnectionConfig` 的可用性及 `reconnect` 刷新行为上不同。

### 2.1 Direct Mode（直连模式）

**描述**  
由调用方在构造参数中直接提供 LiveKit 所需的 `sfuUrl` 与 `clientToken`。SDK 不通过 HTTP 拉取房间配置。

**前置条件**

- 必须提供非空的 `sfuUrl`（LiveKit WebSocket URL）与 `clientToken`（房间访问令牌）。
- 若业务在运行中更换房间或令牌，必须在下次重连前通过 `updateConnectionConfig` 注入新配置（见行为说明）。

**初始化方式**

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

**行为说明**

- `preConnect()` / `connect()`：从当前 Direct 配置（或 `updateConnectionConfig` / `reconnect` 阶段暂存的替换配置）解析出 `livekitUrl` 与 `token`，写入上下文后建立 LiveKit 连接。
- `reconnect()`：在断开当前连接后调用 `refreshConfig()`；Direct 模式下会再次读取 ConfigManager 中的 Direct 配置（含已通过 `replaceDirectConfig` 应用的 `updateConnectionConfig` 结果），不会调用鉴权 HTTP 拉取房间。
- 不会自动「刷新」服务端签发的 token；令牌过期须由业务侧获取新令牌并调用 `updateConnectionConfig` 后再 `reconnect()`。

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
import { createClient } from '@sanseng/livekit-ws-sdk';

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

| 项目                     | Direct Mode                                                                      | Auth Mode                                              |
| ------------------------ | -------------------------------------------------------------------------------- | ------------------------------------------------------ |
| 配置来源                 | 构造参数中的 `sfuUrl`、`clientToken`                                             | HTTP 接口返回的 `ConnectionConfig`                     |
| 是否需要业务 HTTP        | 否（仅当同时使用其他 HTTP 能力时可选配 `http`）                                  | 是（获取鉴权与房间配置）                               |
| 必填字段                 | `sfuUrl` + `clientToken`                                                         | `avatarId` + 有效 `authToken`（构造或 `setAuthToken`） |
| `setAuthToken`           | 不用于解析 LiveKit 连接配置                                                      | 必须或建议在连接前注入                                 |
| `updateConnectionConfig` | 可用；作用于**下一次** `reconnect()` 所使用的 Direct 配置                        | 不可用（抛出错误）                                     |
| `reconnect()` 配置刷新   | 使用 `refreshConfig()` 读取 Direct 路径配置（含已 `replaceDirectConfig` 的更新） | `refreshConfig()` 重新 HTTP 拉取                       |

---

## 3. 快速开始

### 3.1 安装

```bash
npm install @sanseng/livekit-ws-sdk
```

### 3.2 最小流程（Auth 示例）

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

// 必须由业务后端签发后注入，此处省略具体请求实现
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

// 用户点击后再开麦（满足浏览器自动播放 / 采集策略）
document.getElementById('mic')?.addEventListener('click', async () => {
  await client.startAudioCapture();
});
```

### 3.3 最小流程（Direct 示例）

```ts
const client = createClient({
  connectConfig: {
    type: 'direct',
    config: { sfuUrl: 'wss://...', clientToken: '...' },
  },
  video: { containerElement: document.getElementById('avatar')! },
});

await client.connect();
```

---

## 4. 核心概念

### 4.1 questionId（问答关联标识）

`questionId` 是**一次完整问答对**（用户提问 → 服务端处理 → 流式/完整回答）的关联键，用于串联：

- `conversation:question:sent`
- `conversation:answer:waiting`
- `conversation:server:message`
- `conversation:asr:received` / `conversation:asr:streaming`
- `conversation:answer:chunk`
- `conversation:answer:completed`

**约定**：`questionId` 为较短字符串（实现中常见为 **4 字符**），仅在当前会话内保证唯一；**不适合**作为长期持久化主键。

**强烈建议**：在收到 `conversation:answer:completed` 后，清理该 `questionId` 对应的本地缓冲（文本拼接、UI 状态等），避免内存增长与 ID 复用带来的展示错误。

### 4.2 控制面与媒体面

- **媒体面**：LiveKit 房间连接、音视频轨道的订阅与发布、会话文本与协议消息的 **Data Channel** 传输，均由 `LiveKitService` 统一承载。
- **控制面（Auth）**：`HttpController` + `ConfigManager` 负责在鉴权模式下获取 `livekitUrl`、`token`、`roomId` 及服务端可选的 `videoOptions`（如绿幕开关）。**不存在**与 LiveKit 并行的独立「业务 WebSocket」通道对外 API。

### 4.3 视频渲染模式

- `video.renderMode === 'raw'`：直接在 `<video>` 上渲染远端轨道（默认类行为以选项为准）。
- `video.renderMode === 'processed'`：经 Canvas 处理管线（如绿幕）。
- **Auth 模式**下，服务端返回的绿幕配置会映射为 `renderMode` / `greenScreen`（见 `ConfigManager._fromAuthPayload`）。若本地与远端均未显式配置，由 SDK 内部策略判定。

---

## 5. 配置说明（`createClient` 构造参数）

`createClient(options: ClientOptions)`，`ClientOptions` 定义见 `src/types/ClientOptions.d.ts`。

### 5.1 顶层字段

| 字段                 | 类型                        | 必填 | 说明                                                                                    |
| -------------------- | --------------------------- | ---- | --------------------------------------------------------------------------------------- |
| `connectConfig`      | `direct \| auth` 联合       | 是   | 连接模式与模式专属配置                                                                  |
| `audio`              | `AudioOptions`              | 否   | 输入/输出音频选项                                                                       |
| `video`              | `VideoOptions`              | 否   | 渲染容器、绿幕、`renderMode`、`fitMode` 等                                              |
| `reconnect`          | `{ maxAttempts?, delay? }`  | 否   | LiveKit 重连：`maxAttempts` 默认 `3`；`delay` 默认 `10`（**秒**），内部会换算为毫秒上限 |
| `http`               | `{ baseURL?, headers? }`    | 否   | Auth 与 HTTP 控制接口根路径及默认头                                                     |
| `performanceMonitor` | `PerformanceMonitorOptions` | 否   | 默认开启；`enabled: false` 可关闭                                                       |
| `sandbox`            | `boolean`                   | 否   | 沙箱开关（依业务后端约定）                                                              |
| `debug`              | `boolean`                   | 否   | 调试日志                                                                                |

### 5.2 `connectConfig`

**Direct**

| 字段                 | 类型       | 必填 |
| -------------------- | ---------- | ---- |
| `type`               | `'direct'` | 是   |
| `config.sfuUrl`      | `string`   | 是   |
| `config.clientToken` | `string`   | 是   |

**Auth**

| 字段                 | 类型     | 必填                                   |
| -------------------- | -------- | -------------------------------------- |
| `type`               | `'auth'` | 是                                     |
| `config.avatarId`    | `string` | 是                                     |
| `config.authToken`   | `string` | 否（可与 `setAuthToken` 二选一或组合） |
| `config.avatarVoice` | `string` | 否                                     |

### 5.3 `video`（`VideoOptions`）

| 字段               | 说明                                                                  |
| ------------------ | --------------------------------------------------------------------- |
| `containerElement` | 视频渲染挂载的容器（**业务侧应保证存在且已插入 DOM**）                |
| `renderMode`       | `'raw'` \| `'processed'`                                              |
| `greenScreen`      | `{ enabled, chromaKey?, similarity?, smoothness?, despillStrength? }` |
| `fitMode`          | `'contain' \| 'cover' \| 'fill' \| 'none'`                            |
| `debug`            | 继承 `BaseOptions`                                                    |

### 5.4 `audio`（`AudioOptions`）

| 字段     | 说明                                                                                           |
| -------- | ---------------------------------------------------------------------------------------------- |
| `input`  | `{ deviceId?: string, sampleRate?: number; noiseSuppression?: boolean;}`（设备约束、采样率等） |
| `output` | `{ enabled?: boolean, volume?: number, muted?: boolean }` 播放侧默认值                         |

### 5.5 `performanceMonitor`

| 字段       | 说明                                                   |
| ---------- | ------------------------------------------------------ |
| `enabled`  | 默认 `true`；`false` 关闭内置性能采集                  |
| `reporter` | `(metric: PerformanceMetricRecord) => void` 自定义上报 |

---

## 6. 核心 API 方法

以下方法均定义于 `SDKClient`（`src/client/SDKClient.ts`）。除构造与 `setAuthToken` / `updateConnectionConfig` 外，多数媒体与会话 API 要求 **已成功 `connect()`**（内部通过 `sessionState.isConnected` 校验）。

### 6.1 连接与生命周期

| 方法                                       | 说明                                                                                                      |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| `preConnect(): Promise<boolean>`           | 预拉取并缓存连接配置（带 TTL，默认 60s 量级，见 `PRECONNECT_CACHE_DURATION_MS`）。失败抛出错误。          |
| `connect(): Promise<void>`                 | 建立协调器、各域控制器与 LiveKit；内部可在无有效缓存时等价调用预连接路径。                                |
| `disconnect(): Promise<void>`              | 停止采集、断开 LiveKit、结束 HTTP 会话相关清理；实例可再次 `connect()` / `reconnect()`。                  |
| `reconnect(): Promise<ConnectionSnapshot>` | 手动重连：先 `disconnect()`，再按模式刷新配置后 `connect()`。若当前状态不允许重连，返回当前快照并打日志。 |
| `dispose(): void`                          | 释放全部资源；之后不得再使用该实例。                                                                      |

**约束**：浏览器策略下，**带声音的播放**与 **麦克风采集** 建议在用户手势（点击等）回调内触发 `connect()` / `startAudioCapture()`。

### 6.2 鉴权与连接配置

| 方法                                                           | 说明                                                                                                                                            |
| -------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `setAuthToken(token: string): void`                            | 写入鉴权令牌；**Auth 模式**下供 HTTP 与 Config 拉取使用。                                                                                       |
| `updateConnectionConfig(config: DirectConnectionConfig): void` | **仅 Direct**。校验 `sfuUrl` / `clientToken` 非空后暂存，**不影响当前已连接会话**；在下次 `reconnect()` 流程中通过 `replaceDirectConfig` 生效。 |

### 6.3 媒体

| 方法                                                   | 说明                               |
| ------------------------------------------------------ | ---------------------------------- |
| `setRenderFitMode(mode: RenderFitMode): void`          | 设置画面适配模式。                 |
| `startAudioCapture(): Promise<void>`                   | 打开麦克风并通过 LiveKit 发布。    |
| `stopAudioCapture(): Promise<void>`                    | 停止麦克风发布。                   |
| `setVolume(volume: number): void`                      | 播放音量 `0..1`。                  |
| `getVolume(): number`                                  | 读取播放音量。                     |
| `mute()` / `unmute()`                                  | 播放静音控制。                     |
| `get isMuted`                                          | 是否静音。                         |
| `get isAudioCapturing`                                 | 麦克风是否处于活跃发布状态。       |
| `startCamera(): Promise<void>`                         | 开启摄像头并发布。                 |
| `stopCamera(): void`                                   | 停止摄像头。                       |
| `getCameraStream(): MediaStream \| null`               | 本地预览用流。                     |
| `getCameraTrack(): MediaStreamTrack \| null`           | 本地轨道。                         |
| `attachCameraTo(videoElement: HTMLVideoElement): void` | 将本地摄像头画⾯绑定到 `<video>`。 |

### 6.4 会话

| 方法                                              | 说明                                                                                  |
| ------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `sendTextQuestion(text: string): Promise<string>` | 发送文本问题；返回 **消息 UID**（实现上用作问答关联，与事件中的 `questionId` 对应）。 |
| `interrupt(): Promise<void>`                      | 发送打断控制事件（`control.interrupt`）。                                             |

### 6.5 状态与可观测性

| 成员                                                                       | 说明                                                                                    |
| -------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `get isConnected(): boolean`                                               | 会话是否处于已连接态。                                                                  |
| `get connectionSnapshot(): ConnectionSnapshot`                             | 同步只读快照：`http.connected`、`rtc.connected`、`rtc.hasVideoTrack`、`overall.state`。 |
| `setPerformanceMetricReporter(reporter?: PerformanceMetricReporter): void` | 设置或更新性能指标上报回调。                                                            |
| `get events(): PublicEmitterAPI`                                           | 仅 `on` / `off` / `once`。                                                              |

---

## 7. 公共事件介绍

事件通过 `client.events.on(eventName, listener)` 订阅。事件名**仅**下列白名单，其余名称会抛错。

**连接**

### `sdk:connected`

- **触发时机**：内部 `livekit` 与 `http` 连接事实变化后，聚合状态与上次不同时下发。
- **Payload**：`{ livekit: boolean; http: boolean; all: boolean }`，其中 `all === livekit && http`。
- **说明**：用于总览两路通道是否同时就绪。

### `sdk:disconnected`

- **触发时机**：任一路径断开事实导致聚合状态变化时。
- **Payload**：运行时与 `sdk:connected` 相同结构：`{ livekit: boolean; http: boolean; all: boolean }`（与类型文件中的 `reason?` 可能不一致，以运行时负载为准）。
- **说明**：可与 `connectionSnapshot` 交叉校验。

### `sdk:error`

- **触发时机**：内部错误经映射为对外安全负载时。
- **Payload**：`{ message: string; code: string }`（`code` 为 `ErrorCode` 字符串值）。

**视频**

| 事件                       | 触发时机                                | Payload                            |
| -------------------------- | --------------------------------------- | ---------------------------------- |
| `media:video:available`    | 远端视频可用（首帧等语义由 RTC 层驱动） | `undefined`                        |
| `media:video:unavailable`  | 远端视频不可用                          | `undefined`                        |
| `media:video:trackAdded`   | 视频轨道添加                            | `undefined`（内部已做 track 去重） |
| `media:video:trackRemoved` | 视频轨道移除                            | `undefined`                        |

**远端音频**

| 事件                       | 触发时机         | Payload     |
| -------------------------- | ---------------- | ----------- |
| `media:audio:trackAdded`   | 远端音频轨道添加 | `undefined` |
| `media:audio:trackRemoved` | 远端音频轨道移除 | `undefined` |

**本地麦克风**

| 事件                         | 触发时机     | Payload     |
| ---------------------------- | ------------ | ----------- |
| `media:audio:captureStarted` | 本地输入开始 | `undefined` |
| `media:audio:captureStopped` | 本地输入停止 | `undefined` |

**本地摄像头**

| 事件                   | 触发时机       | Payload     |
| ---------------------- | -------------- | ----------- |
| `media:camera:started` | 摄像头采集开始 | `undefined` |
| `media:camera:stopped` | 摄像头采集停止 | `undefined` |

**播放音量**

| 事件                        | 触发时机 | Payload              |
| --------------------------- | -------- | -------------------- |
| `media:audio:volumeChanged` | 音量变化 | `{ volume: number }` |
| `media:audio:muted`         | 输出静音 | `undefined`          |
| `media:audio:unmuted`       | 解除静音 | `undefined`          |

**会话**

| 事件                            | 触发时机     | Payload                                                                                                          |
| ------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------- |
| `conversation:question:sent`    | 问题已发送   | `{ questionId: string; text: string }`                                                                           |
| `conversation:answer:waiting`   | 等待回答     | `{ questionId: string }`                                                                                         |
| `conversation:server:message`   | 服务端消息   | `{ questionId: string; message: string; type: string }`                                                          |
| `conversation:asr:received`     | ASR 最终结果 | `{ questionId: string; text: string }`                                                                           |
| `conversation:asr:chunk`        | ASR 文本分片 | `{ questionId: string; text: string; isComplete: boolean }`                                                      |
| `conversation:answer:chunk`     | 回答文本分片 | `{ questionId: string; chunk: string }`（公开转发层仅保证 `questionId` 与 `chunk`；流式结束以 `completed` 为准） |
| `conversation:answer:completed` | 单次回答结束 | `{ questionId: string; fullAnswer: string }`                                                                     |

---

## 8. 完整使用示例

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

**Direct 模式下更换房间/token**：

```ts
client.updateConnectionConfig({ sfuUrl: 'wss://new-host', clientToken: 'new-token' });
await client.reconnect();
```

---

## 9. 视频绿幕参数调试指南

在启用绿幕前，请确保，或不做配置，sdk内部将做自动判断

- `video.renderMode = 'processed'`
- `greenScreen.enabled = true`

### 9.1 参数调试建议

1. **背景取色（`chromaKey`）**

- 建议从真实视频截图中取色
- 避免直接使用纯绿色
- 仅支持绿色调

2. **相似度（`similarity`）**

- 推荐从 `0.3` 开始逐步调整
- 数值过大可能误抠除人物细节

3. **绿色溢出抑制（`despillStrength`）**

- 用于减少人物边缘绿色反光

4. **边缘平滑（`smoothness`）**

- 可改善头发、半透明区域的融合效果

---

## 10. 常见问题

**视频无画面或黑屏**

1. 是否已为 `video.containerElement` 传入有效 DOM 节点。
2. 是否已成功 `connect()` 且 `client.isConnected === true`。
3. 是否收到 `media:video:available` 或至少 `media:video:trackAdded`（业务可据此展示加载态）。

**Auth 模式 preConnect / connect 失败**

1. `http.baseURL` 是否指向正确环境；`setAuthToken` 是否在连接前注入。
2. 打开 `debug: true` 查看控制台与 `sdk:error` 的 `code`。

**Direct 模式 reconnect 仍使用旧 token**

1. 是否在 `reconnect()` 前调用了 `updateConnectionConfig`；该方法**不会**影响当前连接，仅作用于后续重连路径。

**绿幕卡顿**

1. 适当缩小展示区域分辨率或改用 `renderMode: 'raw'` 对比验证是否为 CPU/GPU 瓶颈。

---

## 11. 错误代码

以下为 `ErrorCode` 枚举（`src/errors/ErrorCodes.ts`）的字符串值，与 `sdk:error` 及抛出 `SDKError` 的 `code` 一致。

### 11.1 HTTP 与控制面

| 代码                                | 说明与处理建议                                                          |
| ----------------------------------- | ----------------------------------------------------------------------- |
| `SDK_AUTH_TOKEN_FAILED`             | 鉴权 HTTP 失败；检查 token 与网关地址。                                 |
| `SDK_GET_LIVEKIT_CONFIG_FAILED`     | 获取房间配置失败；检查后端返回字段是否包含 `livekitUrl` / `roomToken`。 |
| `SDK_SWITCH_VIDEO_FAILED`           | 切换视频相关控制失败；结合日志排查。                                    |
| `SDK_INTERRUPT_CONVERSATION_FAILED` | 打断指令发送失败。                                                      |
| `HTTP_CONTROLLER_NOT_AVAILABLE`     | HTTP 控制器未就绪；避免在错误生命周期调用依赖 HTTP 的操作。             |

### 11.2 SDK 生命周期

| 代码                           | 说明与处理建议                                                        |
| ------------------------------ | --------------------------------------------------------------------- |
| `SDK_PRECONNECT_FAILED`        | 预连接失败；检查网络、Direct 参数是否为空、Auth token。               |
| `SDK_CONNECT_FAILED`           | 连接失败；查看前置错误。                                              |
| `SDK_INITIALIZATION_FAILED`    | 初始化阶段失败；查看控制台堆栈。                                      |
| `SDK_DISCONNECT_FAILED`        | 断开异常；可记录日志后重试新建实例。                                  |
| `SDK_RECONNECT_FAILED`         | 手动重连失败。                                                        |
| `SDK_NOT_CONNECTED`            | 未连接时调用了需要连接态的 API；先 `connect()` 或检查 `isConnected`。 |
| `SDK_INVALID_STATE_TRANSITION` | 非法状态迁移（如 Auth 模式调用 `updateConnectionConfig`）。           |
| `SDK_ERROR`                    | 通用回退错误码。                                                      |

### 11.3 LiveKit / RTC

| 代码                                        | 说明与处理建议                           |
| ------------------------------------------- | ---------------------------------------- |
| `LIVEKIT_CONNECT_FAILED`                    | 房间连接失败；校验 URL/token/TURN 网络。 |
| `LIVEKIT_SEND_VIDEO_AVAILABLE_STATE_FAILED` | 上报视频可用状态失败。                   |
| `LIVEKIT_SEND_TEXT_DATA_FAILED`             | 文本数据经 LiveKit 发送失败。            |
| `LIVEKIT_DATA_MESSAGE_PARSE_ERROR`          | 数据消息解析失败；核对协议版本。         |
| `LIVEKIT_UNPUBLISH_MICROPHONE_FAILED`       | 取消麦克风发布失败。                     |

### 11.4 音频

| 代码                                                                            | 说明与处理建议                          |
| ------------------------------------------------------------------------------- | --------------------------------------- |
| `AUDIO_CAPTURE_START_FAILED`                                                    | 麦克风启动失败；用户手势、HTTPS、权限。 |
| `AUDIO_CAPTURE_FAILED`                                                          | 采集中断。                              |
| `AUDIO_INVALID_SAMPLE_RATE` / `AUDIO_INVALID_CHANNEL` / `AUDIO_INVALID_CODEC`   | 参数与设备/协议不匹配。                 |
| `AUDIO_INVALID_HEADER_LENGTH` / `AUDIO_INVALID_TYPE` / `AUDIO_INVALID_RESERVED` | 上行打包头部非法。                      |
| `AUDIO_CONTROLLER_NOT_AVAILABLE`                                                | 控制器未创建或已释放。                  |

### 11.5 摄像头

| 代码                              | 说明与处理建议       |
| --------------------------------- | -------------------- |
| `CAMERA_CONTROLLER_NOT_AVAILABLE` | 摄像头控制器不可用。 |

### 11.6 会话与状态机

| 代码                                     | 说明与处理建议                         |
| ---------------------------------------- | -------------------------------------- |
| `CONVERSATION_CONTROLLER_NOT_AVAILABLE`  | 会话控制器不可用。                     |
| `STATE_MACHINE_INVALID_STATE_TRANSITION` | 内部状态机收到非法迁移；收集日志反馈。 |

### 11.7 工具与其它

| 代码                     | 说明与处理建议                        |
| ------------------------ | ------------------------------------- |
| `OBJECT_DISPOSED`        | 实例已 `dispose()`。                  |
| `NO_AVATARID`            | Auth 配置缺少 `avatarId`。            |
| `NO_AUTH_TOKEN`          | 缺少鉴权 token（依运行时校验）。      |
| `INVALID_CONNECT_CONFIG` | `createClient` 时 Direct 配置不合法。 |

---

## 12. 性能监控与排查

### 12.1 内置指标

默认开启（`performanceMonitor.enabled !== false`）。指标名 `PerformanceMetricName`：

| `metric`                                | 含义                           |
| --------------------------------------- | ------------------------------ |
| `connect_to_first_frame_ms`             | `connect()` 至首帧渲染完成耗时 |
| `text_send_to_text_response_ms`         | 文本发送到首段文本响应         |
| `text_send_to_audio_response_ms`        | 文本发送到首段音频响应         |
| `no_speech_report_to_audio_response_ms` | 无人声上报到音频响应           |

记录结构：`PerformanceMetricRecord`（`metric`、`durationMs`、`startedAt`、`endedAt`、可选 `questionId`）。

### 12.2 自定义上报

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

或在运行时：

```ts
client.setPerformanceMetricReporter((metric) => {
  /* 上报 OTLP / 自建 */
});
```

### 12.3 排查建议

- **首帧慢**：检查地域、TURN、是否重复失败重试 `connect()`。
- **文本响应慢**：结合后端 LLM/业务耗时与主线程阻塞。
- **音频明显慢于文本**：检查 TTS 链路及远端音频轨道订阅状态。

---

_文档版本与包版本一致：1.0.0。_
