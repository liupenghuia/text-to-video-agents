# 文生视频短剧平台：架构与实施 TODO

更新日期：2026-10-13。

状态：架构规划基线已确定；尚未创建前后端工程，尚未进行真实 Provider、构建、运行和部署验证。

评审已按委托收敛到 [ADR-005](./docs/adr/ADR-005-coding-baseline.md)，[checkList v4](./docs/product/checkList.md)末列记录最终决定及来源；第一轮按 [I002交付](./docs/iterations/I002-foundation.md)开工。技术待验证项不等于未决产品方向，也不等于已经通过。

本文是后续实施入口。勾选表示能力已经实现并有验证证据，不能因完成规划或创建目录而勾选实现任务。用户授权本文件、AGENTS.md 及 docs/ 下正式资料和共享记忆纳入版本管理；.local-meta/ 个人记录不提交。

## 1. 产品目标与范围

正式范围见 [scope](./docs/product/scope.md)，模块职责见 [boundaries](./docs/architecture/boundaries.md)，门禁定义与最小测试场景见 [gates](./docs/architecture/gates.md)。这些规范已确定，尚未在运行代码中落实。

建设完整前后端文生视频短剧平台，分阶段交付可持续维护的系统能力。产品化基础能力贯穿每阶段，不把权限、恢复和费用保护留到最后。

首期包括：用户注册登录、私有项目、多 Episode、剧本与分镜编辑、角色/场景参考素材、单 Shot 生成与版本选择、质量与安全审核、人工复核、对白/TTS/字幕、简单 Timeline、FFmpeg 成片、任务取消与恢复、项目预算、进度和错误展示、部署与备份。

首期主场景为普通话角色对白剧情短剧，必须画内口型同步，支持单角色与同镜头多角色轮流；角色形象与预设声音跨镜头/集一致，逐句表演可变，确认台词逐字保留。对白可跨 Shot，分镜按实际音频时间规划，详见 [音画规范](./docs/architecture/dialogue-timeline.md) 与 [ADR-004](./docs/adr/ADR-004-dialogue-timeline.md)。

首期不包括：组织协作、支付计费、高级多轨拖拽剪辑、全自动模型路由、长期自主 Agent Memory、自动配乐。保留清晰接口，不提前实现空泛扩展框架。

旁白故事短片、方言/多语言、样音定制、同时说话/抢话/合唱后续扩展；能力不支持时阻断，改正反打须用户确认，不静默降级。

## 2. 已确定的架构决策

| 决策 | 采用方案 |
|---|---|
| 代码组织 | frontend/ 与 backend/ 独立项目，同一个 Git 仓库 |
| 依赖管理 | 各自 package.json、pnpm-lock.yaml；根目录无 workspace 或公共 package.json |
| 前端 | Vue 3、Vite、TypeScript、Vue Router；Pinia 管理会话和工作台状态，服务端数据通过 API 获取 |
| 后端 | NestJS、TypeScript、Prisma、PostgreSQL |
| 后端模式 | 模块化业务系统，API 与 Worker 共用业务模块，分进程运行 |
| 编排 | PostgreSQL 保存事实与状态，Workflow 编排；Redis/BullMQ 调度 |
| 前后端契约 | REST + OpenAPI 生成前端类型；SSE 提供进度通知 |
| 存储与媒体 | 私有对象存储，S3 风格内部接口、本地 MinIO；FFmpeg/ffprobe |
| Provider | Mock + 真实适配器，按能力声明支持范围，不绑定业务代码 |
| 部署 | Docker Compose 云服务器，Nginx 同域提供前端及 /api；API/Worker 独立容器 |
| 身份与权限 | 单部署多用户、无组织租户；用户/管理员、项目归属与资源级授权 |
| 创作流程 | 确认台词→实际语音→按时长分镜→对白/分镜联合确认→视频/口型→审核→成片；阻断时暂停 |
| 正式资料 | docs/ 跟踪；README 导航；todo.md 管实施任务 |
| 协作接续 | 根 AGENTS.md + docs/PROJECT_MEMORY.md 共享；.local-meta/ 保存个人记忆和任务接续 |
| 当前运行 | 先本地，无云服务器；约1000注册用户规划，非并发承诺 |
| 费用 | CNY；用户月/项目累计/集累计/Run累计各¥100同时限制；累计测试总预算不限，不自动花费 |
| 本地保护目标 | 30天媒体保留、项目10GiB/实例素材50GiB，每日备份、RPO≤24h/RTO≤4h；容量/恢复未验证 |

上表属于设计决定，不代表已完成或已验证。工具版本在初始化时查官方兼容说明并锁定；真实 Provider、邮件服务和生产存储厂商由实测门禁选择，不能把未验证候选写成已接入。

## 3. 目标目录与职责

```text
text-to-video-agent/
├── README.md
├── todo.md
├── AGENTS.md                         # 团队 Agent 指引，纳入版本管理
├── .gitignore
├── frontend/
│   ├── package.json / pnpm-lock.yaml / .env.example
│   ├── src/
│   │   ├── app/                      # 初始化、路由、布局
│   │   ├── features/                 # auth/projects/scripts/storyboard/assets/review/timeline
│   │   ├── api/client/               # HTTP/SSE、错误处理
│   │   ├── api/generated/            # 契约生成，不手工修改
│   │   └── shared/                   # 通用组件、样式、纯工具
│   └── tests/
├── backend/
│   ├── package.json / pnpm-lock.yaml / .env.example
│   ├── prisma/                       # schema、迁移
│   ├── src/
│   │   ├── api/                      # HTTP/SSE 入口、Controller、DTO、Guard
│   │   ├── worker/                   # 队列运行入口、processor 注册
│   │   ├── modules/                  # identity/projects/characters/scripts/storyboard/assets/jobs/review/audio/timeline/usage
│   │   ├── workflow/                 # 跨模块编排、阶段执行、恢复
│   │   ├── agents/                   # Prompt、创作策略、结构化输出
│   │   ├── infrastructure/           # database/queue/providers/storage/media
│   │   └── common/                   # 配置、日志、基础错误
│   ├── contracts/                    # 导出的 OpenAPI
│   └── tests/                        # integration/contracts/fixtures
├── infra/                            # docker/compose/nginx
├── tests/e2e/                        # 跨前后端用户流程
├── docs/
│   ├── PROJECT_MEMORY.md             # 共享事实、约束和资料索引
│   ├── 项目调研与决策记录指南.md
│   ├── product/scope.md
│   ├── architecture/
│   │   ├── Agent 文生视频短剧系统完整执行流程.md
│   │   ├── AI视频生成系统架构与工作流 infograph.png
│   │   ├── boundaries.md
│   │   ├── gates.md
│   │   └── dialogue-timeline.md       # 存储、展示、跨 Shot 音画/口型调度
│   ├── research/
│   ├── adr/
│   ├── iterations/
│   └── tracking/                     # open-questions.md、risks.md
└── .local-meta/                      # 忽略 Git
    ├── PERSONAL_MEMORY.md           # 个人环境与本地接续
    └── sessions/                    # 按需创建任务记录
```

目录随功能建立，不创建大量空模块和空文档。模块复杂时再细分 application/domain/ports，避免每个模块机械套多层。

## 4. 依赖与数据所有权边界

- 前端只通过 REST/SSE 和生成契约接触后端；不导入后端源码、ORM 实体、队列或 Provider SDK。
- API 负责身份、资源授权、输入校验和应用服务调用；Worker 负责任务执行入口；两者共用模块服务，不重复业务规则。
- 每个模块拥有自己的写入规则，通过公开服务交互；禁止跨模块直接修改表。跨模块事务通过显式应用用例组织，不让 Controller 拼业务。
- Workflow 检查事实、编排依赖，通过所属模块服务迁移状态。核心规则不依赖 NestJS、BullMQ 或具体 Provider。
- Agent 只产出内容和建议，不自主写库、扣预算、入队、授权或触发 FFmpeg；模型输出必须校验。
- infrastructure 实现模块定义的访问接口；common 不放项目、Shot、Job 等业务逻辑。
- 用 lint/依赖检查及针对性测试约束以上规则。

## 5. 业务模型与版本策略

固定层级：User → Project → Episode → Scene → Shot。Episode 表示一集，不表示一次重试。

| 对象 | 所有者与职责 |
|---|---|
| User / Session | identity：身份、角色、会话 |
| Project / Episode | projects：归属、配置、预算关联 |
| Character / CharacterProfileVersion | characters：项目级形象/预设声音身份，跨镜头/集引用版本 |
| ScriptDraft / ScriptVersion | scripts：可编辑草稿、确认后的不可变剧本 |
| StoryboardVersion / Scene / Shot | storyboard：固定版本下的分镜、顺序、时长和连续性 |
| ShotGenerationVersion | storyboard：Prompt、输入引用、策略、模型配置的不可变快照 |
| GenerationRun / StageExecution | workflow：绑定确认版本、记录流程进度、暂停及门禁 |
| Job / JobAttempt / ProviderTask | jobs：逻辑任务、执行尝试、外部任务状态分别保存 |
| Asset / AssetReference | assets：归档、媒体信息、来源、引用和清理 |
| Review | review：关联具体版本的质量和安全结论 |
| DialogueVersion / SpeechAsset / SpeechAlignment | audio：固定文本、说话人、表演、真实时长和对齐，文件归 assets，不绑定单 Shot |
| PerformancePlanVersion / AudioClip / ShotPlacement / SpeechVisualBinding | timeline：视频前联合确认的音画规划、时间与画内外映射 |
| TimelineVersion / Render | timeline：最终选用音画素材、固定输入与成片版本 |
| UsageReservation / UsageRecord | usage：预算预占、实际用量和核对状态 |

默认规则：

- 编辑只修改草稿；确认生成新版本，GenerationRun 固定引用该版本。Job 不读取持续变化的草稿。
- 同Run首次确认后续阶段追加快照，不需另建Run；替换已确认/执行过的输入才新Run，不能把正常联合确认理解为取消旧Run。
- 首期一个 Episode 同时只允许一个活动完整 GenerationRun；独立 Shot 任务仍需检查输入和选用版本冲突。
- 更改已确认剧本使分镜待更新，更改分镜/参考素材使受影响 Shot 输入待更新；保留历史内容和成片。
- 重生成结果先成为候选，媒体/质量/安全检查通过才允许选用；首次可自动选用，已有选用版本时由用户显式替换。质量通过不能覆盖安全阻断。
- 记录 Shot 输入对其他素材版本的依赖，不把 continuous 一律解释为上一镜头尾帧。
- 更换选用素材后标记直接及传递依赖过期；无依赖镜头保持有效。用户确认受影响范围及预计费用后启动重生成。
- 已过期输入、旧批次或已取消任务的晚到结果不得覆盖当前选择或推进当前流程。
- Timeline/Render 固定引用 Asset 和版本，不随“当前选用版本”变化而暗中改变。
- 编辑/确认/选用使用 revision 乐观并发校验，冲突返回 409 并保留用户草稿。
- 文本/声音/表演变化失效对应 Speech、对齐、规划、口型与字幕；只改切点不重生成未变化语音。视频前规划与最终 Timeline 分开建模。

## 6. 门禁与审核默认值

默认流程：输入故事 → 生成剧本/台词 → 用户确认 → 角色声音/表演 → 实际音频/对齐 → 按时长规划分镜 → 用户联合试听确认 → 参考素材/Keyframe → 视频/口型归档 → 审核 → 最终音轨/字幕/Timeline → Render → 成片。

| 门禁 | 必须满足 |
|---|---|
| SCRIPT_CONFIRMED | Schema、内容检查通过，用户确认当前版本 |
| SPEECH_READY | 实际音频/时长/对齐已归档，逐字台词、说话人和声音符合确认版本 |
| STORYBOARD_CONFIRMED | 基于实际时长，Prompt/角色/依赖完整，用户联合试听确认 |
| PERFORMANCE_PLAN_CONFIRMED | 音轨/镜头/画内外映射有效，台词完整覆盖，用户确认 revision |
| GENERATION_ALLOWED | Provider 支持输入能力、权限有效、预算已预占、并发额度可用 |
| REFERENCES_READY | 必需参考素材有效、已归档、通过相关检查 |
| VIDEO_READY | 所有必需 Shot 的选用视频已归档并通过媒体校验 |
| LIP_SYNC_READY | 必需画内对白口型/说话人正确、非说话者不误动嘴；明确画外无需口型 |
| REVIEW_PASSED | 当前选用版本质量通过，安全阻断已解决，无过期依赖 |
| AUDIO_READY | 复用已准备语音、最终字幕映射完整，切镜头不重复/截断对白；不是视频之后才首次 TTS |
| TIMELINE_READY | 引用版本固定、时长有效、媒体可读；合法音轨重叠允许，不误判为错误 |
| RENDER_READY | 上述条件、输入快照和输出编码参数有效 |
| FINAL_READY | 成片校验并归档成功，而不只是 FFmpeg 返回成功 |

质量审核默认自动进行，失败/低置信度暂停并提供用户复核；安全复核仅管理员可执行，硬性策略禁令不可被质量确认绕过。阈值、误判和 Provider 限制必须用样本实测，缺证据时暂停，不默认通过。

门禁输出可解释的失败原因和恢复操作，不用单一 Project.status 表达所有子任务状态。

所有付费调用（含前置剧本、分镜、语音）检查预算/能力门禁。具体 B01～B15、E01～E06 责任与回归场景以正式 gates.md 为准，编号不是执行顺序。

## 7. 任务、取消与异常恢复

- 在同一事务保存业务变更、Job 和 Outbox；Dispatcher 至少一次投递，以稳定 Job ID 去重。
- 数据库保存输入快照、尝试记录、结果和事件；队列只传 ID/版本标识，不保存完整业务事实。
- Job 状态：PENDING、QUEUED、RUNNING、WAITING_PROVIDER、RECONCILIATION_REQUIRED、SUCCEEDED、FAILED、CANCELLED。取消请求独立记录，超时作为失败原因；流程人工暂停属于 Run/Stage，不混入每个 Job。
- JobAttempt 记录尝试与错误分类；预算不足/安全阻断/参数错误不自动重试，短暂网络/限流按退避重试。
- 普通可重试内部操作默认最多 3 次执行（含首次），退避带抖动；付费重新生成默认不自动重做。超时和并发按 Provider/任务类型配置，并由实测确定。
- 请求幂等按用户/操作/key/payload；内容复用与主动再次生成分开。FAILED可显式关联续办Job复用原外部结果；CANCELLED不自动续办，新意图仍检查预算，终态Job不原地重开。
- Job 采用租约、心跳和执行 token；过期执行者不能写入成功结果。恢复扫描器检查数据库事实，不仅依赖 Redis。
- 外部提交前持久化请求意图，支持幂等时传稳定请求键；提交超时/崩溃导致结果未知时转待核对，不能盲目再次收费调用。
- 视频提交、结果轮询、下载归档拆为短任务；Webhook/轮询共用幂等结果处理。回调验签、去重并核对任务归属。
- 阶段推进在事务内检查当前版本和门禁，以唯一约束防重复建任务；并发完成通知允许重复触发。
- 取消停止新任务和下游推进，撤销可撤销队列项，尽力取消外部任务；不承诺 Provider 停止收费。晚到结果进入隔离/归档流程，不恢复 Run。
- API/Worker 优雅退出；渲染子进程有超时、取消、资源限制和临时文件清理。

## 8. 权限、费用、素材和前端契约

### 身份与接口

- 默认邮箱/密码登录，密码强哈希；Session 存 PostgreSQL，浏览器使用 HttpOnly/Secure/SameSite Cookie，同域部署，变更请求有 CSRF 防护。
- 公开注册上线前必须具备邮箱验证、密码重置、限流；邮件适配器开发可 Mock，未接真实邮件时生产限制为管理员邀请。
- User 只能操作自己的资源；所有 Episode/Shot/Job/Asset/SSE/签名下载按项目归属校验；管理员能力单独授权且留审计。
- /api/v1、OpenAPI 为契约源；长操作返回 202 和持久化任务/批次标识。创建生成、重试、渲染接口支持 Idempotency-Key。
- 统一错误码、可恢复动作、requestId；列表游标分页、时间采用 UTC。前端不得显示密钥、原始内部栈或敏感 Provider 响应。
- SSE 带事件序号与版本；数据库保存必要事件，Redis 可通知但不作为唯一来源。断线重新拉取快照，游标过期也回到快照；无真实依据不显示伪精确进度百分比。

### 费用与素材

- 首个付费 Provider 前实现预算预占、并发安全检查与结算；结果未知保留预占待核对，避免提前释放后再次消费。
- 用户/项目/Provider/Worker 分层限流；管理实际用量，不在首期实现支付。预算与并发额度生产必须显式配置，缺配置拒绝付费生成。
- 用户UTC自然月¥100、项目/集/Run累计各¥100同时检查；无Run前置调用也检查归属额度与单操作¥100。未知费用保留原周期预占、核对后结算；不设累计测试总上限不代表无限重试。
- Provider 输出先下载、校验、存对象，再标记 Asset 可用；数据库保存对象 key，签名 URL 按需短期生成。
- 用户参考图优先通过受控上传获取；首期不支持任意远程 URL 导入。Provider 下载限制协议、域名、重定向、内网地址、尺寸、时间和实际媒体类型。
- 对象存储按最小权限使用，FFmpeg 参数采用结构化参数而非 shell 拼接；处理不可信素材设置 CPU/内存/时长限制。
- 删除为逻辑删除→停止新工作→保留期后按引用清理；运行中/被选用/被成片引用的素材不删除，晚到任务不复活项目。
- 30天到期先将相关保留根标EXPIRED/释放引用，其他未到期根及运行租约仍保护源素材；显式延期有记录，不能永远用历史引用阻止清理。只有实现并测试后才启用永久删除，见ADR-005。
- S3 风格抽象不承诺所有 OSS/COS 零改动；厂商适配器须验证上传、下载、签名、跨域和大文件行为。

### 前端验收

- 项目、剧本、分镜、Shot 版本、任务恢复、审核、Timeline、成片为核心工作台。
- 角色档案、逐句表演/试听、实际时长、对白/镜头/说话人区间、联合确认和过期影响必须可见；窄屏采用卡片及覆盖列表。
- 可编辑草稿与已确认版本明确展示；提交中防重复，冲突保留输入，过期依赖和费用影响在操作前展示。
- loading/empty/error/retry/cancel/permission、SSE 断线、上传失败可恢复；桌面/窄屏、键盘、焦点、标签与错误信息可访问。
- 首期 Timeline 采用镜头排序、时长/裁剪和简单音频字幕配置，无高级多轨拖拽；媒体默认以实际探测时长为依据，超预算时提示调整，不静默截断对白。

## 9. 分阶段实施 TODO 与通过条件

### I001：资料归位与边界固化

- [x] 将两份现有 Markdown 和架构图迁入第 3 节 docs/ 位置，保留内容与可追踪的重命名关系，修复全部相对链接（2026-10-13；链接检查通过）。
- [ ] 将现有文档里的 React/Monorepo/未决范围说明与本文对齐；保留历史提案时显式标注已被取代。
- [x] 建立 scope、boundaries、gates 与 ADR-001（前后端独立）、ADR-002（版本与确认流程）、ADR-003（状态机/Outbox/恢复）（2026-10-13；正式文档已建立，链接与门禁覆盖检查通过；执行器尚未实现）。
- [x] 完成清单评审决策/用户答复回写、ADR-005、Run/取消/去重/批次语义修正及I002交付边界（2026-10-13；文档一致性验证，应用未实现）。
- [ ] 正式 research/adr/iterations 纳入 Git，不再遵循旧指南中“全放本地笔记”的默认值。

验收：文档链接有效；已定、默认待验证、未实施三类状态清楚；每个门禁有实施位置与验收场景。

### I002：工程与身份基础

本轮只实现 [I002交付说明](./docs/iterations/I002-foundation.md)列明的身份/项目/集与工程能力；不加入真实收费生成、完整Job/Run/预算账本、公开注册/邮件、音画工作台。各自验收仍未完成。

- [ ] 初始化独立 Vue/NestJS 工程，各自锁定依赖；建立 API/Worker 两个运行入口和边界检查。
- [ ] 配置 PostgreSQL/Redis/MinIO 开发 Compose，配置校验、健康检查、结构化日志和 traceId。
- [ ] 实现身份、Session、项目归属、基本项目/Episode 管理与对应前端。
- [ ] 建立 OpenAPI 生成流程和 CI：契约同步、类型检查、相关测试、构建。

验收：分别安装/构建/启动；两用户越权失败；配置缺失明确失败；秘密不进入日志/客户端；数据库迁移从空库可执行。

### I003：任务与版本基础（先于付费生成）

- [ ] 实现草稿/确认版本、GenerationRun、Job/Attempt、Outbox、阶段记录、事件和 SSE 补偿。
- [ ] 实现幂等、执行租约、恢复扫描、取消、限流、预算预占/结算、输入依赖和过期标记。
- [ ] 使用 Mock 故障注入验证重复消息、提交结果未知、晚到结果、数据库与队列失联。

验收：重复请求/消费不重复推进或扣预算；重启可恢复；取消不复活；并发编辑返回冲突；旧批次不能覆盖新版本。

### 前置专项：核心音画路线验证（清单 H05 待复核）

- [x] 复核H01～H05并写入ADR-005、音画/边界/门禁及清单v4；批次/并发/出现实例/溯源设计已收敛，数据库/API实现仍待后续阶段。
- [ ] 在大规模 Provider 专用工作台投入前，制定并执行普通话固定音频、双角色同镜头轮流、跨切点和反应镜头的受控可行性测试；先明确预算与最小费用/核对/归档措施，无预算只做研究/Mock。
- [ ] 保存支持范围、失败样本/费用和通过/失败/证据不足结论；调整路线需 ADR，不降低核心需求。正式生产适配及完整故障测试仍在 I005。

该专项可在 I002/I003 的通用基础能力期间准备；执行真实测试不允许绕过预算、权限或安全门禁。未通过不阻断无厂商依赖的基础工程，但阻断对外开放真实口型能力。

### I004：剧本与分镜工作台

- [ ] 实现 Script/Storyboard Agent、输出校验、角色形象/预设声音档案、普通话固定台词/逐句表演与两次确认（第二次联合试听）。
- [ ] 实现 DialogueVersion/SpeechAsset/Alignment 与 Mock 实际音频，按时长规划 PerformancePlan/AudioClip/ShotPlacement/画内外 binding；文本核验与版本失效。
- [ ] 建立角色/场景/Shot 输入策略、显式依赖、快照及版本选择。
- [ ] 前端支持草稿、台词/角色试听与表演/实际时长、跨镜头轨道与区间、联合确认、失败恢复与影响预览。

验收：无效输出不推进；一句跨两镜头/一镜头多句合法，仅画内需口型；未联合确认不提交视频；前置语音/分镜付费调用单独经过预算门禁。

### I005：素材与真实 Provider

- [ ] Mock 支持成功/失败/超时/重复回调；Mock 视频输出真实可探测媒体，不只返回假 URL。
- [ ] 调研 LLM/图片/视频/TTS/lip-sync/审核 Provider：普通话/预设音色/固定音频台词、多角色口型/切点、成本/商用/核对/取消/归档。
- [ ] 每类先接一个通过合约测试的真实 Provider，保存日期、官方来源、实测证据和 ADR。
- [ ] 实现参考素材上传、媒体归档、下载限制、候选版本、单 Shot 重生成及依赖处理。
- [ ] 接真实 TTS/可信对齐，按交集调度口型并保存上下文/裁剪映射；验证单/多角色/画外，不支持时阻断并提供用户确认的正反打方案。

验收：归档/幂等/预算/下载恢复通过；普通配音不算口型；固定台词/说话人/声音/单多角色同步实测，无静默降级。

### I006：音画审核、最终 Timeline 与成片

- [ ] 实现自动质量/安全检查及分权人工复核，校准阈值并验证门禁。
- [ ] 复用 I004/I005 Speech/对齐，完成跨 Shot 连续主音轨/字幕，视频原对白静音，验证画内外/多角色/切点口型、台词完整与形象声音一致。
- [ ] 实现简单 Timeline、固定版本渲染、编码规范、输出探测、最终 Asset。
- [ ] 提供最终预览、签名下载和历史版本追溯。

验收：质量确认不绕过安全或台词/口型硬失败；过期输入禁止渲染；切镜头不截断/重复声音，反应镜头不误动嘴；成片音画字幕完整同步。

### I007：部署与运行验收

- [ ] API/Worker/前端镜像、生产 Compose、Nginx HTTPS/SSE、持久化卷、最小权限与资源限制。
- [ ] 迁移作为独立发布步骤；API/Worker 优雅退出与健康检查；渲染 Worker 独立并发配置。
- [ ] 数据库及对象存储备份、恢复演练、资产清理、日志脱敏、失败/积压/费用告警。
- [ ] 通过 Mock 全链 E2E 和至少一条受控预算的真实链路，记录容量、时间、成本与限制。

验收：Worker/Redis 重启可恢复；备份能恢复业务与素材引用；数据库恢复与晚到 Provider 结果能核对；未完成硬门禁不得标记可上线。

### 后续迭代

- [ ] 按实测需求扩容不同 Worker、迁移托管存储/数据库。
- [ ] 再评估团队协作、计费、高级时间线、自动路由；新增能力先写范围/ADR/迁移与验收，不提前实现。

## 10. 风险与验证清单

| 风险 | 必须验证的场景 |
|---|---|
| 版本变化覆盖新内容 | 旧 Job 晚到、选用版本替换、连续镜头传递依赖 |
| 重复付费 | 提交后保存前崩溃、超时、重复回调、并发预算竞争 |
| 状态错乱 | 多 Shot 同时完成、Redis 丢消息、租约过期、阶段重复触发 |
| 取消与删除复活 | 外部无法取消、删除后回调、清理与运行竞争 |
| 越权与安全绕过 | 枚举资源 ID、SSE/下载越权、用户冒充安全审核 |
| 不可信媒体 | 上传伪装、内网下载/重定向、超大文件、FFmpeg 超时 |
| 契约与界面失配 | OpenAPI 变化、SSE 乱序/断线、草稿冲突、移动布局 |
| 备份不可用 | 空环境恢复 DB 与素材、在途任务核对 |

单元测试靠近源码；集成测试验证数据库/队列/存储；Provider 合约测试验证外部边界；E2E 验证用户关键路径。每个关键门禁关联测试证据，不用纯快照替代行为测试。

## 11. 文档、记忆与接续规则

- todo.md：实施顺序、验收和未完成能力；docs/：正式规范、证据与决策；README：导航与可运行命令；docs/PROJECT_MEMORY.md：稳定事实与资料索引；个人记录不作为团队依赖。
- 原始流程位于 docs/architecture/，调研指南位于 docs/。冲突按当前明确用户指示→新接受 ADR→本文基线→历史提案处理；实际代码用于确认是否已实现，不能用未实施文档宣称运行行为。
- research 保存官方来源、日期、测试条件和待验证项；ADR 标记 Proposed/Accepted/Superseded/Rejected，并记录原因、影响、验证。
- 每轮 iterations/Ixxx 记录目标、关联需求/ADR、接口/数据/部署变化、测试证据和遗留问题。
- 文档变更不自动改代码。影响行为的决策必须关联迭代、代码和测试；完成后同步流程基线。
- 后续 Agent 先读 AGENTS.md、共享记忆、当前 TODO 与相关正式资料，再按需读取个人记录，并用源码和测试确认事实。
- 公共行为或约束变化在同一 PR 更新正式资料；有长期价值时同步共享记忆，并附日期和实际文件、测试、PR/提交证据。不要重复整份 TODO 或维护任务抢占状态。
- 个人环境、未验证推测和临时接续保留本地；多人冲突按主题和证据合并，不能简单覆盖。开发团队协作不扩大首期产品组织协作范围。
- 用户授权持续维护本项目共享及个人记忆，不写秘密，不修改全局记忆。2026-10-13 已完成协作资料整理；应用任务状态不因此改变。

下一步：按I002交付说明实现并验证基础工程；I001历史材料全文对齐继续独立维护，历史段落不得覆盖Accepted基线。当前应用尚未创建，依赖未安装，付费调用/上线未执行。
