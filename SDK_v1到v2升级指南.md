# Facemarket 实时数字人 SDK v1 → v2 升级指南

> **已验证的升级路径**：从 `@sanseng/liveavatar-js-sdk` **v1.3.2 或 v1.3.3** 升级到 **v2.3.0**。
>
> **更早 v1 版本**：尚未声明已直接验证。请先升级到 v1.3.2 或 v1.3.3 基线，再按本指南升级。
>
> **读者**：Web 应用集成方、负责 LiveKit/数据通道的后端维护者，以及使用对话事件、麦克风音量或 A/V 诊断能力的业务方。

本指南说明升级所需的验证与建议改造。当前 v2.3.0 保留 v1.3.2–v1.3.3 末期基线的主要连接配置和对话事件兼容字段；请不要依据历史 v2.0.0 的短期参与者过滤规则单独改造后端。

## 1. 升级结论

| 范围 | 是否必须改代码 | 迁移结论 |
| --- | --- | --- |
| Direct / Auth 连接配置 | 否 | `connectConfig`、`setAuthToken()`、`updateConnectionConfig()` 的模式约束保持不变。按原有模式完成连通性回归。 |
| DataChannel 发送者身份 | 否，但必须联调验证 | v2.3.0 同时接收 `renderer_*` 与 `coordinator_*` 身份发送的数据通道消息；无需为本次升级强制切换后端前缀。 |
| 对话事件文本字段 | 建议 | 旧字段仍可用，但新代码应改用 `rawDelta` / `rawText` 与 `representations`。 |
| 断连和错误处理 | 是，如业务按错误码分支 | 被动断连应以 `sdk:disconnected` 为准；已建立会话的严重运行期故障使用 `LIVEKIT_DISCONNECTED`。 |
| 麦克风音量阈值 | 是，如业务使用该数值 | 音量现在是本地 PCM 的线性 RMS，不再是 LiveKit 私有字段或固定兜底值。 |
| A/V Sync 与插件 | 否 | 均为 v2 可选新能力；只有接入时才需要采用新 API。 |

## 2. 升级前准备

1. 将当前生产依赖、锁文件和 v1.3.2 或 v1.3.3 集成回归结果归档，确保可通过恢复原锁文件和依赖版本回退。
2. 升级包并重新安装依赖：

   ```bash
   npm install @sanseng/liveavatar-js-sdk@2.3.0
   ```

3. 使用与生产一致的浏览器、Direct/Auth 配置、LiveKit 房间和后端参与者身份执行本指南第 7 节的验证矩阵。
4. 不要依赖 SDK 内部模块、私有 LiveKit 字段或 SDK 内部创建的媒体元素；它们不是升级兼容契约。

## 3. 连接与后端协议

### 3.1 保持现有 Direct 或 Auth 配置

Direct 模式继续由调用方提供 `sfuUrl` 和 `userToken`；令牌需要更新时，先调用 `updateConnectionConfig()`，再执行下一次 `preConnect()` / `connect()` 或 `reconnect()`。Auth 模式继续由后端下发房间配置，业务在首次连接前提供 `authToken`，并通过 `reconnect()` 重新拉取配置。

本次升级不要求将一种连接模式改为另一种。请继续遵守 Auth 模式不能调用 `updateConnectionConfig()` 的限制。

### 3.2 验证 DataChannel 消息

当前 SDK 对 `RoomEvent.DataReceived` 同时接受以下远端身份：

- `renderer_*`
- `coordinator_*`

因此，v1.3.2–v1.3.3 后端可保持现有 `coordinator_*` 文本/协议消息发送方式。联调时分别验证生产实际使用的身份能够触发问答、ASR 和服务端消息事件；来自其他身份的消息仍会被 SDK 忽略。

## 4. 应用代码迁移

### 4.1 将对话事件改为规范文本字段

v2 保留旧字段以兼容 v1.3.2–v1.3.3，但它们已标记为 TypeScript 弃用字段。新代码应使用规范字段，避免在未来 major 版本删除别名时再次改造。

| 事件 | v1 兼容字段 | v2 推荐字段 |
| --- | --- | --- |
| `conversation:answer:chunk` | `chunk` | `rawDelta`、`rawText`、`representations` |
| `conversation:answer:completed` | `fullAnswer` | `rawText`、`representations` |
| `conversation:server:message` | `message` | `rawText`、`representations` |
| `conversation:asr:chunk` / `conversation:asr:received` | `text` | `rawText`、`representations` |

```ts
// v1：仍可运行，但字段已弃用
client.events.on('conversation:answer:chunk', ({ chunk }) => {
  appendAnswer(chunk);
});

// v2：使用流式增量、完整文本和可选插件投影
client.events.on('conversation:answer:chunk', ({ rawDelta, rawText, representations }) => {
  appendAnswer(rawDelta);
  cacheCompleteAnswer(rawText);
  renderRepresentations(representations);
});
```

在 v2.3.0 中，`chunk === rawDelta`、`fullAnswer === rawText`、`message === rawText`，ASR 的 `text === rawText`。`representations` 为按插件注册顺序排列的 JSON-safe 投影；未安装文本转换插件时它可以为空数组。

### 4.2 调整断连与错误处理

`sdk:disconnected` 始终是被动断连的生命周期通知，payload 的 `reason` 用于排障；不要把 `sdk:error` 当作所有断连的唯一触发条件。

v2.3.0 中：

- `LIVEKIT_CONNECT_FAILED` 继续用于 Room 建连或既有操作失败路径。
- 已成功连接的 Room 因不可恢复运行期故障（传输、媒体、协议、Agent 或 SIP trunk）终止时，会额外发送 `sdk:error`，错误码为 `LIVEKIT_DISCONNECTED`。
- 服务端结束、重复身份、房间关闭/删除、迁移、用户拒绝等普通会话结束只发送 `sdk:disconnected`，不会再额外发送 `sdk:error`。

```ts
client.events.on('sdk:disconnected', ({ reason, allConnected }) => {
  if (!allConnected) scheduleReconnect(reason);
});

client.events.on('sdk:error', ({ code, message }) => {
  if (code === 'LIVEKIT_DISCONNECTED') reportRuntimeRtcFault(message);
});
```

### 4.3 重新校准麦克风音量阈值

`getMicrophoneAudioLevel()` 的返回签名仍为 `number | null`，但数值语义已改为最近一个 4096-sample 本地 PCM 帧的线性 RMS：

- 无活动麦克风轨道时返回 `null`。
- 启动采集但尚未收到首帧时返回 `0`；停止采集或释放帧发送链后也返回 `0`。
- 有本地帧时返回 `0..1` 的实际 RMS，不再读取 LiveKit 私有 `_volume`，也不再返回固定 `0.5`。

如 v1 业务以 `0.5` 作为“麦克风可用”或静音阈值，必须改为先处理 `null`，再根据真实 RMS 重新标定阈值。

## 5. 可选的 v2 能力

### 5.1 A/V Sync 诊断

A/V Sync 不会因升级自动开启。只有显式配置 `debug.avSync: true` 或 `debug.avSync.enabled: true` 才启用后台采样。

```ts
const client = createClient({
  // ...现有连接、音频和视频配置
  debug: { avSync: { enabled: true, intervalMs: 2000, historyLimit: 60 } },
});
```

`getAvSyncRawTelemetry(query?)` 和 `getAvSyncAnalysis(query?)` 都是异步方法。读取原始日志时必须 `await`，每次只返回一页，并使用 `page.nextCursor` 获取下一页：

```ts
const telemetry = await client.getAvSyncRawTelemetry({ pageSize: 200 });
const older = telemetry.page.nextCursor
  ? await client.getAvSyncRawTelemetry({ cursor: telemetry.page.nextCursor })
  : null;
```

监听 `sdk:avSyncAlert` 时，应以不透明 `alertId` 保存活跃事件：`raised` 和 `updated` 都表示事件仍活跃，只有 `recovered` 才移除该事件。

### 5.2 Plugin API v1

Plugin API v1 是 v2 首次公开的插件契约，不存在需要从已发布插件 API 转换的 v1 格式。只有需要扩展日志、对话文本或 A/V Sync 时，才通过 `ClientOptions.plugins` 或 `installPlugin()` 接入 `SDKPlugin`；完整约束见[插件接入文档](./SDK_插件接入文档.md)。

### 5.3 绿幕运行期更新

仅在连接期间调用 `updateGreenScreenOptions()`。传入 `enabled: true` / `false` 会立即在 processed Canvas 与 raw video 路径间切换，取色、阈值、边缘平滑、despill 和 `isBackgroundKeying` 等其他参数也会即时生效。所有运行期补丁只作用于当前连接；`reconnect()` 后会恢复构造参数或本次 Auth 配置。

v2.3.0 新增 `isBackgroundKeying`（默认 `false`）：启用后，SDK 对 `32 × 18` 采样帧的四条边分别检查 key 色覆盖率，至少三条边各有 `75%` 命中才执行抠图。`chromaKey` 只接受高色度绿色（色相 `80°..160°`）或蓝色（色相 `200°..260°`）；无效元组回退到 `[0, 255, 0]`。其余默认值为 `similarity: 0.4`、`smoothness: 0.25`、`despillStrength: 1.15`。

## 6. 上线前验证清单

| 场景 | 通过条件 |
| --- | --- |
| 安装与类型检查 | 应用以 v2.3.0 依赖成功安装、构建和启动，无内部模块或私有字段引用。 |
| Direct 连接 | 使用现有 URL/token 成功连接、断开和重连；刷新 token 后连接到目标房间。 |
| Auth 连接 | `setAuthToken()`、首次连接和 `reconnect()` 的服务端配置刷新成功。 |
| DataChannel | 生产实际的 `coordinator_*` 或 `renderer_*` 发送者能够收到问答、ASR 和服务端消息。 |
| 对话 UI | 规范字段与兼容字段均符合预期；新实现只消费 `rawDelta` / `rawText`。 |
| 远端媒体 | 音频和视频共同播放，替换轨道、断连和重连后无停滞或重复绑定。 |
| 麦克风 | 用户手势内启动采集；`null`、首帧 `0` 与实际 RMS 阈值行为符合业务预期。 |
| 断连处理 | 普通会话结束进入 `sdk:disconnected`；严重运行期断连能识别 `LIVEKIT_DISCONNECTED`。 |
| 可选 A/V Sync | 已启用时可 await 分页结果，告警状态仅在 `recovered` 时按 `alertId` 清除。 |
| 绿幕 | 连接后可验证 `enabled` 在 raw / processed 间即时切换，其他参数即时生效；重连后运行期改动重置。 |

## 7. 回退策略

在灰度阶段保留 v1.3.2 或 v1.3.3 的锁文件或显式依赖版本。若上述连接、协议或媒体回归未通过，停止扩大 v2.3.0 流量，恢复已验证的 v1 基线依赖和对应构建产物；不要通过调用 SDK 私有 API 或强行修改内部媒体元素绕过问题。收集 `sdk:disconnected.reason`、`sdk:error.code`、浏览器版本和后端参与者身份后，再完成定向排查。

## 8. 相关文档

- [SDK 使用手册](./SDK_使用手册.md)
- [SDK User Manual](./SDK_User_Manual.md)
- [插件接入文档](./SDK_插件接入文档.md)
- [Plugin Integration Guide](./SDK_Plugin_Integration_Guide.md)
- [CHANGELOG](https://github.com/newportAI-lab/liveavatar-js-sdk/blob/main/CHANGELOG.md)
