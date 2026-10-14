# ADR-001：前后端独立与后端模块化

日期：2026-10-13。状态：Accepted（设计选择，未实现）。

## 背景与选择

用户要求前端、后端目录明确，独立维护与部署。选择 frontend/、backend/ 两个项目，各有依赖与锁文件；后端 API/Worker 共用模块、分进程运行；infra/ 为全系统部署配置，docs/ 为共享资料。

放弃根 pnpm workspace/公共 packages/，减少跨项目依赖理解成本；暂不拆业务微服务，避免早期分布式事务与运维开销。

## 影响与验证

前后端以 OpenAPI/REST/SSE 联系；模块端口与数据所有权见 [边界](../architecture/boundaries.md)。独立构建、契约同步、依赖检查、API/Worker 共用规则及独立运行列入工程验收，详见 [门禁](../architecture/gates.md)。
