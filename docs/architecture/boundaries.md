# 模块与运行边界

日期：2026-10-13。状态：Accepted（设计约束，尚无代码落实）。关联 [范围](../product/scope.md)、[门禁](./gates.md) 和 [架构决定](../adr/ADR-001-project-structure.md)。

## 应用边界

| 部分 | 职责 | 禁止承担 |
|---|---|---|
| frontend | 页面、草稿交互、进度、版本选择；REST/SSE 调用 | ORM/队列/后端源码、Provider 凭据、权威授权与门禁 |
| backend/api | HTTP/SSE、身份/资源授权、输入校验、应用服务调用 | 同步等待长生成、直接执行渲染、复制 Worker 业务逻辑 |
| backend/worker | 消费任务、调用应用服务、执行外部工作 | 绕过资源归属与快照、独立一套状态规则 |
| modules | 业务用例、状态迁移、数据写入规则、公开服务/端口 | 私自写其他模块的表 |
| workflow | GenerationRun、StageExecution、跨模块依赖和门禁 | 直接修改其他模块状态或让 Agent 决定流程权限 |
| agents | 创作、Prompt、结构化内容和辅助判断 | 自主授权、扣额度、写库、入队或启动 FFmpeg |
| infrastructure | 端口实现：数据库、队列、模型、对象存储、媒体 | 决定业务政策、公开内部凭据 |
| infra/ | 容器、服务拓扑、代理、发布配置 | 业务源码与真实秘密 |

API 和 Worker 两个组合入口共用模块；domain 规则不依赖框架/ORM/具体 Provider。模块的 application 服务依赖自身规则、访问端口和其他模块公开用例；基础设施实现端口并在启动入口注入，不形成反向循环。小模块不强制多层文件夹。

## 数据所有权

| 模块 | 权威写入对象 |
|---|---|
| identity | User、角色、Session、身份验证 |
| projects | Project、Episode、归属、删除意图 |
| characters | 项目级 Character、形象/预设声音档案版本，跨镜头/集选用身份 |
| scripts | 剧本草稿、确认版本 |
| storyboard | 分镜版本、Scene/Shot、生成输入版本、选用关系、依赖引用 |
| workflow | GenerationRun、StageExecution、阶段阻断与恢复 |
| jobs | Job、JobAttempt、ProviderTask、提交意图和执行租约 |
| assets | Asset、归档状态、媒体元数据、素材引用与清理 |
| review | 具体版本的质量/安全结论与复核审计 |
| audio | DialogueVersion、逐句表演、SpeechAsset 元数据与 SpeechAlignment/字幕对齐；不绑定单一 Shot |
| audio（补充） | SpeechBatchVersion与逐句源映射、DialogueOccurrence的文本/说话人身份；时间放置仍归timeline |
| timeline | PerformancePlanVersion、AudioClip、ShotPlacement、SpeechVisualBinding、最终 TimelineVersion/Render；实际时长音画映射 |
| usage | 预算配置、预占、实际用量、核对记录 |

跨模块用例需要原子性时，显式事务协调器通过模块公开服务使用同一事务上下文；不允许任意跨模块表写入。Outbox 与事件是基础设施能力，业务事实与待投递事件同事务落库。

语音文件归 assets、对白与对齐归 audio、角色身份归 characters、音画排布归 timeline，shot 出镜绑定归 storyboard。Workflow 按确认规划计算任务依赖，不把台词硬绑定 Shot；Worker 消费固定口型输入快照。存储与前端字段详见 [音画架构](./dialogue-timeline.md)。

Job生成溯源保存模板/编译器/适配器版本、Prompt编译结果、输入素材/区间和非敏感参数，访问按项目归属；费用/审核各归所属模块。媒体到期由assets协调引用释放，不能绕过timeline/job运行租约删文件；当前本地容量与保留决定见 [ADR-005](../adr/ADR-005-coding-baseline.md)。

## 信任与存储边界

- PostgreSQL 是业务事实源；Redis 是调度/通知辅助；对象存储保管大文件，不自行决定素材是否可用；SSE 是通知，不是状态权威。
- 每个资源追溯到 Project 所有者，HTTP、SSE、下载、批量操作均校验；Worker 使用已持久化的执行意图，并在执行/阶段提交时检查取消、删除与版本有效性。
- 管理员不默认拥有任意创作/下载权；安全复核通过专用、审计的授权入口访问必要内容，不把角色检查当成所有权检查的替代。
- 用户上传、LLM 输出、Provider 回调/文件、FFmpeg 输入皆不可信；分别进行 Schema、签名、网络/媒体与资源限制校验。
- 候选/隔离素材不能通过普通选用或下载入口绕过安全策略。签名链接按权限签发，日志不记录凭据。

## 契约与检查落点

- 后端 OpenAPI 为外部契约源，前端生成类型，不导入后端实体；长操作返回持久化 ID，预算/并发/版本检查由后端决定。
- 模块公开服务和事件是内部契约；记录 revision、run/version、request/trace 标识，避免晚到结果和重复投递造成误推进。
- lint/依赖检查禁止 frontend 导入 backend、modules 导入 api/worker、业务规则直接引用 Prisma/BullMQ/厂商 SDK。基础设施实现不得通过循环依赖回调启动入口。
- 授权、状态迁移、跨模块事务、预算竞争和版本失效由行为测试落实，不能以目录存在或文档检查代替。

## API、前端与媒体契约

- `/api/v1` 与 OpenAPI 是外部契约源。长操作返回 `202` 和持久化 Job/批次 ID；生成、续办、渲染等写操作使用 `Idempotency-Key`。错误返回稳定原因码、可恢复动作和 requestId；列表使用游标分页，时间统一 UTC。
- Session 归 identity 所有：密码强哈希，服务端持久化 Session，浏览器使用 HttpOnly/Secure/SameSite Cookie；变更请求须 CSRF 与限流保护。未具备邮箱验证、重置与限流前，生产仅管理员邀请，不开放公开注册。
- SSE 只通知，不是状态权威；事件携带序号与版本，必要事件可从数据库补偿。断线、游标过期或乱序时前端重新拉取资源快照，不能展示无依据的伪精确进度。
- 前端必须区分草稿和确认版本，防止重复提交并保留冲突输入；展示预算/过期影响。loading、empty、error、retry、cancel、permission、上传失败和 SSE 断线均有可恢复状态；桌面/窄屏、键盘、焦点、标签与错误文本是验收范围。
- Provider 文件与用户上传只能经受控入口进入：限制协议、域名、重定向、内网地址、尺寸、时长、实际媒体类型和资源消耗。数据库保存对象 key 与元数据，按权限签发短期下载链接；S3 风格接口不承诺未经适配即可兼容任意厂商。
