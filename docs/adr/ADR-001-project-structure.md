# ADR-001：前后端独立与后端模块化

日期：2026-10-13。状态：Accepted（设计选择，未实现）。

## 背景与选择

用户要求前端、后端目录明确，独立维护与部署。选择 frontend/、backend/ 两个项目，各有依赖与锁文件；后端 API/Worker 共用模块、分进程运行；infra/ 为全系统部署配置，docs/ 为共享资料。

放弃根 pnpm workspace/公共 packages/，减少跨项目依赖理解成本；暂不拆业务微服务，避免早期分布式事务与运维开销。

## 目标目录与依赖方向

目录随已实现能力建立，不预建空模块或空文档；模块复杂时才在自身内部细分 application/domain/ports。

```text
text-to-video-agent/
├── frontend/                 # 独立依赖与锁文件；页面、API 客户端、生成类型、前端测试
├── backend/                  # 独立依赖与锁文件
│   ├── prisma/               # schema 与迁移
│   ├── src/api/              # HTTP/SSE 入口、Controller、DTO、Guard
│   ├── src/worker/           # 队列运行入口、Processor 注册
│   ├── src/modules/          # 业务模块及其公开用例
│   ├── src/workflow/         # Run、阶段执行、恢复与门禁编排
│   ├── src/agents/           # Prompt、创作策略、结构化输出
│   ├── src/infrastructure/   # database、queue、provider、storage、media 端口实现
│   └── src/common/           # 配置、日志、基础错误；不放业务规则
├── infra/                    # Compose、Nginx、发布拓扑；不放业务源码或真实秘密
├── tests/e2e/                # 跨前后端关键流程
├── docs/                     # 版本管理的范围、架构、ADR、迭代与调研资料
└── .local-meta/              # 忽略 Git 的个人记忆、草稿、任务接续
```

前端只通过 REST/SSE 和 OpenAPI 生成契约访问后端；不导入后端源码、ORM、队列或 Provider SDK。API 与 Worker 是后端两个组合入口，调用同一模块公开用例，不形成两套业务规则。

## 影响与验证

前后端以 OpenAPI/REST/SSE 联系；模块端口与数据所有权见 [边界](../architecture/boundaries.md)。独立构建、契约同步、依赖检查、API/Worker 共用规则及独立运行列入工程验收，详见 [门禁](../architecture/gates.md)。
