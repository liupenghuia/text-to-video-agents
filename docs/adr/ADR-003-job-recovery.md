# ADR-003：数据库事实、Outbox 与可核对恢复

日期：2026-10-13。状态：Accepted（设计选择，未实现）。

## 背景与选择

业务落库与队列投递、Provider 提交与本地记录均可能部分失败。选择 PostgreSQL 权威状态、事务 Outbox、BullMQ 至少一次执行、幂等/唯一约束、执行租约及数据库恢复扫描。

Job、Attempt、ProviderTask、Asset、Review、Run 各自表达状态；异步提交/轮询/归档拆短任务。外部结果未知先核对，不能宣称 exactly-once 或自动再次付费；取消阻断后续，但不保证外部停费。

经 [ADR-005](./ADR-005-coding-baseline.md) 澄清：非终态Job可内部Attempt重试；FAILED可显式建关联续办任务复用已有外部结果，CANCELLED不得自动续办。请求幂等与内容复用不同，相同输入的主动重生成必须新意图/候选及预算预占。

暂不引入 Temporal，不仅依赖 Redis 恢复，不用长期占用 Worker 等待 Provider，不把下载失败等同于生成失败。

## 任务状态与执行细则

- Job 状态为 `PENDING`、`QUEUED`、`RUNNING`、`WAITING_PROVIDER`、`RECONCILIATION_REQUIRED`、`SUCCEEDED`、`FAILED`、`CANCELLED`。人工暂停属于 Run/Stage，不伪装成单个 Job 状态；取消请求与失败原因独立记录。
- 同一事务写入业务事实、Job 和 Outbox；Dispatcher 至少一次投递稳定 Job ID。队列只传 Job ID、版本或执行标识，不能承载完整业务事实；Worker 每次从 PostgreSQL 读取固定输入快照与当前取消/删除/版本状态。
- JobAttempt 记录尝试、执行 token、错误分类和外部关联。可重试的短暂网络/限流错误默认最多三次执行（含首次），使用带抖动的退避；预算不足、安全阻断、参数错误和新的付费生成不自动重试。具体超时/并发由 Provider 与任务类型配置并经实测确定。
- Job 使用租约、心跳和执行 token；租约过期的执行者不能提交成功结果。恢复扫描器以数据库事实检查卡住任务，不仅依赖 Redis 或 Worker 内存。
- 外部提交前持久化提交意图，并在 Provider 支持时传稳定幂等键；网络中断或崩溃导致提交结果未知时转入 `RECONCILIATION_REQUIRED`，先按外部 ID/幂等键核对，禁止盲目再次付费调用。
- Provider 提交、轮询、下载与归档是短任务；Webhook 与轮询使用同一幂等结果处理，回调必须验签、去重并核对项目归属。Provider 输出先下载、媒体校验并归档到对象存储，随后才能标记 Asset 可用。
- 阶段推进在事务中重查 revision、门禁、取消/删除和唯一推进键；并发完成通知可以重复到达，但不能重复推进。取消停止新任务与下游推进，尽力撤销队列/外部任务但不承诺停止收费；晚到结果隔离或归档，不复活 Run。
- API/Worker 优雅退出；媒体/渲染子进程有超时、取消、CPU/内存/时长限制及临时文件清理。对象存储使用最小权限，FFmpeg 采用结构化参数而非 shell 拼接。

## 影响与验证

所有付费操作先预算预占，未知结果保留待核对。恢复核对外部 ID/请求意图、版本与取消事实。验证重复消息、事务后入队失败、提交成功但记录失败、租约过期、重复回调、Redis 重启、取消后晚到，详见 [门禁](../architecture/gates.md)。
