# Facemarket 实时数字人 SDK 插件接入文档（v2.4.0）

本文档面向**插件开发者**，说明如何基于 `@sanseng/liveavatar-js-sdk` 的首个公开 Plugin API v1 开发、打包和维护插件。

如果你只是要在应用中安装已有插件，请阅读 [SDK 使用手册](./SDK_使用手册.md) 的“插件引入与配置”章节。已发布 SkyWalking 插件；其 v2.4.0 首帧能力状态见“当前已支持插件包”。

## 1. 适用范围与边界

Plugin API v1 是首个公开的可信进程内 TypeScript 扩展契约。插件可以向 SDK 已定义、从包根入口导出的版本化扩展点注册 contribution，或消费 SDK 暴露的只读 CoreService。

插件不是安全沙箱，且不能：

- 自行定义 extension point，或消费未导出的内部 token；
- 替换 RTC、transport、Conversation FSM、MessageAssembler、媒体所有权或 AV Sync 采集器；
- 取得 SDKClient、EventBus、LiveKit Room、token、HTTP controller 或内部 controller；
- 依赖插件间通信或插件提供的 Provider；当前没有公开的业务 Provider token。

SDK 保持协议解析、连接状态、会话原文和媒体管线的所有权。插件负责自己的 HTTP、鉴权、队列、重试、持久化、采样、去重和 drain。

## 2. 包结构与依赖

将 SDK 声明为 peer dependency，避免在插件中内嵌另一份 SDK：

```json
{
  "peerDependencies": {
    "@sanseng/liveavatar-js-sdk": "^2.4.0"
  }
}
```

插件应导出一个或多个 `SDKPlugin` 对象。不要从 `src/` 或其他 SDK 内部路径导入类型或 token；只从包根入口 `@sanseng/liveavatar-js-sdk` 导入。

## 3. Plugin API v1 契约

```ts
import type { SDKPlugin } from '@sanseng/liveavatar-js-sdk';

export const examplePlugin: SDKPlugin = {
  id: 'com.example.liveavatar-plugin',
  apiVersion: 1,
  version: '1.0.0',
  setup(registrar) {
    // 在这里同步注册 observe / transform / provide，或 consume CoreService。
  },
  dispose() {
    // 释放插件自己的资源；不要调用 SDK 内部对象。
  },
};
```

| 字段               | 要求                                                                                    |
| ------------------ | --------------------------------------------------------------------------------------- |
| `id`               | 当前 SDK 实例内唯一，且匹配 `[A-Za-z0-9][A-Za-z0-9._-]*`。卸载后可用同一 ID 重新安装。  |
| `apiVersion`       | 必须为字面量 `1`。它独立于各 extension point 的 major 版本。                            |
| `version`          | 可选的插件版本字符串，用于插件自己的发布和排障。                                        |
| `setup(registrar)` | 必须同步完成。返回 Promise/thenable、抛出异常或注册非法 contribution 时，整次安装回滚。 |
| `dispose()`        | 可选；卸载/SDK dispose 时调用。抛出不会中断 SDK，但会产生脱敏的 `sdk:pluginFault`。     |

`registrar` 只在 `setup()` 执行期间有效。不要保存它以便稍后注册或消费服务；setup 返回后再次调用会抛出 `SDK_PLUGIN_CONTEXT_INACTIVE`。

每个 contribution 需要一个在**同一插件内唯一**的 ID 和一个 handler。安装时会校验 point 类型、handler、contribution ID 与 Provider 冲突，然后一次性提交。

## 4. 当前可开发的扩展点

所有 token 均从 `@sanseng/liveavatar-js-sdk` 根入口导出。每个 token 的 `{ id, major, kind }` 是稳定身份；兼容扩展留在同一 major，破坏性修改会引入新的 token major。

| Token                              | 类型                                                | 用途与执行语义                                                                                                                       |
| ---------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `SDK_OPERATIONAL_LOGS_V1`          | `ObserverPoint<SDKOperationalLogRecord>`            | 观察脱敏的 SDK 运营日志。detached，不阻塞 SDK；记录不含 token、Authorization、正文、原始 DataChannel payload、媒体帧或 Error stack。 |
| `SDK_PERFORMANCE_METRICS_V1`      | `ObserverPoint<SDKPerformanceMetricsReport>`        | 订阅不可变的首帧终态指标批次；detached、有界、schema 校验且深度冻结。 |
| `SDK_CONVERSATION_INBOUND_TEXT_V1` | `TransformPoint<SDKInboundText, SDKTextProjection>` | 为下行 answer、system prompt、ASR 产生附加 representation。同步、并行、fail-open；不能修改原文、协议或 FSM。                         |
| `SDK_AVSYNC_ALERTS_V1`             | `ObserverPoint<AvSyncAlertEvent>`                   | 观察 AV Sync 告警生命周期。detached；安装 observer 不会启动分析。                                                                    |
| `SDK_AVSYNC_ANALYSES_V1`           | `ObserverPoint<AvSyncAnalysisReport>`               | 观察一次实际 analysis 的结果。每个 single-flight analysis 仅发布一次。                                                               |
| `SDK_AVSYNC_SESSION_ENDED_V1`      | `ObserverPoint<AvSyncSessionEndedEvent>`            | 观察本地 segment 封口与 worker segment close 之后的会话结束事实。                                                                    |
| `SDK_AVSYNC_DIAGNOSTICS_V1`        | `CoreServicePoint<SDKAvSyncDiagnosticsService>`     | 消费诊断快照、原始遥测和 analysis 查询服务；结果为防御性复制，插件卸载后旧 proxy 失效。                                              |

`ProviderPoint` 的通用运行时已实现，但 v2.4.0 没有对插件开放的业务 Provider token，因此插件当前不能调用 `registrar.provide()` 交付可被 SDK 使用的业务能力。

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

当前 Operational Log 和 AV Sync Observer 都采用 detached 调度：SDK 不等待 handler Promise。handler 的 throw、reject 或 timeout 会禁用该 contribution，并最多触发一次 `sdk:pluginFault`；SDK 的生产、断开、持久化和 dispose 流程仍会继续。

### 4.2 首帧性能指标与时间线

使用 `SDK_PERFORMANCE_METRICS_V1` 消费数值性能指标。每个有效 payload 的 schema 为 `sdk.performance-metrics`、schemaVersion 为 `1`，包含递增 `attemptId`、outcome `first-frame`、`connect-failed` 或 `first-frame-timeout`，以及仅含已发生阶段的 metrics。各指标从有效 connect 起点累计：配置就绪、RTC 连接、视频轨道加入、视频 metadata、首帧 loadeddata；失败和 timeout 使用各自的终态指标。缺失阶段会省略，绝不填充为 0。

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

该 point 是 detached：队列上限 100、溢出时丢弃最旧记录、handler timeout 为 30 秒且故障隔离。每个 handler 得到深拷贝并冻结的快照；不得修改它、不得让 SDK 等待它，也不得将其当作上传确认。插件自行负责传输、鉴权、批处理、重试、持久化、采样与去重。

需要时间线或 reportFrameErrors 输入时，可在 `SDK_OPERATIONAL_LOGS_V1` 中匹配脱敏事件 `performance.first-frame`。其 correlation 仅有 `{ attemptId }`；attributes 仅有 outcome、render mode、timeout 场景的 timeoutAfterRtcMs 与稀疏数值 nodes。它刻意排除 URL、token、连接配置、room/session/track/participant 标识、文本、原始 DataChannel 数据、媒体帧、DOM/LiveKit 对象、Error、message 与 stack。

当前已经发布的 SkyWalking 插件不会因 SDK 升级而开始上传；它必须在独立的兼容版本中注册该 Observer 和/或 Operational Log handler。

### 4.3 Conversation 文本 Transform

`SDKInboundText` 提供以下字段：

| 字段                      | 含义                                                                   |
| ------------------------- | ---------------------------------------------------------------------- |
| `kind`                    | `'answer'`、`'server-message'` 或 `'asr'`。                            |
| `phase`                   | `'partial'` 或 `'final'`。                                             |
| `questionId` / `sequence` | 对应会话和可选协议序号。                                               |
| `rawDelta`                | answer chunk 的真实当前增量；完成回答、server message 和 ASR 为 `''`。 |
| `rawText`                 | 当前累计完整 answer，或完整 server message / ASR 文本。                |

handler 必须同步返回 `SDKTextProjection`：

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

`mediaType` 必须是非空字符串；`value` 与可选 `metadata` 必须是 JSON-safe 值。多个插件或 contribution 即使使用相同 `mediaType` 也会保留，按成功安装顺序和 setup 注册顺序出现在公开 Conversation 事件的 `representations` 中。

该 point 是 `sync + parallel + fail-open`：handler 返回 Promise/thenable、抛出异常或返回非法 projection 后会被禁用；原文、Conversation FSM、其他 contribution 和公开事件继续正常工作。

### 4.4 AV Sync 诊断与告警

CoreService 必须在 `setup()` 中消费，异步上传工作放在 Observer handler 中：

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

可调用的诊断服务为：

- `captureRemoteAvSyncDiagnostics()`
- `getAvSyncRawTelemetry(query?)`
- `getAvSyncAnalysis(query?)`

插件卸载会 abort 正在运行的异步 Transform/Provider；所有 execution context 的 `signal` 都应被 handler 观察。若插件在卸载后调用旧 CoreService proxy，调用会抛出 `SDK_PLUGIN_CONTEXT_INACTIVE`。

## 5. 安装、热插拔与故障处理

应用可在构造期通过 `ClientOptions.plugins` 安装，也可在运行期调用 `client.installPlugin(plugin)`。两种方式使用相同的事务安装流程。

- contribution 的执行顺序是插件成功安装顺序，再按该插件的 setup 注册顺序；无 priority。
- 当前一次 observe、transform 或 provider 调用固定使用开始时的 Registry 快照。安装/卸载只从下一次操作生效，不会重放历史事件。
- 卸载会原子移除全部 contribution、丢弃未开始回调、abort 在途异步 Transform/Provider、失效 retained registrar/CoreService proxy，再调用 `dispose()`。
- `SDKPluginInstallResult` 的拒绝原因包含重复 ID、不支持的 API 版本、非法插件、setup 失败、非法 contribution 和 Provider 冲突。拒绝单个插件不会影响 SDKClient 构造或已安装插件。
- `sdk:pluginFault` 只包含插件/贡献 ID、可选 point 描述、phase、reason 和时间戳；不包含 Error、stack、token、业务正文、原始协议 payload 或插件私有数据。该事件不进入 Operational Log Observer，避免日志递归。

## 6. 当前已支持插件包

| 插件包 | 用途                                                                                                | Plugin API | 状态       |
| ------ | --------------------------------------------------------------------------------------------------- | ---------- | ---------- |
| `@sanseng/liveavatar-js-sdk-skywalking` | 已发布的 SkyWalking 集成，当前用于 A/V Sync 诊断与 SDK error；已发布版本不会因 SDK 升级自动消费 v2.4.0 首帧 observer 或日志。 | 需插件独立兼容版本 |

SkyWalking 插件后续独立版本注册 `SDK_PERFORMANCE_METRICS_V1` 和/或 `performance.first-frame` 后，才可自行上报 setPerformance 或 reportFrameErrors。SDK 不拥有其鉴权、队列、重试或上传。

## 7. 发布前检查

- 使用 SDK 根入口导入，且将 SDK 声明为 peer dependency。
- `setup()` 不返回 Promise；每个 contribution ID 在插件内唯一。
- Observer/async handler 正确处理 `AbortSignal`，不让上传失败影响 SDK 核心路径。
- Conversation projection 使用 JSON-safe 值，且不尝试修改原始输入。
- 不记录或上传 SDK 不应暴露的 token、正文、原始 DataChannel payload、媒体帧或 Error stack。
- 在目标 SDK 版本上验证安装、拒绝结果、卸载、重复安装、dispose 和 `sdk:pluginFault` 行为。
