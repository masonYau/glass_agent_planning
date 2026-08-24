# Rokid Glasses 视听采集协议与状态契约 v1

版本：1.1.0-draft
日期：2026-08-24  
状态：M3 开发基线（M0 真机验证和 M3 契约测试后冻结 1.0）
目标硬件：Rokid Glasses / Glass3，优先使用企业版 Glass3 SDK  
适用范围：眼镜端应用、Android 手机端应用、云端 Ingestion/ASR/OCR/索引服务

## 1. 文档目的

本文把 `confirm` 目录中的 A0/A1/A2 音频方案和 L0/L1/L3 视觉方案收口为可编码、可测试、可演进的协议与状态契约。

本文重点回答：

1. 手机、眼镜和云端分别拥有哪个状态的最终解释权。
2. 控制命令如何处理重复、乱序、延迟、断线和进程重启。
3. A1 尚未创建会话时，音频帧如何编号并在进入 A2 后绑定会话。
4. 音频实时流、眼镜 spool、手机分片和迟到修复如何同时保持不可变与可追溯。
5. 图片如何完成崩溃安全提交、批量传输、双 ACK 和最终清理。
6. 隐私暂停如何抢占所有自动采集，并阻止旧命令重新打开麦克风或相机。
7. 云端如何做幂等接入、完整性判断和版本化重处理。

本文不定义完整产品 UI、RAG 问答、计费、多租户运营后台，也不承诺尚未经过真机验证的功耗数值。

### 1.1 M3 工程边界

本文所称 M3 是“可验证的采集链路工程 MVP”，不是已经具备完整检索体验的产品 MVP。M3 必须完成：

1. 眼镜 `GLASSES` 单音频源的 A0/A1/A2、预录、60 秒分片、连续滚动 spool、range ACK 和显式 gap。
2. A2 期间 L1 保底拍照、关键词/规则触发 L3、journaled bundle、P2P 批传和 ACK1。
3. 隐私 fence 对音频、视觉、在途 SDK 回调和旧租约具有抢占权；恢复必须是明确用户动作。
4. 手机端持久化 reducer、崩溃恢复、命令幂等、乱序拒绝和可观测性。
5. 云端可以先使用幂等 Ingestion stub 验证对象和 final manifest；完整 ASR/OCR/索引属于 M5。

M3 不强制自动切换手机麦克风。协议仍定义多音源时间线；如果能力协商返回 `phone_mic_fallback=false`，眼镜链路中断必须生成显式 gap，不得伪造连续音频。LLM 触发属于 M4，M3 只实现可审计的关键词/规则触发器。

## 2. 规范优先级

开发时按以下优先级解释需求：

1. 本文：协议、状态、数据模型、幂等和异常恢复的唯一开发基线。
2. `minimal_text_capture_architecture.md`：架构目标、模块边界和 MVP 范围。
3. `audio_sequence_diagrams_viewer.html` 与 `image_layered_capture_flow_viewer.html`：流程说明和评审视图。
4. `sound/`、旧版 Android 设计和示例代码：仅作历史参考，不得覆盖本文。

如旧资料仍包含“30 秒 Scene Probe 上传”“隐私恢复后续接旧会话”“手机逐张下发 takePhoto”等逻辑，一律以本文为准：

- L3 自动触发只使用手机端最近 10-20 秒滚动音频语义。
- A2 中进入 A0 时立即封存旧会话；恢复后下一次 A2 创建新 `conversation_id`。
- 手机下发 L1/L3 采集租约；眼镜在租约内用本地定时器拍照，不逐张等待手机命令。

## 3. 规范用语

- **必须（MUST）**：违反即视为协议不兼容或验收失败。
- **不得（MUST NOT）**：禁止行为。
- **应该（SHOULD）**：默认实现；偏离时必须有 ADR 和测试证据。
- **可以（MAY）**：兼容实现可自行决定。
- **持久化（durable）**：数据和状态已写入崩溃后可恢复的存储，不能只存在于内存或 SDK 回调中。
- **业务 ACK**：本协议定义的持久化确认，不等于蓝牙/P2P/文件 SDK 的发送完成回调。
- **本地硬门**：眼镜端无需等待手机即可执行的隐私、租约过期、存储、温度和电量降级规则。

## 4. Rokid SDK 能力映射

### 4.1 已确认的厂商能力

Rokid Glass3 SDK 提供手机端和眼镜端双端 SDK，支持蓝牙、Wi-Fi P2P、消息、文件、拍照、音频采集和音视频流。本文采用以下逻辑映射：

| 逻辑能力 | Rokid 侧候选能力 | 本协议要求 |
|---|---|---|
| 眼镜端应用注册 | `GlassSdk.bindSecurityService`、`registerClient` | 手机和眼镜 `clientId` 必须匹配 |
| 手机端 SDK 初始化 | `PSecuritySDK.initSDK` | 初始化成功后才能建立控制器 |
| 低频控制消息 | BLE/经典蓝牙文本消息 | 只承载命令、ACK、心跳和状态快照 |
| P2P 建连 | 手机端 P2P Client、眼镜端 P2P Go | 蓝牙先连接，P2P 后协商 |
| 拍照 | 眼镜端 `IMediaServer.takePhoto` + `PhotoFileCallback` | 由眼镜本地租约执行器调用 |
| 音频采集 | `IMediaServer.startAudioRecord` + `AudioCallback` | 通过适配器补充序号、时间和 codec 元数据 |
| 实时音频传输 | SDK 音频流或自定义二进制流 | 上层必须收到本文定义的 `AudioFrameV1` |
| 文件发送 | `IGlassFileOperate.sendFile` | SDK 完成后仍需 SHA 校验、手机事务和 ACK1 |
| 文件接收 | 手机端 `FileReceiveV2Listener` | `onComplete` 仅表示传输结束，不代表 PHONE_COMMITTED |
| 相机关闭 | `IMediaServer.closeCamera` | A0、租约撤销和硬降级时调用并等待结果 |

### 4.2 不允许直接固化的厂商假设

以下内容必须由 `RokidGlassDeviceAdapter` 隔离，业务状态机不得直接依赖 SDK 细节：

- SDK 音频回调的实际采样率、声道、位宽、帧长和是否经过处理。
- SDK 内置音频流使用 P2P 还是经典蓝牙，以及能否附加业务帧头。
- 眼镜端 App 在熄屏、锁屏、系统清理和 OTA 后的后台存活行为。
- 相机输出目录、文件可见性和应用层加密文件是否可直接走 SDK 文件发送。
- 电池、充电和温度是否能通过 Rokid SDK 直接读取；读取不到时使用 Android 系统接口并标注来源。
- `setKeepP2PConnect(true)` 的持续功耗和热影响。

这些项目进入第 31 节的 M0 真机验收，不得在业务层写死。

## 5. 端侧职责和状态所有权

| 状态/数据 | 最终解释权 | 副本/观察者 |
|---|---|---|
| A0/A1/A2 音频业务状态 | 手机 `AudioSessionManager` | 眼镜保存已应用快照 |
| L0/L1/L3 目标等级 | 手机 `VisualLevelScheduler` | 眼镜执行不高于目标的有效等级 |
| 隐私禁用 | 手机发起，眼镜本地可独立收紧 | 任一端禁用均生效，放开必须由手机重新授权 |
| 租约是否仍有效 | 眼镜本地单调时钟 | 手机通过心跳续租 |
| 眼镜电量/温度/存储硬约束 | 眼镜 | 手机接收遥测后更新策略 |
| 文件是否 CAPTURED | 眼镜持久化存储 | 手机读取 manifest |
| 文件是否 PHONE_COMMITTED | 手机持久化存储 | 眼镜通过 ACK1 获知 |
| 文件是否 CLOUD_ACKED | 云端接入层 | 手机通过 ACK2 获知 |
| 云端 ASR/OCR 最终版本 | 云端处理管线 | 手机查询或订阅 |

两端状态冲突时执行“更严格者胜出”：

```text
A0/隐私禁止
  > 眼镜本地硬降级
  > 会话结束
  > L3
  > L1
  > L0
```

手机可以请求升级；眼镜只能在租约和本地硬门允许时执行。眼镜可以自行降级，但不得自行从 L0 升级到 L1/L3。

## 6. 标识符契约

所有 ID 使用 UUIDv7 字符串；设备硬件标识不得直接作为云端业务 ID。

| 字段 | 生命周期 | 生成方 | 用途 |
|---|---|---|---|
| `device_id` | 配对设备生命周期 | 手机注册服务 | 脱敏后的业务设备 ID |
| `controller_id` | 手机 App 安装生命周期 | 手机 | 标识唯一控制器实例 |
| `boot_id` | 音频来源设备每次开机或采集进程全新启动 | 当前音频来源 | 区分单调时钟和序号重置；M3 由眼镜生成 |
| `stream_id` | 每次音频来源启动或重建采集 | 当前音频来源 | 单个来源内 A1/A2 共用；不同来源不得续序号 |
| `conversation_id` | 每次进入 A2 | 手机 | 正式会话身份；A1 时为空 |
| `binding_id` | 每次会话与音频流建立归属关系 | 手机 | 关联 OPEN/CHECKPOINT/CLOSE 绑定事件 |
| `privacy_lock_id` | 每次手机或眼镜本地进入隐私态 | 发起端 | 防止普通启动命令绕过显式恢复 |
| `command_id` | 每条控制命令 | 手机 | 去重和审计 |
| `lease_id` | 每次音频/L1/L3 租约 | 手机 | 租约续期、撤销和过期 |
| `decision_id` | 每次 L3 触发判断 | 手机 | 关联语义判断、策略门和 burst |
| `capture_id` | 每张成功提交的图片 | 眼镜 | 图片幂等键 |
| `capture_attempt_id` | 每次计划拍照 tick | 眼镜 | 记录成功、跳过和失败 |
| `chunk_id` | 每个手机音频基础分片 | 手机 | 音频上传幂等键 |
| `repair_id` | 每个迟到 spool 修复段 | 手机 | 不可变补偿数据幂等键 |
| `event_id` | 每个状态/策略/缺口事件 | 事件发生端 | 跨端审计和重放 |
| `clock_map_id` | 每次时钟映射更新 | 手机 | 单调时钟对齐 |

### 6.1 音频源身份

`AudioFrameV1` 中的 `boot_id + stream_id + source_seq` 是跨 A1/A2 不变的原始身份。`boot_id` 表示产生该帧的采集进程启动实例：M3 的 `GLASSES` 源由眼镜生成；未来启用 `PHONE` 源时由手机采集进程生成。`stream_id` 使用 UUIDv7，因此不同来源不得尝试续用对方的 source_seq。

支持的 `source_kind`：

- `GLASSES`：M3 必须实现。
- `PHONE`：schema 枚举保留；M3 未声明该能力时可以返回 `AUDIO_SOURCE_UNSUPPORTED`，自动兜底实现可选。

同一音频 chunk 只能包含一个 `boot_id + stream_id + source_kind`。音源切换必须关闭当前 chunk，再打开新来源 chunk；不得把两套 source_seq 混入一个基础 chunk。

### 6.2 A1 帧与会话绑定

A1 阶段尚无 `conversation_id`，因此音频帧必须使用：

```text
stream_id + boot_id + source_seq
```

作为原始身份，`conversation_id` 必须允许为空。进入 A2 时，手机不修改历史帧，而是写入追加式 `ConversationBindingEventV1`。绑定投影具有三个阶段：

```text
OPEN -> CHECKPOINT* -> CLOSE
```

`OPEN`：

```json
{
  "schema": "memory.conversation-binding-event.v1",
  "event_id": "0191...",
  "binding_id": "0191...",
  "event_type": "OPEN",
  "conversation_id": "0191...",
  "source_kind": "GLASSES",
  "stream_id": "0191...",
  "boot_id": "0191...",
  "first_source_seq": 88012,
  "first_voice_source_seq": 88112,
  "preroll_ms": 2000,
  "control_epoch": 42,
  "created_phone_monotonic_ms": 18531000
}
```

`CHECKPOINT` 只推进手机已持久化的连续位置：

```json
{
  "schema": "memory.conversation-binding-event.v1",
  "event_id": "0191...",
  "binding_id": "0191...",
  "event_type": "CHECKPOINT",
  "committed_through_source_seq": 123009,
  "created_phone_monotonic_ms": 18591000
}
```

`CLOSE` 固化最终期望边界：

```json
{
  "schema": "memory.conversation-binding-event.v1",
  "event_id": "0191...",
  "binding_id": "0191...",
  "event_type": "CLOSE",
  "last_expected_source_seq": 123109,
  "end_reason": "AUTO_SILENCE",
  "end_boundary_quality": "EXACT",
  "created_phone_monotonic_ms": 18593000
}
```

绑定不变量：

1. 同一 `boot_id + stream_id` 同时只能有一个未关闭 binding。
2. 新 `OPEN` 前必须先 `CLOSE` 旧 binding；眼镜重启时旧 binding 以 `DEVICE_RESTART` 关闭或由手机标记尾部未知。
3. CHECKPOINT 和 CLOSE 的序号只能单调增加，不得小于 `first_source_seq`。
4. OPEN 至 CLOSE 的闭区间属于该 conversation；后续 A2 帧携带的 `conversation_id` 仅为冗余，不是最终解释权。
5. 手机事务必须先提交 OPEN 和预录选择结果，再对外宣布 A2；眼镜收到 OPEN 后将后续 spool 标为“已绑定优先保留”。
6. 每个 event 以 `event_id` 幂等；相同 `binding_id + event_type` 出现冲突边界时隔离并报警。

### 6.3 多音源会话时间线

未来启用手机麦克风时，每个来源保持独立序号，通过 `AudioTimelineSegmentV1` 映射到会话时间线：

```json
{
  "schema": "memory.audio-timeline-segment.v1",
  "timeline_segment_id": "0191...",
  "conversation_id": "0191...",
  "source_kind": "PHONE",
  "boot_id": "0191...",
  "stream_id": "0191...",
  "first_source_seq": 0,
  "last_source_seq": 4200,
  "conversation_start_ms": 301200,
  "conversation_end_ms": 385200,
  "alignment_method": "AUDIO_CORRELATION",
  "offset_us": -18300,
  "uncertainty_us": 12000,
  "time_quality": "GOOD"
}
```

音源切换时优先保留 1-2 秒重叠并使用相关性对齐；没有重叠时可以使用 ClockMap，但必须标记 `time_quality=DEGRADED`。M3 未启用 `PHONE` 源时不得创建伪造的 PHONE segment，只记录 GLASSES missing range。

## 7. 时间契约

### 7.1 基本规则

1. 排序、租约和超时必须使用各端单调时钟，不得使用 UTC 判断租约是否过期。
2. 所有眼镜事件必须携带 `boot_id + source_monotonic_us`。
3. UTC 只用于展示、检索和跨设备粗对齐，由手机根据时钟映射生成。
4. 眼镜重启后必须创建新 `boot_id`，`source_seq` 可以重置为 0。
5. 同一 `boot_id + stream_id` 内，`source_seq` 必须严格递增，不得复用。

### 7.2 时钟同步

手机每次连接后执行四时间戳同步，并在连接稳定时每 5 分钟刷新：

```text
t0 = 手机发送时刻
t1 = 眼镜接收时刻
t2 = 眼镜发送响应时刻
t3 = 手机接收时刻

rtt = (t3 - t0) - (t2 - t1)
offset = ((t1 - t0) + (t2 - t3)) / 2
```

`ClockMapV1` 必须记录：

```json
{
  "schema": "memory.clock-map.v1",
  "clock_map_id": "0191...",
  "boot_id": "0191...",
  "phone_monotonic_ms": 18530000,
  "glass_monotonic_ms": 923000,
  "offset_ms": 17607000,
  "rtt_ms": 18,
  "uncertainty_ms": 9,
  "phone_utc": "2026-08-24T18:30:00.000+08:00"
}
```

当 `uncertainty_ms > 100` 时，跨模态证据必须标记 `time_quality=DEGRADED`；不得假装精确对齐。

## 8. 控制协议

### 8.1 传输选择

- 蓝牙/BLE：启动、隐私停止、租约、状态查询、心跳、P2P 建连协商。
- 实时音频：通过能力协商选择 `CLASSIC_BT_AUDIO` 或 `P2P_STREAM`；无论底层通道如何，上层都必须收到相同的 `AudioFrameV1`。
- P2P 文件通道：spool、图片和 manifest 文件；不得用 BLE 发送大媒体文件。
- A0/隐私停止命令应该在所有已连接控制通道上并发发送；眼镜依靠命令序号去重。
- 数据通道断开不得阻塞 A0 命令；控制面必须保留蓝牙路径。

### 8.2 控制信封

所有命令使用同一个版本化信封：

```json
{
  "schema": "memory.control-command.v1",
  "command_id": "0191...",
  "controller_id": "0191...",
  "device_id": "0191...",
  "control_epoch": 42,
  "command_seq": 1071,
  "policy_revision": 18,
  "command_type": "INSTALL_VISUAL_LEASE",
  "issued_phone_monotonic_ms": 18532000,
  "ttl_ms": 10000,
  "clock_map_id": "0191...",
  "max_clock_uncertainty_ms": 100,
  "payload": {},
  "auth": {
    "key_id": "pairing-key-3",
    "nonce": "base64...",
    "signature": "base64..."
  }
}
```

`clock_map_id` 对会启动/升级采集的普通命令为必填。`HELLO`、`REQUEST_STATE_SNAPSHOT`、`SYNC_CLOCK` 在建立 ClockMap 前允许为空，但这些命令不得产生采集副作用；`ENTER_PRIVATE` 按 fail-closed 规则处理。

### 8.3 顺序与 fencing

眼镜必须持久化最后应用的：

```text
controller_id + control_epoch + command_seq
```

处理规则：

1. 先验证 schema、device/controller 身份、签名和 nonce；未认证请求不得通过 command_id 查询历史 ACK。
2. 再按 `command_id` 查询持久化幂等表；已处理命令返回原 ACK，不再受后续 command_seq 变化影响。
3. `control_epoch` 小于已保存值：拒绝，返回 `STALE_EPOCH`。
4. `control_epoch` 相同且 `command_seq` 小于已保存值：拒绝，返回 `STALE_COMMAND`。
5. `control_epoch` 更大：进入重同步，先使所有旧租约失效，再应用命令。
6. `ENTER_PRIVATE` 必须先持久化新 epoch 和 privacy lock，再停止采集；崩溃恢复后不得回到旧租约。
7. 会启动、续租或升级采集的普通命令使用 `clock_map_id` 将签发时刻换算到眼镜单调时钟；映射缺失、过旧或 uncertainty 超过 `max_clock_uncertainty_ms` 时返回 `CLOCK_UNCERTAIN`，不得猜测执行。HELLO/状态查询/SYNC_CLOCK 只能同步，不能借此启动采集。
8. `ENTER_PRIVATE` 是 fail-closed 命令：只要 epoch 更新且签名有效，即使时钟映射不可用也必须应用；TTL 只影响普通升级/启动命令。

手机进入 A0 时必须增加 `control_epoch`。因此，A0 之前发出但延迟到达的 L1/L3 命令无法重新打开相机。

### 8.4 命令 ACK

```json
{
  "schema": "memory.control-ack.v1",
  "command_id": "0191...",
  "device_id": "0191...",
  "control_epoch": 42,
  "command_seq": 1071,
  "status": "APPLIED",
  "error_code": null,
  "applied_glass_monotonic_ms": 924188,
  "state_revision": 388,
  "effective_audio_state": "LISTENING",
  "effective_visual_level": "L0_OFF"
}
```

`status` 枚举：

- `APPLIED`
- `ALREADY_APPLIED`
- `REJECTED`
- `EXPIRED`
- `DEGRADED_APPLIED`

### 8.5 v1 命令集合

| 命令 | 作用 | 必需 payload |
|---|---|---|
| `HELLO` | 建立协议会话 | 支持的 schema、SDK/固件/App 版本 |
| `REQUEST_STATE_SNAPSHOT` | 请求眼镜当前持久化状态 | 无 |
| `SYNC_CLOCK` | 建立时钟映射 | `t0_phone_monotonic_ms` |
| `ENTER_PRIVATE` | 强制 A0/L0 并设置隐私 fence | `reason_code`、`privacy_fence` |
| `RESUME_FROM_PRIVATE` | 用户明确解除隐私锁并进入 A1 | privacy_lock_id、user_action_id、新音频租约 |
| `START_LISTENING` | 非隐私态启动或重启 A1 音频 | codec、lease、spool 上限 |
| `OPEN_CONVERSATION_BINDING` | A1→A2，打开 source_seq 绑定 | `ConversationBindingEventV1(OPEN)` |
| `CHECKPOINT_CONVERSATION_BINDING` | 推进手机已持久化连续位置 | `ConversationBindingEventV1(CHECKPOINT)` |
| `ENTER_END_PENDING` | 进入会话结束候选 | 静音、上下文原因 |
| `CANCEL_END_PENDING` | 人声恢复，回 A2 active | conversation_id |
| `FINALIZE_CONVERSATION` | 封存会话并关闭 binding | `ConversationBindingEventV1(CLOSE)`、end_reason |
| `INSTALL_VISUAL_LEASE` | 安装 L1 本地拍照计划 | interval、首拍延迟、租约 |
| `START_VISUAL_BURST` | 安装 L3 短 burst | fps、duration、decision、budget token |
| `RENEW_LEASE` | 续期音频或 L1/L3 控制租约 | lease_id、extend_by_ms |
| `REVOKE_VISUAL` | L1/L3→L0 | reason_code |
| `REQUEST_BATCH` | 请求 manifest/媒体批量回传 | ACK 游标、时间窗、类型 |
| `ACK_PHONE_COMMITTED` | ACK1，允许眼镜清理媒体 | object_id、sha256 |
| `ACK_AUDIO_RANGES` | 确认手机已持久化实时/补偿帧 | boot_id、stream_id、连续游标、selective ranges、missing ranges |
| `UPDATE_POLICY` | 更新有签名的远程策略 | policy blob |

## 9. 连接与重同步状态机

### 9.1 连接状态

```text
DISCONNECTED
  -> BT_CONNECTING
  -> BT_READY
  -> SYNCING
  -> CONTROL_READY
  -> P2P_CONNECTING
  -> DATA_READY
```

允许的退化：

- `DATA_READY -> CONTROL_READY`：已协商的数据通道断开，但蓝牙控制仍可用。
- `CONTROL_READY -> DISCONNECTED`：手机失联，眼镜按本地租约和硬门运行。
- 任意状态收到本地隐私操作：立即进入 A0/L0，不等待手机。

### 9.2 重同步顺序

重新连接后必须依次执行：

1. 双端交换协议、App、SDK、固件和 `controller_id`。
2. 眼镜返回 `StateSnapshotV1`。
3. 双端比较 `control_epoch`，较大的隐私 epoch 优先。
4. 旧租约一律不自动恢复；手机根据当前 A0/A1/A2 重新签发。
5. 手机携带视觉 ACK scope/ranges 和每个音频 stream 的 `AudioRangeAckV1` 请求缺失数据，不使用会跨空洞的单一 source_seq。
6. 完成实时音频通道协商后启动音频数据面；P2P 成功后再启动大文件传输。

### 9.3 状态快照

```json
{
  "schema": "memory.glass-state-snapshot.v1",
  "device_id": "0191...",
  "boot_id": "0191...",
  "control_epoch": 42,
  "last_command_seq": 1071,
  "state_revision": 388,
  "audio": {
    "capture_state": "STREAMING",
    "source_kind": "GLASSES",
    "stream_id": "0191...",
    "next_source_seq": 90120,
    "capture_generation": 7,
    "active_conversation_id": "0191...",
    "active_binding_id": "0191...",
    "lease_id": "0191...",
    "lease_remaining_ms": 118000
  },
  "visual": {
    "effective_level": "L1_SCOUT",
    "lease_id": "0191...",
    "lease_remaining_ms": 220000,
    "burst_remaining_ms": 0
  },
  "durability": {
    "pending_capture_count": 14,
    "spool_first_seq": 89880,
    "spool_last_seq": 90119,
    "storage_used_bytes": 144020480
  },
  "health": {
    "battery_percent": 67,
    "charging": false,
    "thermal_state": "NORMAL",
    "telemetry_source": "ANDROID_OS"
  },
  "privacy": {
    "locked": false,
    "privacy_lock_id": null,
    "last_generated_source_seq": 90119
  }
}
```

## 10. 音频状态契约

### 10.1 手机业务状态

```text
A0_PRIVATE
A1_LISTENING
A2_ACTIVE
A2_END_PENDING
A2_FINALIZING
```

| 当前状态 | 事件 | 下一状态 | 必须动作 |
|---|---|---|---|
| A0 | 用户明确恢复且隐私允许 | A1 | 校验 privacy_lock_id，执行 RESUME_FROM_PRIVATE，必须新建 stream 并签发音频租约 |
| A1 | 持续人声/日程/手动开始 | A2_ACTIVE | 创建 conversation、冻结预录、事务提交 binding OPEN |
| A1 | 隐私禁止 | A0 | 清空未绑定预录和 spool，停止采集 |
| A2_ACTIVE | 静音+上下文候选 | A2_END_PENDING | 继续录音，不关闭 chunk |
| A2_END_PENDING | 人声恢复 | A2_ACTIVE | 取消结束候选 |
| A2_ACTIVE/END_PENDING | 正常/手动结束 | A2_FINALIZING | 关闭最终分片、校验、提交 binding CLOSE 和 final manifest |
| A2_ACTIVE/END_PENDING | 隐私禁止 | A0 | 立即本地 seal，进入 BOUNDARY_PENDING，停止采集后异步关闭 binding/finalize |
| A2_FINALIZING | 本地 seal 完成 | A1 | 新建下一次预录上下文；云端定稿异步进行 |

### 10.2 眼镜音频执行状态

```text
STOPPED
STARTING
STREAMING
DISCONNECTED_SPOOLING
DRAINING
ERROR_STOPPED
```

眼镜不自行判断 A1/A2；它只执行音频租约并维护 `stream_id/source_seq`。A1/A2 的正式区别由手机的会话绑定和持久化策略决定。

### 10.3 音频租约

v1 默认：

```json
{
  "lease_id": "0191...",
  "lease_type": "AUDIO_CAPTURE",
  "ttl_ms": 300000,
  "renew_before_ms": 120000,
  "max_disconnected_spool_ms": 120000
}
```

- 音频租约过期后眼镜必须停止麦克风并写 `AUDIO_LEASE_EXPIRED`。
- 眼镜在音频租约有效时持续维护短时滚动 spool；实时数据通道断开后进入 `DISCONNECTED_SPOOLING`，但不得到断链后才开始保存补偿数据。
- spool 达到时间或存储上限后，丢弃最旧未绑定 A1 帧；A2 帧不得静默丢弃，必须生成 gap 事件并停止或按策略降级。
- 若能力协商启用 `phone_mic_fallback=true`，手机可创建独立 PHONE stream 和 timeline segment；M3 默认可以关闭该能力，关闭时必须写显式 missing range。

## 11. 实时音频帧契约

逻辑消息使用 Protobuf 3；在线路上采用 4 字节大端长度前缀 + protobuf payload + CRC32C。M0 可以临时使用 JSON 调试控制消息，但音频帧不得使用 JSON。

```protobuf
enum AudioSourceKind {
  AUDIO_SOURCE_KIND_UNSPECIFIED = 0;
  GLASSES = 1;
  PHONE = 2;
}

message AudioFrameV1 {
  string schema = 1;                 // memory.audio-frame.v1
  string device_id = 2;
  string boot_id = 3;
  string stream_id = 4;
  optional string conversation_id = 5;
  uint64 source_seq = 6;
  uint64 source_monotonic_us = 7;
  uint32 codec_config_id = 8;
  uint32 sample_count = 9;
  uint32 duration_us = 10;
  bytes payload = 11;
  uint32 payload_crc32c = 12;
  AudioSourceKind source_kind = 13;  // GLASSES / PHONE
  uint64 capture_generation = 14;    // 启动采集时的本地 generation
}
```

`CodecConfigV1`：

```json
{
  "schema": "memory.codec-config.v1",
  "codec_config_id": 3,
  "codec": "OPUS",
  "sample_rate_hz": 16000,
  "channels": 1,
  "sample_format": "S16LE",
  "frame_duration_ms": 20,
  "target_bitrate_bps": 20000
}
```

规则：

1. 首选眼镜端把 Rokid `AudioCallback` 输出转换为 Opus 16kHz 单声道 20ms 帧。
2. 若 M0 证明眼镜端编码不可行，可临时使用 PCM，但必须保持同一 `AudioFrameV1` 语义并重新评估功耗/带宽。
3. codec 变化前必须先发送新 `CodecConfigV1`，并使用新 `codec_config_id`。
4. 手机发现 source_seq 缺口时立即持久化 `PENDING_GAP`，不得只在 finalize 时发现。
5. 重复帧以 `boot_id + stream_id + source_seq + payload_crc32c` 去重。
6. 同一帧身份出现不同 CRC 时进入 `CONFLICTING_FRAME` 隔离，不允许最后写入覆盖前一个版本。
7. `conversation_id` 在 A1 必须为空，在 A2 也只作冗余；真实归属由 binding OPEN/CLOSE 决定。
8. `capture_generation` 在每次启动、恢复或隐私 fence 后递增。旧 generation 的迟到 SDK 回调不得构造成可持久化 AudioFrame。
9. M3 只要求发送 `source_kind=GLASSES`；解析器必须认识 PHONE 枚举，但能力未启用时应返回 `AUDIO_SOURCE_UNSUPPORTED`，不得把 PHONE 帧误当 GLASSES 保存。

## 12. 预录、spool 与音频修复

### 12.1 两类缓冲不可混淆

| 缓冲 | 所在端 | 目的 | 默认窗口 |
|---|---|---|---:|
| `PreRollAudioBuffer` | 手机 | A1→A2 时补齐首声前 2 秒 | 90 秒 |
| `LocalAudioSpool` | 眼镜 | 实时链路断开后的传输补偿 | 120 秒 |

手机预录是会话选择缓冲；眼镜 spool 是链路可靠性缓冲。二者可以包含同一 source_seq，但不能产生两个正式音频身份。

### 12.2 spool block

眼镜在音频租约有效期间持续将帧写入短时加密滚动 spool，并按 5 秒左右生成不可变 block。实时发送和 spool 共用同一个 AudioFrame 身份；不能在断链后重新编号或重新编码为另一个正式帧：

```json
{
  "schema": "memory.audio-spool-block.v1",
  "spool_block_id": "0191...",
  "device_id": "0191...",
  "boot_id": "0191...",
  "stream_id": "0191...",
  "first_source_seq": 89880,
  "last_source_seq": 90119,
  "start_source_monotonic_us": 921000000,
  "end_source_monotonic_us": 926000000,
  "codec_config_ids": [3],
  "sha256": "...",
  "state": "SPOOLED"
}
```

手机只有在实时/补偿帧、manifest 和 gap event 的事务提交后，才发送 `ACK_AUDIO_RANGES`：

```json
{
  "schema": "memory.audio-range-ack.v1",
  "boot_id": "0191...",
  "stream_id": "0191...",
  "contiguous_through_source_seq": 90119,
  "selective_ranges": [
    {"first_source_seq": 90200, "last_source_seq": 90300}
  ],
  "missing_ranges": [
    {"first_source_seq": 90120, "last_source_seq": 90199}
  ],
  "committed_phone_monotonic_ms": 18594000
}
```

眼镜先持久化 ACK range tombstone，再删除被连续游标或 selective range 完整覆盖的 block。`missing_ranges` 只是手机当前缺口声明，不是删除许可。不得使用一个会跨过空洞的标量 ACK 删除未确认帧。

M3 存储压力处理顺序固定为：停止 L3 → 停止 L1 → 触发紧急 P2P → 淘汰最旧未绑定 A1 block。已绑定 A2 block 达到硬上限时不得静默淘汰；必须写 `AUDIO_STORAGE_HARD_LIMIT`、停止新音频并让手机以显式 gap/结束原因封存会话。

### 12.3 不可变修复模型

已经 `SEALED` 或上传的 60 秒基础 chunk 不得重写。迟到 spool 帧写成独立 `AudioRepairSegmentV1`：

```json
{
  "schema": "memory.audio-repair-segment.v1",
  "repair_id": "0191...",
  "conversation_id": "0191...",
  "stream_id": "0191...",
  "first_source_seq": 89880,
  "last_source_seq": 89940,
  "target_chunk_ids": ["0191..."],
  "sha256": "...",
  "reason": "LATE_GLASS_SPOOL",
  "assembly_version": 2
}
```

云端按 `基础 chunk + repair segments - confirmed missing ranges` 组装会话版本。迟到修复只能生成新版本，不能静默修改旧 ASR 结果。

## 13. 手机音频分片状态

```text
OPEN
  -> SEALED
  -> PHONE_COMMITTED
  -> UPLOAD_PENDING
  -> UPLOADING
  -> CLOUD_ACKED
  -> RETAINED
  -> DELETED
```

规则：

- `SEALED` 后音频 bytes、起止 source_seq、样本数和 SHA 不可变。
- `PHONE_COMMITTED` 表示 chunk 文件、manifest 和 timeline event 在同一数据库事务中可恢复。
- 上传失败进入 `UPLOAD_PENDING`，不得回到 `OPEN`。
- `CLOUD_ACKED` 只表示云端接入层已持久化，不表示 ASR 已完成。
- 删除必须同时满足云端 ACK、保留期和无进行中的修复/重处理引用。

`AudioChunkManifestV1` 最小字段：

```json
{
  "schema": "memory.audio-chunk.v1",
  "chunk_id": "0191...",
  "conversation_id": "0191...",
  "sequence_no": 7,
  "type": "NORMAL",
  "source": "GLASSES",
  "boot_id": "0191...",
  "stream_id": "0191...",
  "first_source_seq": 120010,
  "last_source_seq": 123009,
  "start_source_monotonic_us": 10000000,
  "end_source_monotonic_us": 70000000,
  "clock_map_id": "0191...",
  "codec_config_ids": [3],
  "sample_count": 960000,
  "sha256": "...",
  "durability_state": "PHONE_COMMITTED"
}
```

## 14. 视觉状态契约

### 14.1 状态

```text
L0_OFF
L1_SCOUT
L3_BURST
```

不变量：

1. A0 必须对应 L0。
2. L3 必须满足 A2 active、有效 L1 基础租约、有效 L3 burst 租约和策略门批准。
3. L3 只能从 L1 进入。
4. A2 结束后不得保留 L1/L3 租约。
5. 眼镜有效等级不得高于手机授权等级。
6. 租约过期、本地隐私、低电、过热或存储硬上限只能降级，不能升级。

### 14.2 L1 租约

v1 默认：

```json
{
  "lease_id": "0191...",
  "lease_type": "VISUAL_L1",
  "conversation_id": "0191...",
  "interval_ms": 60000,
  "first_capture_delay_ms": 5000,
  "ttl_ms": 300000,
  "resolution_profile": "L1_LOW_MID",
  "max_pending_captures": 500
}
```

首张 L1 默认在 A2 开始后 5 秒拍摄，避免不足 60 秒的短会话完全没有视觉锚点。该参数可远程调节，但不得超过一个完整 L1 周期。

### 14.3 L3 burst 租约

```json
{
  "lease_id": "0191...",
  "lease_type": "VISUAL_L3",
  "conversation_id": "0191...",
  "decision_id": "0191...",
  "reason_code": "PAGE_OR_SCREEN_REFERENCE",
  "fps": 1,
  "duration_ms": 20000,
  "lease_ttl_ms": 25000,
  "fallback_level": "L1_SCOUT",
  "budget_token": "signed-token..."
}
```

眼镜收到命令时用本地单调时钟计算：

```text
burst_deadline = now + min(duration_ms, lease_ttl_ms, local_safe_max_ms)
```

续租只能维持控制有效性，不得延长 `burst_deadline`。再次升入 L3 必须使用新的 `decision_id + lease_id + budget_token`。

### 14.4 拍照 attempt

每个 L1/L3 tick 都必须先持久化或可恢复地产生 `capture_attempt_id`。结果枚举：

- `CAPTURED`
- `SKIPPED_PRIVACY`
- `SKIPPED_LEASE_EXPIRED`
- `SKIPPED_BATTERY`
- `SKIPPED_THERMAL`
- `SKIPPED_STORAGE`
- `CAMERA_BUSY`
- `CAMERA_TIMEOUT`
- `CAMERA_ERROR`
- `COMMIT_FAILED`

这使云端能够区分“策略上没有拍”与“应该拍但证据丢失”。图片 `sequence_no` 只对成功 CAPTURED 的图片递增；attempt 使用独立连续序号。

M3 冻结序号作用域：`attempt_no` 和成功图片 `sequence_no` 都在 `device_id + boot_id` 内从 0 单调递增并持久化；重启生成新 boot_id 后可以归零。ACK 游标必须连同 boot_id 解释，不得用一个设备级标量跨 boot 删除文件。`capture_id`/`capture_attempt_id` 仍是全局幂等身份，conversation_id 不参与序号唯一性。

## 15. 图片崩溃安全提交

文件和 SQLite 无法实现真正的跨介质原子事务，因此 v1 使用 journaled bundle commit，不再把 JSONL append 当作唯一事实源。

### 15.1 提交流程

1. 写 `capture_id.tmp`，完成后 `fsync`。
2. 计算 SHA-256 和文件长度。
3. SQLite 事务写入 `CAPTURE_STAGED`、attempt、租约和时间信息。
4. 原子 rename 为最终文件名。
5. SQLite 事务更新为 `CAPTURED`。
6. JSONL 只作为导出/调试视图，由数据库重建，不参与正确性判断。

### 15.2 崩溃恢复

| 恢复现场 | 动作 |
|---|---|
| 孤立 `.tmp`，无数据库行 | 删除临时文件并记清理指标 |
| `CAPTURE_STAGED` + `.tmp` | 校验后继续 rename/commit |
| `CAPTURE_STAGED` + 最终文件 | 校验 SHA 后标记 CAPTURED |
| `CAPTURED` 但文件缺失 | 标记 `LOCAL_CORRUPT`，不得上传空证据 |
| 最终文件存在但无 manifest | 隔离为 orphan，人工/后台恢复，不生成新 capture_id |

### 15.3 VisualCaptureManifestV1

```json
{
  "schema": "memory.visual-capture.v1",
  "capture_id": "0191...",
  "capture_attempt_id": "0191...",
  "device_id": "0191...",
  "boot_id": "0191...",
  "conversation_id": "0191...",
  "visual_level": "L3_BURST",
  "sequence_no": 17,
  "attempt_no": 19,
  "source_monotonic_us": 18532188000,
  "clock_map_id": "0191...",
  "captured_at_utc": "2026-08-24T09:16:12.188+08:00",
  "decision_id": "0191...",
  "lease_id": "0191...",
  "trigger_reason": "PAGE_OR_SCREEN_REFERENCE",
  "resolution": {"width": 1280, "height": 720},
  "quality_tags": ["EXPOSURE_OK"],
  "file_size_bytes": 284102,
  "sha256": "...",
  "durability_state": "CAPTURED"
}
```

## 16. 隐私抢占契约

### 16.1 ENTER_PRIVATE fence

`ENTER_PRIVATE` payload：

```json
{
  "reason_code": "USER_PAUSE",
  "privacy_lock_id": "0191...",
  "privacy_fence": {
    "phone_monotonic_ms": 18533000,
    "best_known_glass_monotonic_us": 18532992000,
    "clock_map_id": "0191..."
  },
  "retain_pre_fence_committed_evidence": true
}
```

眼镜收到后必须按以下顺序执行：

1. 持久化新 `control_epoch`、`privacy_lock_id`、递增后的 `capture_generation` 和 `PRIVACY_ENTERED` 事件。
2. 使全部音频/视觉租约失效；在此之后任何旧 generation 的 SDK 回调都只能产生审计/gap 事件。
3. 停止接受音频回调进入 spool/实时发送，再调用 SDK 停止音频。
4. 停止视觉 timer，调用相机关闭。
5. 删除未绑定 A1 预录/spool 原始音频。
6. 删除尚未提交为 CAPTURED 的 staged 图片。
7. 对 fence 前已经 CAPTURED 的媒体保留原状态，由手机按既有会话策略处理。
8. 返回 ACK 和最终状态快照，其中必须包含 `privacy_lock_id`、`last_generated_source_seq`、`last_committed_capture_attempt_id` 和实际停止时间。

禁止规则：

- `privacy_fence` 之后产生的原始音频不得写盘、传输或进入预录。
- fence 持久化后才到达的旧 generation 音频/拍照回调一律 fail-closed 丢弃，即使 SDK 声称其采样早于 fence；允许形成显式 gap，不允许形成隐私泄漏。
- fence 时仍为 staged 的图片不得继续提交。
- 恢复后不得补录 A0 区间。
- 恢复只进入 A1/L0；不得恢复旧 A2 或旧 L3。
- 普通 `START_LISTENING` 不得解除 privacy lock；只接受引用当前 `privacy_lock_id` 且具有明确 `user_action_id` 的 `RESUME_FROM_PRIVATE`。

### 16.2 手机本地 seal 与最终边界

隐私停止不能等待网络，但手机在下发 ENTER_PRIVATE 时可能尚不知道眼镜最后生成的 source_seq。M3 将“停止向当前会话写入”和“完整性边界已确认”拆开：

```text
LOCAL_SEALED
  -> BOUNDARY_PENDING
  -> FINAL_MANIFEST_COMMITTED
```

1. 用户暂停时，手机先进入 `LOCAL_SEALED`，关闭当前 OPEN chunk 并停止向该 conversation 追加新实时帧。
2. 手机立即下发 ENTER_PRIVATE，不等待 ACK 即完成 UI 隐私反馈和本地禁采集。
3. 眼镜 ACK 返回 `last_generated_source_seq` 后，手机写 binding CLOSE，`end_boundary_quality=EXACT`，再提交 final manifest。
4. 若眼镜失联直到边界等待超时，手机以最后已知序号关闭 binding，设置 `end_boundary_quality=UNKNOWN`、`missing_tail=true`，云端只能输出带缺口版本。
5. BOUNDARY_PENDING 不阻塞下一次用户显式恢复，但恢复后的流必须使用新 stream_id，不能跨越隐私区间续用旧 binding。

### 16.3 本地隐私操作

若眼镜端按键/语音提供本地暂停，它必须先生成新的 `privacy_lock_id`、递增 capture generation、执行本地 A0/L0，再向手机发送 `LOCAL_PRIVACY_ENTERED`。事件必须包含 `boot_id`、本地 fence 单调时间、`last_generated_source_seq`、当前 binding/conversation 和 privacy_lock_id。手机不得因为自身状态仍是 A1/A2 而自动反向恢复；只有用户明确恢复后才能引用该 privacy_lock 签发新 epoch 和新租约。

## 17. 视觉文件传输与双 ACK

### 17.1 状态

```text
CAPTURE_STAGED
  -> CAPTURED
  -> TRANSFERRING
  -> PHONE_COMMITTED
  -> GLASS_ACKED
  -> CLOUD_ACKED
  -> RETAINED/DELETED
```

其中 `PHONE_COMMITTED` 和 `CLOUD_ACKED` 分别由手机、云端拥有；眼镜保存的是相应 ACK tombstone。

### 17.2 ACK1 流程

1. 手机通过 manifest 获取 `capture_id + sha256 + size`。
2. 眼镜用 Rokid P2P 文件接口发送文件。
3. 手机 SDK 的 `onComplete(filePath)` 只进入 `TRANSFER_RECEIVED`。
4. 手机重新计算 SHA-256、检查长度、写入 staging。
5. 手机数据库事务提交文件定位和 `PHONE_COMMITTED`。
6. 手机发送 `ACK_PHONE_COMMITTED(capture_id, sha256)`。
7. 眼镜持久化 ACK tombstone 后删除本地文件。

ACK1 丢失时，手机对相同 `capture_id + sha256` 返回 `ALREADY_COMMITTED`。相同 `capture_id` 携带不同 SHA 时必须返回 `CONFLICTING_HASH` 并隔离双方副本。

### 17.3 ACK2 流程

1. 手机使用 `capture_id + sha256` 幂等上传。
2. 云端验证 bytes、size、schema、所有权和 SHA 后持久化。
3. 云端返回 `ingestion_id`，即 ACK2。
4. 手机标记 `CLOUD_ACKED`。
5. 手机只按保留策略清理本地副本；不得因 ACK1 已完成而提前删除。

### 17.4 ACK 游标

不得只使用一个可能跳过空洞的标量游标。v1 使用：

- `device_id + boot_id`：游标作用域。
- `committed_through_sequence_no`：此前全部连续提交。
- `selective_acks`：游标之后已提交的离散 capture ID。
- `missing_sequence_ranges`：显式空洞。

眼镜只删除被连续游标或 selective ACK 明确覆盖且 SHA 匹配的文件。

## 18. 会话最终 manifest 与云端完整性

`FinalConversationManifestV1`：

```json
{
  "schema": "memory.final-conversation.v1",
  "conversation_id": "0191...",
  "end_reason": "AUTO_SILENCE",
  "end_boundary_quality": "EXACT",
  "missing_tail": false,
  "local_manifest_version": 1,
  "binding_ids": ["0191..."],
  "base_chunks": [
    {"sequence_no": 0, "chunk_id": "0191...", "sha256": "..."}
  ],
  "repair_segments": [],
  "expected_stream_ranges": [
    {
      "source_kind": "GLASSES",
      "boot_id": "0191...",
      "stream_id": "0191...",
      "first_source_seq": 88012,
      "last_source_seq": 160210
    }
  ],
  "known_missing_ranges": [],
  "source_history": [
    {
      "source_kind": "GLASSES",
      "boot_id": "0191...",
      "stream_id": "0191...",
      "start_timeline_ms": 0,
      "end_timeline_ms": 602000,
      "alignment_method": "CLOCK_MAP",
      "time_quality": "GOOD"
    }
  ],
  "sealed_phone_monotonic_ms": 19134000
}
```

云端以 `expected_stream_ranges` 为主完整性基准，同时校验基础 chunk、repair、每个 source stream 的 source_seq 覆盖和 binding OPEN/CLOSE。`end_boundary_quality=UNKNOWN` 或 `missing_tail=true` 时不得输出无缺口 FINAL。

云端状态：

```text
INGESTING
  -> WAITING_COMPLETENESS
  -> PROCESSING
  -> FINAL
  -> FINAL_WITH_GAPS
  -> REPROCESSING
  -> REVISED
```

规则：

- 收齐后输出 `FINAL(version=1)`。
- 截止时间到仍缺失时输出 `FINAL_WITH_GAPS(version=1)`，并保存 missing ranges。
- 迟到 chunk/repair 不能覆盖 version 1；必须产生 version 2，并记录 `supersedes=1` 与原因。
- 最终 ASR 不得在缺失区间生成看似有证据的文本；缺口在 transcript 中必须可见。

## 19. OCR 与 ASR 协调

图片可能早于最终音频到云端，因此视觉管线使用以下状态：

```text
VISUAL_INGESTED
  -> OCR_RAW_READY
  -> WAITING_AUDIO_CONTEXT
  -> EVIDENCE_READY
  -> EVIDENCE_REVISED
```

- OCR 可以在音频未定稿时先运行，但不得把依赖会话语义的结果标记为最终证据。
- ASR final 或 final_with_gaps 到达后，按 `conversation_id + clock_map_id` 对齐图片。
- ASR 或 OCR 任一侧出现新版本时，生成新的 `evidence_version`。
- 索引记录必须包含 `capture_id`、图片坐标、音频时间范围、ASR version、OCR version 和证据质量。
- 低质量或时间不确定的证据必须显式标记，不得被索引层隐藏。

## 20. 云端 API 与幂等契约

推荐最小接口：

| 方法 | 路径 | 幂等键 |
|---|---|---|
| `PUT` | `/v1/audio-chunks/{chunk_id}` | `chunk_id + sha256` |
| `PUT` | `/v1/audio-repairs/{repair_id}` | `repair_id + sha256` |
| `PUT` | `/v1/visual-captures/{capture_id}` | `capture_id + sha256` |
| `PUT` | `/v1/conversations/{id}/final-manifest/{version}` | conversation + local version + SHA |
| `POST` | `/v1/conversations/{id}:request-finalize` | request_id |
| `GET` | `/v1/conversations/{id}/status` | 无 |

统一响应语义：

- `200/201`：已持久化。
- `200 already_committed=true`：相同 ID 和 SHA 已存在。
- `409 CONFLICTING_HASH`：相同 ID 但内容不同。
- `422 INVALID_SCHEMA`：协议字段或状态不合法。
- `429/503 + retry_after`：可重试。
- `401/403`：不得盲目重试，必须刷新鉴权或停止上传。

所有云端对象使用：

```text
tenant_id + device_id + object_id + sha256
```

判定幂等。服务端不得以文件名作为唯一身份。

## 21. 策略配置契约

`CapturePolicyV1` 必须签名、版本化并具有生效时间：

```json
{
  "schema": "memory.capture-policy.v1",
  "policy_revision": 18,
  "valid_from_utc": "2026-08-24T00:00:00Z",
  "audio": {
    "preroll_sec": 90,
    "audio_lease_ttl_sec": 300,
    "spool_max_sec": 120,
    "chunk_duration_sec": 60
  },
  "visual": {
    "l1_interval_sec": 60,
    "l1_first_capture_delay_sec": 5,
    "l1_lease_ttl_sec": 300,
    "l3_default_sec": 20,
    "l3_max_sec": 30,
    "l3_budget_sec_per_15min": 180,
    "l3_min_confidence": 0.75
  },
  "safety": {
    "disable_l3_battery_below_percent": 20,
    "disable_auto_visual_below_percent": 10,
    "storage_soft_limit_percent": 80,
    "storage_hard_limit_percent": 95
  },
  "signature": "base64..."
}
```

编译期安全上限优先于远程配置。远程配置可以收紧采集，不得突破：

- L3 单次最大 30 秒。
- 允许隐私关闭后自动恢复，或允许普通 START_LISTENING 绕过 privacy lock。
- 绕过本地温控或存储硬上限。
- CLOUD_ACKED 前删除手机唯一副本。

## 22. 安全、加密和保留

### 22.1 配对与控制消息

- 手机与眼镜首次配对必须由用户在近场完成。
- 双端生成设备绑定密钥并存入 Android Keystore；控制信封使用会话 nonce 和签名防重放。
- `controller_id` 变化时进入 `UNTRUSTED_CONTROLLER`，不得直接继承旧租约。
- 开发构建可关闭业务层加密，但必须显示明显的 DEV 标识，且不得采集真实用户数据。

### 22.2 媒体加密

- 手机媒体库必须应用层加密或使用等价的受保护存储。
- 眼镜长期缓存和 P2P 发送目录中的媒体应该使用每文件 AES-GCM 密钥；明文相机临时文件应尽快清理。
- 如果 Rokid 拍照/文件接口无法直接处理加密或私有目录，M0 必须记录具体限制并采用“短时明文 staging → 加密 blob → 删除明文”的适配流程。
- 日志不得记录原始转写、图片内容、精确位置、密钥、文件明文路径或用户对话。

### 22.3 默认保留

| 数据 | 默认保留 |
|---|---|
| 未绑定 A1 预录/眼镜 spool | 最多 120 秒；A0 时立即清除 |
| 普通原始音频 | 30 天 |
| 普通图片 | 30 天 |
| 标记重要会话原始音频 | 90 天 |
| ASR/OCR 文本 | 长期，用户可删除 |
| 协议/策略审计事件 | 180 天，不含原始内容 |

用户删除会话时必须级联删除原始媒体、repair、ASR/OCR、索引和派生摘要，并留下不含内容的删除审计记录。

## 23. 错误码

| 错误码 | 是否重试 | 处理 |
|---|---|---|
| `STALE_EPOCH` | 否 | 请求状态快照，重新同步 |
| `STALE_COMMAND` | 否 | 视为旧消息，不执行 |
| `ALREADY_APPLIED` | 否 | 使用原 ACK |
| `CLOCK_UNCERTAIN` | 条件重试 | 只允许 fail-closed 隐私命令；刷新 ClockMap 后重发普通命令 |
| `INVALID_STATE` | 条件重试 | 同步状态后由 reducer 决定 |
| `LEASE_EXPIRED` | 否 | 降级，重新签发新 lease |
| `PRIVACY_FENCED` | 否 | 保持 A0/L0，必须用户恢复 |
| `P2P_UNAVAILABLE` | 是 | 退回控制面并指数退避 |
| `AUDIO_FORMAT_UNSUPPORTED` | 否 | 切换已协商 codec 或停止 |
| `AUDIO_SOURCE_UNSUPPORTED` | 否 | M3 未启用 PHONE 等来源时拒绝该帧并保持显式能力状态 |
| `STORAGE_SOFT_LIMIT` | 条件重试 | 触发批传并降级视觉 |
| `STORAGE_HARD_LIMIT` | 否 | 停止新视觉，保护已提交证据 |
| `CAMERA_BUSY` | 是，有限次 | 当前 tick 记失败，不补发无限重试 |
| `CAMERA_TIMEOUT` | 是，有限次 | 关闭相机并重建媒体服务 |
| `HASH_MISMATCH` | 是，最多 3 次 | 删除接收 staging 后重传 |
| `CONFLICTING_HASH` | 否 | 隔离并报警，不覆盖 |
| `INVALID_SCHEMA` | 否 | 协议不兼容，停止该对象 |
| `AUTH_FAILED` | 否 | 停止传输并刷新配对/鉴权 |

## 24. 并发模型与持久化原则

### 24.1 单写者 reducer

手机和眼镜各自必须有一个串行状态 reducer。SDK 回调、用户操作、计时器和网络事件先转换为事件，再由 reducer 修改状态；不得由多个回调直接写状态变量。

事件处理顺序：

1. 持久化输入事件或幂等键。
2. reducer 计算新状态和副作用。
3. 持久化状态 revision。
4. 执行 SDK/网络副作用。
5. 持久化副作用结果。

副作用失败必须产生新事件，不能回滚已经持久化的历史。

音频和相机 SDK 副作用启动时必须捕获当时的 `capture_generation + control_epoch + lease_id`。回调进入 reducer 后先校验 generation、隐私锁和租约，再决定是否允许进入 staging；校验失败的迟到回调只记录无内容审计事件，不得直接写媒体文件。

### 24.2 状态 revision

每次状态变化增加 `state_revision`。心跳和 ACK 携带 revision，便于判断状态快照是否过旧。revision 只在单个设备/控制器内有序，不作为跨设备时间戳。

## 25. 故障恢复矩阵

| 故障点 | 恢复结果 |
|---|---|
| 眼镜在应用 L3 命令前崩溃 | 命令重发；若已过 TTL 则拒绝，不启动 burst |
| 眼镜在 A0 持久化后、关闭相机前崩溃 | 重启读取 A0 epoch，先保持麦克风/相机关闭 |
| 图片写完但 manifest 未提交 | orphan/staged 恢复，不生成新 capture_id |
| 手机收到文件后、事务前崩溃 | 无 ACK1，眼镜重传 |
| 手机事务后、ACK1 前崩溃 | 返回 `ALREADY_COMMITTED`，再发 ACK1 |
| 眼镜 ACK tombstone 后、删文件前崩溃 | 重启继续清理；重复 ACK 无副作用 |
| 眼镜删文件后云端失败 | 手机副本重试，不回滚 ACK1 |
| 实时数据通道断开 | 蓝牙控制保留；持续 spool 补传；M3 默认写显式 gap，能力启用时才切独立 PHONE stream |
| 手机完全失联 | 租约内有限缓存；租约到期自动停止/降级 |
| 迟到 spool 修复已上传 chunk | 新建 repair segment 和 assembly version |
| 云端 final_with_gaps 后收到修复 | 生成 ASR/evidence 新版本，不覆盖旧版本 |
| OTA/重启导致 boot_id 变化 | 旧 stream 封存或标 gap；新建 stream，禁止跨 boot 续序号 |
| 隐私 fence 后收到旧音频/拍照回调 | generation 校验失败，丢弃内容并记录审计/gap，不提交媒体 |
| 隐私停止时眼镜未返回最后序号 | 手机保持 BOUNDARY_PENDING；超时后以 UNKNOWN 边界和 missing_tail 定稿 |

## 26. 可观测性

必须采集不含原始内容的指标：

- 控制命令延迟、重复、乱序、过期和拒绝数。
- A0 指令到音频停止、相机关闭的时延。
- source_seq 丢失率、spool 恢复率、repair 数和 remaining gaps。
- L1/L3 attempt、成功、跳过和失败原因。
- P2P 建连耗时、成功率、吞吐和重连次数。
- CAPTURED→PHONE_COMMITTED→CLOUD_ACKED 各阶段时延。
- 电量下降、温度档位、相机激活时长、发送字节数。
- 云端 final、final_with_gaps 和 revised 比例。

日志关联字段只使用：`device_id`、`boot_id`、`stream_id`、`conversation_id`、`command_id`、`object_id` 和状态码。

## 27. Schema 演进

- `schema` 使用 `memory.<name>.v<major>`。
- 同一 major 只能添加可选字段或新增枚举值；读取方必须忽略未知可选字段。
- 删除字段、改变字段语义或单位必须升级 major。
- 所有单位写入字段名，例如 `_ms`、`_us`、`_bytes`、`_percent`。
- 写入端默认只写当前 major；读取端至少兼容当前 major 和前一个 major。
- `HELLO` 协商不到共同 major 时进入 `PROTOCOL_INCOMPATIBLE`，不得开始采集。

## 28. 开发接口基线

### 28.1 眼镜适配层

```kotlin
interface RokidGlassDeviceAdapter {
    val capabilities: StateFlow<RokidCapabilities>
    val health: StateFlow<GlassHealth>
    val linkState: StateFlow<LinkState>

    suspend fun startAudio(config: CodecConfigV1): Result<Unit>
    suspend fun stopAudio(): Result<Unit>
    fun audioFrames(): Flow<RawRokidAudioBuffer>

    suspend fun takePhoto(request: RokidPhotoRequest): Result<RokidPhotoFile>
    suspend fun closeCamera(): Result<Unit>

    suspend fun sendControl(bytes: ByteArray, channel: ControlChannel): Result<Unit>
    suspend fun sendStream(tag: String, bytes: ByteArray): Result<Unit>
    suspend fun sendFile(path: String, remoteDir: String): Flow<FileTransferEvent>

    suspend fun connectP2p(): Result<Unit>
    suspend fun disconnectP2p(): Result<Unit>
}
```

### 28.2 眼镜协议执行器

```kotlin
interface GlassProtocolEngine {
    suspend fun accept(command: ControlCommandV1): ControlAckV1
    fun stateSnapshots(): Flow<GlassStateSnapshotV1>
    suspend fun recoverFromDisk()
}

interface GlassAudioLeaseEngine {
    suspend fun install(lease: AudioLeaseV1)
    suspend fun revoke(reason: ReasonCode)
    fun frames(): Flow<AudioFrameV1>
}

interface GlassVisualLeaseEngine {
    suspend fun installL1(lease: VisualL1LeaseV1)
    suspend fun startL3(lease: VisualL3LeaseV1)
    suspend fun revoke(reason: ReasonCode)
}
```

### 28.3 手机状态内核

```kotlin
interface CaptureStateStore {
    suspend fun append(event: CaptureEventV1): Long
    suspend fun reduceThrough(eventOffset: Long): CaptureSnapshotV1
    suspend fun loadSnapshot(): CaptureSnapshotV1
}

interface ConversationBindingStore {
    suspend fun append(event: ConversationBindingEventV1)
    suspend fun activeFor(bootId: String, streamId: String): ConversationBindingProjection?
    suspend fun recoverOpenBindings()
}

interface AudioRangeCommitStore {
    suspend fun commitFramesAndGaps(
        frames: List<AudioFrameV1>,
        gaps: List<SourceSeqRange>
    ): AudioRangeAckV1

    suspend fun appendRepair(segment: AudioRepairSegmentV1)
}

interface GlassControlClient {
    suspend fun send(command: ControlCommandV1): ControlAckV1
    suspend fun requestState(): GlassStateSnapshotV1
}
```

业务模块不得直接调用 Rokid SDK；所有 SDK 调用必须经过 adapter 和协议执行器。

## 29. 契约测试与验收

### 29.1 控制面

1. 相同 `command_id` 发送 10 次，只执行一次副作用并返回一致 ACK。
2. 已执行命令之后再执行更大 command_seq，随后重发旧 command_id；仍返回原 ACK，不返回 STALE_COMMAND。
3. 先发送 L3，再发送更高 epoch 的 A0，随后重放旧 L3；相机保持关闭并返回 `STALE_EPOCH`。
4. L1/L3 租约过期且手机失联，眼镜在规定时间内自动 L0。
5. 眼镜进程在 A0 各持久化点被强杀，重启后均不能恢复采集。
6. ClockMap 缺失或 uncertainty 超限时普通启动命令返回 CLOCK_UNCERTAIN；ENTER_PRIVATE 仍 fail-closed 生效。

### 29.2 音频

1. A1 帧 `conversation_id=null`，进入 A2 后通过 binding OPEN 正确固化前 2 秒，并在正常/隐私结束时产生唯一 CLOSE。
2. 人为丢弃 1%、5%、20% 实时帧，source_seq 缺口可检测。
3. 实时音频通道中断前随机丢弃尾部帧，再中断 30/60/120 秒；持续 spool 补传去重后无重复样本，断链前尾部也可补齐或显式标 gap。
4. 补偿帧晚于基础 chunk 上传到达，生成 repair version，不修改旧 SHA。
5. 眼镜重启后 boot_id 变化，序号重置不与旧流冲突。
6. ACK_AUDIO_RANGES 在连续游标后留下空洞并 selective ACK 更晚区间；眼镜不得删除空洞 block。
7. ENTER_PRIVATE 后注入延迟音频回调；旧 capture_generation 内容不落盘、不发送，binding 以返回的最后序号或 UNKNOWN 边界关闭。
8. M3 关闭 phone mic fallback 时，链路不可恢复部分写 missing range；开启能力的兼容测试才要求独立 PHONE stream、1-2 秒重叠和 source history。

### 29.3 视觉

1. A2 后 5 秒内产生首张 L1 attempt，之后每 60 秒一次，允许误差不超过 5 秒。
2. L3 在允许决策后 2 秒内开始，20 秒后即使续租也必须退出。
3. 相机 busy/timeout/存储满均产生 attempt 结果，不伪造 sequence。
4. 在文件、manifest、PHONE_COMMITTED、ACK1 各阶段强杀进程，恢复后不丢失、不重复、不提前删除。
5. 相同 capture ID 不同 SHA 返回冲突并隔离。

### 29.4 云端

1. 同一对象重复上传返回相同 ACK2。
2. 最终 manifest 先于最后 chunk 到达时不得提前 final。
3. 超时输出 final_with_gaps；迟到 repair 产生 version 2。
4. OCR 先于 ASR 到达时停在 WAITING_AUDIO_CONTEXT，最终证据版本可追溯。

## 30. M3 工程验收基线

| 领域 | M3 必须通过 | M3 可延后 |
|---|---|---|
| 音频源 | GLASSES 单源、A1/A2、binding OPEN/CLOSE、60 秒 chunk | PHONE 自动兜底 |
| 音频可靠性 | 持续 spool、range ACK、gap、repair、重启恢复 | 跨设备相关性对齐优化 |
| 隐私 | privacy lock、显式恢复、generation 拦截迟到回调、BOUNDARY_PENDING | 复杂地点/日程隐私自动化 |
| 视觉 | A2→L1；关键词/规则→L3；租约到期本地回落 | LLM/VLM 触发 |
| 持久化 | 图片 journaled bundle、PHONE_COMMITTED 后 ACK1 | 长期云端保留策略自动化 |
| 云端 | 幂等 Ingestion stub 可接收对象和 final manifest | 完整 ASR/OCR/索引（M5） |

M3 发布门：

1. 连续运行 60 分钟，音频 source_seq 缺口均可解释为已修复或显式 missing range。
2. 完成 A1→A2→L1→规则 L3→A2 结束的端到端流程，L3 决策与每张 attempt 可追溯。
3. 在音频、图片、ACK1、binding CLOSE 各持久化点执行进程强杀，恢复后不重复、不提前删除、不跨隐私恢复。
4. ENTER_PRIVATE 在实时流、spool、拍照回调和 P2P 同时活动时仍阻止 fence 后内容提交。
5. Fake adapter 契约测试与 Rokid 真机 M0 关键项使用同一 protocol/reducer，不维护两套业务逻辑。

## 31. M0 Rokid 真机验证清单

以下项目不改变业务契约，但决定 `RokidGlassDeviceAdapter` 的实现：

| 编号 | 验证项 | 通过标准 |
|---|---|---|
| R0-01 | 企业/消费固件、Glass SDK、Phone SDK 版本匹配 | 双端 Demo、注册和重连稳定 |
| R0-02 | `AudioCallback` 实际格式 | 明确采样率、声道、位宽、帧长、处理链 |
| R0-03 | 眼镜端 Opus 实时编码 | CPU/温度可接受，无持续积压 |
| R0-04 | 自定义带帧头音频数据通道 | 手机能收到完整 `AudioFrameV1` 并检测 source_seq |
| R0-05 | 经典蓝牙音频与 P2P 流对比 60 分钟 | 选出默认实时通道；记录稳定性、吞吐、功耗和温度 |
| R0-06 | 蓝牙控制与 P2P 数据并存 | P2P 断开时 A0 命令仍可在蓝牙到达 |
| R0-07 | 本地 L1/L3 timer | 锁屏、后台 30 分钟后仍按租约执行并到期停止 |
| R0-08 | `takePhoto` 回调和路径 | 确认分辨率、延迟、错误回调、可用目录 |
| R0-09 | 加密文件经 SDK 发送 | 手机 V2 回调接收并校验 SHA |
| R0-10 | App/设备重启恢复 | SQLite、tombstone、租约失效和 boot_id 正确 |
| R0-11 | 电量/温度来源 | 明确 SDK 或 Android API；无数据时报告 UNKNOWN |
| R0-12 | 相机/音频冲突 | L1/L3 拍照不造成不可接受音频缺口 |
| R0-13 | 固定焦相机文本质量 | 屏幕、白板、A4、药盒在目标距离 OCR 可用 |
| R0-14 | P2P 与手机公网并存 | 手机仍可向业务云上传或可可靠切换网络 |

任何真机限制只能通过 adapter 能力降级和策略配置解决，不得绕过状态、隐私、幂等或持久化契约。

## 32. 后续开发路线

本路线从当前 M3 开发设计基线出发，采用“先证明协议闭环，再接真机，再形成产品闭环”的推进方式。阶段完成以可复现证据和退出门为准，不以功能演示或日历时间代替验收；在取得 M0 真机数据前，不承诺依赖设备能力的固定排期。

### 32.1 路线原则

1. **单一业务逻辑**：Fake Glass、Fake Phone 和 Rokid 真机必须复用同一套 command、event、reducer、幂等和持久化逻辑；设备 SDK 仅通过 adapter 接入。
2. **协议先于适配**：先冻结字段、状态、失败语义和恢复规则，再实现设备调用，避免 SDK 行为反向污染业务状态机。
3. **阶段门以证据为准**：每一阶段必须同时交付代码、自动化测试、遥测证据和已知限制；“能够演示一次”不等于阶段完成。
4. **严格控制 M3 范围**：M3 只要求 `GLASSES` 主音源、确定性规则触发、视觉/听觉耐久链路和 Phone Ingestion stub。Phone 麦克风兜底、模型触发和正式 ASR/OCR 不阻塞 M3。
5. **稳定契约、可替换能力**：M4 只替换 trigger provider，M5 只消费 M3 已冻结的 manifest、chunk、gap 和 binding，不得绕过隐私门、代际隔离或耐久 ACK。

### 32.2 阶段路线图

| 阶段 | 核心目标 | 必须交付 | 退出门 |
|---|---|---|---|
| D0：协议内核 | 把本文档变成可执行契约 | schema、纯 reducer、SQLite / journal migration、Fake adapter、fixture 生成器、contract test | 命令幂等、乱序/重复事件、`capture_generation`、binding OPEN/CHECKPOINT/CLOSE、精确 range ACK、崩溃恢复测试全部通过 |
| M0：真机可行性 | 验证 Rokid SDK、相机、音频、P2P 和计时器的真实能力 | Glass/Phone adapter 骨架、能力矩阵、第 31 节逐项报告、不可满足项的降级 ADR | R0-01～R0-09、R0-12 无未解释失败；其余项要么通过，要么有不破坏协议的明确降级方案 |
| M1：音频耐久薄切片 | 形成从触发前音频到 Phone 耐久确认的完整链路 | A0/A1/A2、preroll、source identity、binding、60 秒 chunk、连续 spool、range ACK、gap marker | 连续 60 分钟证据链完整；触发前丢失不超过 2 秒；隐私停止、断链重传、进程重启和空间不足路径通过 |
| M2：视觉耐久薄切片 | 形成从 lease 到 Phone `ACK1` 的视觉闭环 | L1 lease、首拍/周期拍摄、journaled bundle、manifest、P2P batch、ACK1 后回收 | 首拍、周期误差、重复传输、断电/崩溃恢复和“未 ACK1 不删除”全部满足本文约束 |
| M3：工程 MVP | 用确定性规则把听觉和视觉链路联成可运行系统 | 关键词 trigger provider、policy gate、完整遥测、Phone Ingestion stub、端到端演示脚本和验收报告 | 第 30 节全部 release gate 通过；无 P0 数据丢失、隐私泄漏或代际串扰缺陷 |
| M5-T：产品薄闭环 | 让 M3 原始证据变成用户可检索、可追溯的结果 | 正式 ingestion、ASR、OCR、基础索引、证据定位、版本状态和删除链路 | 一段真实会话可从原始 chunk/bundle 追溯到文本结果；`final/gap/revised` 语义和删除一致性通过 |
| M4：智能触发 | 在不改变安全与耐久契约的前提下提高触发质量 | 本地 LLM/ML trigger provider、离线评测集、阈值配置、灰度开关和规则回退 | 误触发、漏触发、延迟、功耗达到产品目标；关闭模型后可无损回退确定性规则 |
| M5：记忆产品化 | 扩展检索、证据和跨版本处理能力 | 增量处理、重算/回填、语义检索、证据查看器、保留策略和运营工具 | 输出可查、可解释、可删除、可重算；管线升级不修改 M3 原始证据身份 |
| P：发布加固 | 从内部 MVP 进入外部试用 | 配对密钥、权限/同意、数据保留、可观测性、告警/runbook、兼容矩阵、升级/回滚 | 安全、隐私、长稳、升级和设备兼容验收通过，P0/P1 缺陷有明确发布门 |

说明：`M5-T` 是最小产品闭环，建议在 M3 后优先完成；M4 的模型化触发属于质量增强，不应阻塞用户第一次获得可检索结果。若 M0 证明 Glass 音频链路无法达到 M1 门槛，则 Phone 麦克风兜底应升级为 M1 的条件分支，并通过新的 `source_id` 和 binding 纳入同一契约，而不是临时旁路。

### 32.3 关键依赖与可并行关系

```text
D0 协议内核 ──> M0 真机验证 ──> M1 音频耐久 ──┐
      │                │                         ├─> M3 工程 MVP ──> M5-T 产品薄闭环 ──> P 发布加固
      │                └────────> M2 视觉耐久 ──┘             │
      └──────────────> Phone Ingestion stub ──────────────────┘
                                                               └─> M4 智能触发 ──> M5 能力扩展
```

- D0 schema 和 reducer 接口冻结后，Glass adapter、Phone 持久化、Fake 故障注入可并行开发。
- M0 真机实验可以与 Fake contract test 并行，但所有真机特例必须回写为 adapter capability 或正式 ADR，不能写入 reducer 的设备分支。
- M1 和 M2 可以由不同工作流并行推进；M3 集成必须等待二者各自通过耐久与隐私退出门。
- Phone Ingestion stub 在 object schema 稳定后即可并行开发，只负责幂等接收和回执，不提前耦合正式 ASR/OCR。
- M4 与 M5-T 可以并行，但资源冲突时优先 M5-T，因为它直接决定 MVP 是否形成用户价值闭环。

### 32.4 每阶段强制交付物

每一个里程碑都必须产生以下六类可审计产物：

1. **实现**：协议模块、adapter 或 pipeline 代码，不允许只有原型脚本；
2. **测试**：contract、property、故障注入和必要的真机测试，能够在固定 fixture 上重放；
3. **证据**：关键 trace、状态快照、对象 manifest、ACK 记录和测试报告；
4. **可观测性**：阶段新增状态和失败分支必须有 metric、structured log 或 diagnostic export；
5. **变更说明**：schema migration、配置默认值、兼容策略和回滚路径；
6. **偏差记录**：任何不符合本文契约的设备限制必须形成 ADR，说明影响、降级、恢复和后续清偿计划。

阶段评审至少回答：数据是否耐久、是否可重放、是否可能跨隐私代际、失败后由谁恢复、重复执行是否安全、用户能否删除，以及该阶段新增了哪些无法自动验证的假设。

### 32.5 M3 后的产品推进优先级

1. **先做 M5-T，而不是先做 M4**：确定性规则已经足以验证采集链路，下一步最重要的是让用户得到可检索、可回看、可追溯的文本结果。
2. **再以数据驱动 M4**：使用 M3/M5-T 获得的匿名化或授权样本建立 trigger 离线评测集，先量化误触发和漏触发，再替换 provider。
3. **按真机数据决定音源兜底**：仅当 M0/M1 证明 Glass 音频在断链、功耗或后台限制上无法达标时，引入 Phone mic；其身份、绑定、权限和删除语义必须与 Glass source 一致。
4. **最后进入发布加固**：在用户价值闭环和触发质量稳定后，再扩大设备矩阵、运营能力和外部试用范围，避免用发布工程掩盖协议缺口。

### 32.6 推荐实施顺序

1. 冻结本协议的数据结构、状态机和失败语义；
2. 实现纯 reducer 与 SQLite / journal 持久化层；
3. 用 Fake Glass / Fake Phone 跑 contract/property test 和崩溃注入；
4. 接入 Rokid 真机，完成第 31 节 M0 清单并记录 capability/ADR；
5. 打通音频 A0/A1/A2、preroll、source identity、binding 和 60 秒 chunk；
6. 打通连续音频 spool、精确 range ACK 与 gap marker，完成 M1；
7. 打通视觉 lease、拍摄、bundle 落盘、P2P 批次和 ACK1，完成 M2；
8. 接入确定性规则、policy gate、完整遥测和 Phone Ingestion stub，完成 M3 集成；
9. 通过第 30 节故障、隐私、耐久和功耗门后，冻结 capture protocol v1；
10. 在不修改原始证据契约的前提下完成 M5-T：ASR/OCR、基础索引和证据追溯；
11. 以真实评测集推进 M4 模型触发，并始终保留规则回退；
12. 扩展 M5 产品能力并完成发布加固。

## 33. 已冻结的关键决策

| 决策 | v1 结论 |
|---|---|
| 主硬件 | Rokid Glasses / Glass3 |
| 智能位置 | 手机做状态和触发；眼镜执行租约与本地硬门 |
| 控制与数据 | 蓝牙控制；实时音频在经典蓝牙/P2P 中协商；P2P 传文件；适配器屏蔽 SDK 细节 |
| A1 帧身份 | source_kind + stream_id + boot_id + source_seq；conversation_id 可空 |
| 会话绑定 | 追加式 OPEN/CHECKPOINT/CLOSE；OPEN 至 CLOSE 区间拥有正式归属 |
| M3 音源 | 强制 GLASSES 单源；PHONE 数据模型保留、自动兜底可延后，未启用时显式 gap |
| 视觉调度 | 手机签发租约，眼镜本地 timer，不逐张远程调用 |
| 隐私防乱序 | control_epoch + privacy_lock + capture_generation；A0 增加 epoch，恢复引用 lock |
| 图片原子性 | SQLite journaled bundle；JSONL 仅作导出视图 |
| 音频迟到修复 | 不改基础 chunk；新增 repair segment 和 assembly version |
| 文件确认 | SDK 完成不算 ACK；PHONE_COMMITTED 后 ACK1，云端持久化后 ACK2 |
| 时钟 | 单调时钟排序，ClockMap 对齐 UTC，记录不确定度 |
| 云端定稿 | source range 完整性屏障；缺片和迟到数据版本化 |

## 34. 官方参考

- [Rokid Glass3 SDK 概览](https://x-docs.rokid.com/docs/terminal-sdk/getting-started/%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97.html)
- [Rokid Glass3 手机端 SDK API](https://x-docs.rokid.com/docs/terminal-sdk/api-reference/Glass3%20%20SDK%28%E6%89%8B%E6%9C%BA%E7%AB%AF%29%20API%E6%96%87%E6%A1%A3.html)
- [Rokid Glass3 眼镜端 SDK API](https://x-docs.rokid.com/docs/terminal-sdk/api-reference/Glass3%20%20SDK%28%E7%9C%BC%E9%95%9C%E7%AB%AF%29%20API%E6%96%87%E6%A1%A3.html)
- [Rokid Glass3 P2P 连接排查](https://x-docs.rokid.com/docs/faq/P2P%E9%97%AE%E9%A2%98%E6%8E%92%E6%9F%A5.html)
- [Rokid Glass3 Demo 运行指南](https://x-docs.rokid.com/docs/downloads/demo-guide.html)
