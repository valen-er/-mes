# MES 代码逻辑分析（`new_open_mes_server`）

## 1. 总体分层与模块边界

- 后端是 **Spring Boot 多模块**结构，根工程 `new_open_mes_server/pom.xml` 聚合 `metaxk-server`（启动器）与 `metaxk-module-mes`（MES 业务模块）。
- 启动入口 `MetaxkServerApplication` 通过 `${metaxk.info.base-package}.module` 扫描业务包，并开启定时任务。
- MES 模块按典型三层拆分：
  - `controller/admin/*`：按领域暴露管理端 REST API；
  - `service/*` 与 `service/impl/*`：业务编排；
  - `dal/mysql/*` + `dal/dataobject/*`：MyBatis-Plus 数据访问与实体。

## 2. 关键技术栈与公共能力

- `metaxk-module-mes-biz/pom.xml` 显示依赖了 Web、Security、MyBatis、Excel、操作日志等 starter，说明 MES 业务运行在统一基础设施之上。
- API 统一返回 `CommonResult` / `PageResult`，权限通过 `@PreAuthorize` 控制，符合“控制器薄、服务承载业务”的设计。
- `MesConfig` + `ServerConfig` 提供文件存储路径与服务 URL 组装；`CommonController` 封装通用上传/下载，供业务模块复用。

## 3. 生产工单主链路（`pro`）

### 3.1 查询与展示

- `WorkOrderController#list` 调用分页查询后，会对日期做字符串截断、并按状态排序（`NOSCHEDUL -> SCHEDUL -> COMPLETED`）。
- 分页条件在 `WorkOrderMapper#selectPage` 中按字段动态拼装，包含订单号、产品、客户、日期等过滤项。

### 3.2 外部订单同步（核心）

- `WorkOrderController#orderSynchronization` 触发 `WorkOrderServiceImpl#syncOrders`。
- `syncOrders` 的逻辑不是只写工单，而是**一并补齐主数据**：
  1. 拉取第三方接口 JSON；
  2. 仅处理 `status = PREPARE` 的记录；
  3. 若缺失则新增：工单、工单BOM、物料、物料分类、车间、工序、工位、工艺路线、产品-工艺关联。
- 该设计的优点是“一次同步补全上下游基础数据”，但也带来一致性压力（见风险部分）。

### 3.3 二维码与文件上传

- `WorkOrderServiceImpl` 通过 `BarcodeUtil` 生成临时二维码图片，再经 `FileClientFactory` 上传到文件服务，并清理本地临时文件。
- 这是典型“本地生成 + 统一文件中心托管”模式，避免业务库存二进制。

## 4. 入库单链路（`order`）

- `InboundServiceImpl#saveInbound` 保存入库单时，会遍历行项目：
  - 写入入库明细；
  - 同步更新采购明细状态（置为已入库）。
- `updateInbound` 会先回滚旧明细对应采购状态，再删除旧明细、写入新明细并再次置状态。
- 这体现了“**单据头 + 明细 + 关联单据状态回写**”的典型 ERP/MES 业务编排。

## 5. 编码规则能力（自动编码）

- `AutoCodeUtil#genSerialCode` 通过“规则 + 组成部分”生成业务编码，支持：
  - 按 part 动态组装；
  - 流水段识别；
  - 左/右补位；
  - 结果落库（更新最近流水号与生成记录）。
- 方法加了 `synchronized`，表示作者在单实例内考虑了并发冲突；但分布式场景仍需 DB 锁/Redis 锁兜底。

## 6. 当前代码的逻辑特征总结

- **优点**
  - 领域覆盖广（计划、工艺、订单、质检、仓储等）。
  - 分层清晰，Controller/Service/Mapper 职责基本明确。
  - 大量查询封装在 Mapper 默认方法里，降低 Service 样板代码。

- **主要风险点（从代码直接可见）**
  1. `syncOrders` 跨多表写入但未见事务注解，外部调用失败或中途异常可能造成部分写入。
  2. 外部 URL 和 Token 在代码中硬编码（`callProductionOrderAPI`、`synchronous`），安全性与可运维性较弱。
  3. `WorkOrderController#list` 对日期做 `substring(0,10)`，若字段为空或格式异常会抛错。
  4. 多处状态值使用字符串字面量（如 `"1"`、`"PREPARE"`），建议统一枚举常量化。

## 7. 建议的优化优先级

1. **先做稳定性**：为同步与单据保存关键路径补 `@Transactional`，并在异常时记录可追踪日志。
2. **再做安全与配置化**：将第三方接口 URL/Token 改为配置中心或环境变量读取。
3. **再做可维护性**：状态码枚举化、日期处理改为类型化（`LocalDate/LocalDateTime`）并做空值防御。
4. **最后做性能**：对同步流程中的“查后插”改批量 upsert/缓存映射，减少 N 次数据库往返。

---

如果你愿意，我可以下一步继续做两件事之一：
1) 按“生产订单同步链路”画一份更细的时序图；
2) 直接给出一版可落地的重构清单（含事务边界、异常码、配置项改造点位）。
