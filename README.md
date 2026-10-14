# 文生视频短剧系统

## 架构实施入口

当前项目目标为完整前后端文生视频短剧平台，采用独立 `frontend/` 与 `backend/`，API 与 Worker 在后端内共用业务模块、分别运行。

- [架构与实施 TODO](./todo.md)：当前已确认架构、推荐默认规则、风险、阶段任务与验收门禁。后续启动工程从这里开始。
- 正式参考资料已归位到 `docs/`；下方现有资料仍是历史流程与调研基线，技术/组织方式等与 TODO 冲突时以新决定为准。
- [团队 Agent 规则](./AGENTS.md) 与 [共享项目记忆](./docs/PROJECT_MEMORY.md) 纳入版本管理，新成员按这两个入口接续。
- 个人记忆 `.local-meta/PERSONAL_MEMORY.md` 与任务记录 `.local-meta/sessions/` 忽略 Git；共享流程不依赖它们。

当前尚无前后端实现，不应将规划完成等同于能力完成或部署验证通过。

这是一个 Agent 文生视频短剧系统的架构与调研目录。

当前目录中的资料分为两类：

| 文档 | 用途 | 当前状态 |
|---|---|---|
| [Agent 文生视频短剧系统完整执行流程](./docs/architecture/Agent%20文生视频短剧系统完整执行流程.md) | 描述运行链路和工程分工 | 历史流程参考，待按 TODO 对齐 |
| [项目调研与决策记录指南](./docs/项目调研与决策记录指南.md) | 记录调研问题、候选方案和记录方法 | 历史调研基线与记录模板 |
| [AI视频生成系统架构与工作流 infograph](./docs/architecture/AI视频生成系统架构与工作流%20infograph.png) | 可视化系统架构、业务流程和数据模型 | 架构参考图 |

## 文档关系

```text
流程文档
  ↓ 描述系统应该怎样运行

调研与决策指南
  ↓ 指出哪些内容还需要验证和选择

research / ADR / risk register
  ↓ 保存调研证据、最终决策和风险

回写流程文档
  ↓ 形成稳定的系统规范
```

两份 Markdown 文档暂不合并：

- 流程文档保留完整运行链路，便于理解系统。
- 调研指南保留问题、候选方案和决策方法，便于持续评审。
- 已确认的长期架构选择，后续应使用 ADR 单独记录，避免把假设和结论混在流程说明中。

## 后续设计与验证重点

以下主题已在 TODO 中确定基线或设置验证门禁，具体完成状态以 TODO 为准：

1. `Project → Episode → Scene → Shot` 的数据层级。
2. Shot 连续性和首帧、尾帧、参考图输入策略。
3. 第一版视频、图片、LLM、TTS 和 Vision Provider。
4. Job 状态机、重试规则、取消和恢复机制。
5. Asset 版本、Provider URL 下载和对象存储策略。
6. Timeline、音频和字幕的时间对齐方式。
7. 第一版前端技术、登录权限、并发和成本上限。

## 推荐的后续记录目录

正式记录按需要在 `docs/` 下建立；`.local-meta/` 仅用于个人记忆、本地草稿和任务接续：

```text
docs/
  PROJECT_MEMORY.md
  product/scope.md
  architecture/
  research/
  adr/
  iterations/
  tracking/
    open-questions.md
    risks.md
```

记录原则：

```text
事实写进调研记录
选择写进 ADR
范围写进需求表
担忧写进风险登记
未决定事项写进开放问题
```

## 第一版建议目标

先验证最小闭环：

```text
输入故事
  ↓
结构化剧本
  ↓
3～5 个 Shot
  ↓
Keyframe / Mock 视频
  ↓
Asset 存储
  ↓
Timeline
  ↓
FFmpeg
  ↓
输出 MP4
```

当前资料足以支撑 MVP 设计，但在 Provider、权限、成本、内容安全、并发和生产部署等方面完成调研和决策前，不应把流程文档视为生产系统最终规格。
