# ADR-003：数据库事实、Outbox 与可核对恢复

日期：2026-10-13。状态：Accepted（设计选择，未实现）。

## 背景与选择

业务落库与队列投递、Provider 提交与本地记录均可能部分失败。选择 PostgreSQL 权威状态、事务 Outbox、BullMQ 至少一次执行、幂等/唯一约束、执行租约及数据库恢复扫描。

Job、Attempt、ProviderTask、Asset、Review、Run 各自表达状态；异步提交/轮询/归档拆短任务。外部结果未知先核对，不能宣称 exactly-once 或自动再次付费；取消阻断后续，但不保证外部停费。

经 [ADR-005](./ADR-005-coding-baseline.md) 澄清：非终态Job可内部Attempt重试；FAILED可显式建关联续办任务复用已有外部结果，CANCELLED不得自动续办。请求幂等与内容复用不同，相同输入的主动重生成必须新意图/候选及预算预占。

暂不引入 Temporal，不仅依赖 Redis 恢复，不用长期占用 Worker 等待 Provider，不把下载失败等同于生成失败。

## 影响与验证

所有付费操作先预算预占，未知结果保留待核对。恢复核对外部 ID/请求意图、版本与取消事实。验证重复消息、事务后入队失败、提交成功但记录失败、租约过期、重复回调、Redis 重启、取消后晚到，详见 [门禁](../architecture/gates.md)。
