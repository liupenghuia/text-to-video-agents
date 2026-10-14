# I002：本地基础工程、身份与项目

日期：2026-10-13。状态：Planned，未开始编码。依据 [ADR-005](../adr/ADR-005-coding-baseline.md)、[边界](../architecture/boundaries.md)、[门禁](../architecture/gates.md)、[TODO](../../todo.md)。

## 本轮交付边界

交付独立 frontend/、backend/ 工程；Vue/Vite 前端、NestJS API/Worker两个入口；本地 PostgreSQL/Redis/MinIO Compose；配置校验/健康检查/日志；用户Session、私有Project/Episode；OpenAPI与前端生成类型、CI及启动说明。

本轮不实现收费调用、公开注册/邮件、完整业务Job/Run/Outbox/预算账本、TTS/口型、角色工作台、时间轴或成片。Mock仅用于契约测试/基础设施连通性，不在主界面伪装成已生成内容；业务异步链路在I003/I004实现。

## 模块、契约与页面

- identity拥有User/Session；projects拥有Project/Episode及逻辑删除。只有两个模块先建立真实业务代码，不提前创建所有空服务。
- 本地管理员通过一次性开发引导命令创建；密码安全输入并强哈希，不将管理员密码硬编码或写日志；测试账号只在隔离测试环境使用。邀请/公开注册业务后续完善，不开放无验证的生产注册。
- API最小接口：GET /api/v1/health/live、GET /api/v1/health/ready；GET /api/v1/auth/csrf、POST /api/v1/auth/login、POST /api/v1/auth/logout、GET /api/v1/me；GET/POST /api/v1/projects、GET/PATCH/DELETE /api/v1/projects/:id；GET/POST /api/v1/projects/:id/episodes、GET/PATCH/DELETE /api/v1/episodes/:id。
- csrf接口供登录前与会话中取得令牌，变更接口在Header提交并校验来源；登录接受email/password、返回当前用户摘要并设置cookie，前端不保存会话Token。me只返回必要id/email/role；邀请注册和重置不加入本轮API。
- Project接受title/description，ownerId由会话决定，revision用于更新；Episode接受title/description/排序，项目归属不可由客户端任意修改。ID与时间由后端产生；派生计数/权限不是可写字段；全量DTO/错误schema实现时导出OpenAPI。
- 更新/删除携带revision，过期返回409；未登录401，非拥有者资源统一404以免泄漏；参数错误400；创建201。列表稳定游标分页默认20、最大100，无无界扫描。
- Session使用服务端持久化和HttpOnly cookie，本地与生产cookie配置分开；登录/退出/变更请求有CSRF/限流保护。会话有效期本地默认24小时、可配置，非隐式永久会话；退出撤销会话，受保护查询失败需重新登录。
- 页面：登录、项目列表/创建、项目详情与Episode列表/创建/编辑；loading/empty/error/冲突/未保存提示、权限跳转、窄屏布局。生成入口明确“后续阶段尚未实现”，不返回虚假进度。
- Worker有独立启动/退出/健康日志；可用隔离的非业务队列探针验证Redis，但无真实Provider请求及业务扣费。SSE项目任务流属于I003，本轮不创建假事件API。

## 验收与证据

- [ ] 前后端各自安装、类型检查、测试、构建；API/Worker独立启动，启动命令写README。
- [ ] 开发Compose可连接三项基础设施；缺配置启动失败、依赖未就绪readiness失败，日志无秘密。
- [ ] 空库迁移/一次性引导可执行，不在每个Worker启动时并发迁移。
- [ ] 两用户的项目及EpisodeCRUD隔离，客户端伪造ownerId无效；项目逻辑删除后资源访问失败。
- [ ] 登录/退出/过期、CSRF和限流、旧revision冲突有相关测试，失败不吞异常或丢输入。
- [ ] OpenAPI生成与前端类型一致，CI检查类型/lint/边界/相关测试/构建；普通CI不访问真实AI或花费。
- [ ] 实际浏览器验证登录、创建/编辑/删除及空/错误状态，桌面与窄屏无主要溢出，关键控件键盘可用。
- [ ] 文档与源码能力一致；没有把“本地启动”写成“生产可发布/1000并发已通过”。

本记录勾选须附实际命令/运行结果，当前全部待验收。完成后进入I003版本/任务/预算；真实效果探索可并行研究，但执行费用与能力门禁不省略。
