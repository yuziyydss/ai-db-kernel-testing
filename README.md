# AI 增强数据库内核测试平台建设方案（计划书）

| 项目 | 内容 |
|---|---|
| 文档版本 | v1.0 |
| 日期 | 2026-10-19 |
| 状态 | 待评审 |
| 目标读者 | 内核测试团队、内核开发团队、AI 平台团队、技术负责人 |
| 依据 | 2025–2026 年 SIGMOD/VLDB/ICDE/ICSE/ASE/ASPLOS 前沿成果与开源/产业实践（见附录 C） |

---

## 0. 执行摘要

**一句话**：用 12 个月、4–6 人的小队，把"LLM 造生成器 + 多 oracle 组合 + Agent triage/复现"的测试闭环跑起来，目标北极星指标：**每周确认唯一 bug 数 ≥ 基线 3 倍、误报率 ≤ 25%、每个确认 bug 的综合成本下降 50%**。

**为什么是现在**：

- 学术界已完成范式验证：LLM 发现 oracle（SIGMOD'26）、Agent 进化生成器（Gungnir）、生成器自适应接入新 DBMS（SQLancer++，ASPLOS'26）、白盒覆盖引导（DBcover，ICSE'26 工业）、小模型 + 搜索适配专有方言（MCTS，ICSE'26 工业）。
- 工业界已跑通：ClickHouse 把 fuzzer 合入主干；Google Big Sleep 在 fuzzing 覆盖不到的 SQLite 代码里找到漏洞；Anthropic 开源了"侦察→发现→验证→报告→修复"的完整 harness。
- 开源底座成熟：SQLancer（21 种 DBMS、oracle 全家桶）、BuzzHouse、Elle/Jepsen、确定性仿真（DST）。

**核心设计原则（不可妥协）**：

1. LLM 不进入百万级执行热路径；热路径必须确定性、可复现、便宜。
2. 任何 AI 产出的"发现"必须携带可执行证据链（最小复现 + 确定性环境 + oracle 依据 + 计划/覆盖证据 + 去重签名），否则不进人工 review。
3. Oracle 可靠性优先于生成多样性；宁可少报，不可刷屏。
4. 人保留严重性判断与 patch 签字权。

---

## 1. 背景与目标

### 1.1 现状与痛点（模板，启动时填写）

- [ ] 现有测试资产：单元测试 __ 个、集成测试 __ 个、随机 SQL 工具 ____、覆盖率工具 ____、历史 bug 库 __ 条
- [ ] 痛点排序：方言/特性覆盖不足 / 逻辑 bug 发现难 / 并发与分布式场景缺测试 / 性能回归发现晚 / 测试用例维护成本高 / 复现与定位耗时
- [ ] 现有 CI 环境：Jenkins / GitLab CI / 其他 ____；是否支持按需容器实例 ____

### 1.2 目标

| 层级 | 指标 | 基线（启动时测） | 3 个月 | 6 个月 | 12 个月 |
|---|---|---|---|---|---|
| 北极星 | 每周确认唯一 bug 数 | __ | ×1.5 | ×2 | ×3 |
| 北极星 | 每个确认 bug 的综合成本（算力+人力） | __ | -20% | -35% | -50% |
| 质量 | 上报误报率 | __ | ≤30% | ≤25% | ≤20% |
| 质量 | 上报重复率 | __ | ≤30% | ≤20% | ≤15% |
| 效率 | time-to-repro（发现→最小复现） | __ 小时 | -30% | -50% | -60% |
| 质量 | 回归逃逸率 | __ | 持平 | -10% | -20% |
| 领先 | 新特性接入测试成本 | 数周 | ≤1 周 | ≤2 天 | ≤1 天 |

### 1.3 范围

- **包含**：SQL 逻辑正确性（优化器/执行器/表达式）、存储引擎崩溃一致性、事务隔离、分布式一致性、性能/计划回归、兼容性
- **不包含**：SQL 注入安全扫描（已有 SAST/DAST）、上层应用测试、Benchmark 营销测试
- **决策点**：首期主攻模块（建议优化器或存储引擎二选一，见 §3）

---

## 2. 总体架构

### 2.1 分层架构

```text
┌────────────────────────────────────────────────────────────┐
│ L6 治理层：指标看板 / 预算配额 / 审计 / mutation 评估        │
├────────────────────────────────────────────────────────────┤
│ L5 Agent 层：生成器进化 / Oracle 提案 / Triage / 根因 / 修复建议│
│   （LLM 只活在这一层，全部产出带可执行 witness）              │
├────────────────────────────────────────────────────────────┤
│ L4 知识层：方言规则库 / 优化器规则 / 历史 bug 库 / 代码地图   │
│   / 覆盖率知识图谱（RAG + 向量 + 图）                        │
├────────────────────────────────────────────────────────────┤
│ L3 判定层：oracle 组合执行 / 复现 / 最小化 / 去重 / 二分      │
├────────────────────────────────────────────────────────────┤
│ L2 执行层：容器化 DB 生命周期 / 故障注入 / DST 模拟器 / 调度  │
├────────────────────────────────────────────────────────────┤
│ L1 生成层：SQLancer++ 派生生成器 / 数据合成 / 场景生成        │
│   （确定性、快、便宜；由 L5 的 Agent 进化，不逐条调用 LLM）    │
└────────────────────────────────────────────────────────────┘
```

### 2.2 数据流

```text
生成器 → 用例(SQL/数据/场景) → 执行器(多模式/多配置) → oracle 判定
   ↑                                                      │
   │                                                      ▼
   │                                            差异/失败 → 复现器(干净环境)
   │                                                      │
   │                                                      ▼
   │                                        最小化 + 去重 + 证据链
   │                                                      │
   └──────────── 知识回流（确认 bug/新 oracle/新规则） ←─ Agent triage
```

### 2.3 目录结构（仓库骨架）

```text
dbtest-platform/
├── generators/           # SQLancer++ 派生 + 自研生成器（方言适配层）
├── oracles/              # oracle 实现（TLP/NoREC/CODDTest/差分/…）
├── runners/              # DB 生命周期管理（容器、故障注入、DST）
├── triage/               # 复现、最小化(delta-debugging)、去重、证据链
├── agents/               # LLM agent 定义、prompt、witness replay 协议
├── knowledge/            # bug 语料、方言规则、代码地图、索引构建
├── eval/                 # 历史 bug replay、mutation、指标计算
├── dashboards/           # 看板配置
├── campaigns/            # 测试任务定义（YAML）
└── docs/                 # 本计划书、设计文档、复盘
```

---

## 3. 首期切入点选择（决策矩阵）

| 候选模块 | 发现价值 | 工程难度 | 数据可得性 | 建议 |
|---|---|---|---|---|
| 优化器（join/谓词下推/聚合） | 高（逻辑 bug 重灾区） | 中 | 好（oracle 成熟） | **推荐首发** |
| 存储引擎（WAL/崩溃恢复） | 高（数据正确性命门） | 高（需 DST/故障注入） | 中 | 第二期 |
| 事务/隔离 | 高（金融级场景刚需） | 中高（Elle 可复用） | 中 | 第二期 |
| 执行器（向量化/并行） | 中高 | 低（模式差分即可） | 好 | 可并行 |
| 分布式一致性 | 高 | 高 | 中 | 第三期 |
| 性能/计划回归 | 中（产品感知强） | 低 | 好 | 可并行（pg_plan_advice 类机制） |

**【决策】**：首期 = 优化器逻辑 bug + 执行器模式差分（工程上同源，风险低、见效快）；存储/事务在 M4 启动。

---

## 4. 生成层方案

### 4.1 基座：fork SQLancer++ 做方言适配

- 底座：`github.com/sqlancer/sqlancer`（Java，21 种 DBMS 适配层可抄）+ SQLancer++（ASPLOS'26）的自适应生成器架构。
- 任务：
  1. 实现目标引擎的 Provider（连接、建库、日志、expected-errors 配置）；
  2. 用 ParserFuzz 思路从引擎自带语法定义/文档提取 grammar rule，生成初始方言面；
  3. 打开 `--use-reducer` 类 delta-debugging 最小化。
- **验证方式**：跑 48 小时，要求 0 语法错误、语义错误率可解释（expected-errors 白名单收敛）。

### 4.2 LLM 进化生成器（Gungnir 模式，M2 起）

目标：让 LLM **修改生成器代码**来支持新特性（窗口函数新 frame、专有 hint、新类型……），而不是逐条生成 SQL。

**Witness Replay 协议（关键防幻觉机制）**：

```json
{
  "feature": "PARTITION BY .. RANGE BETWEEN INTERVAL",
  "witness": {
    "setup": ["CREATE TABLE t(a int, ts date);", "INSERT ..."],
    "generator_choices": {"rule_path": ["select","window","range_interval"], "params": {...}},
    "expected_ast_node": "window_spec.range_interval"
  },
  "validation": {
    "replay_result": "generated_sql_contains_feature",
    "db_accepts": true,
    "parse_tree_match": true
  }
}
```

- 流程：Agent 提交代码改动 + witness → CI 回放 witness → 生成器确实产出该特性且被引擎接受 → 才合并。
- 成本：每个新特性约 2–10 次 LLM 调用；之后生成免费。

### 4.3 数据合成

- 通用：按 schema + 分布参数生成（倾斜、NULL 率、唯一性、外键链）。
- 性能复现场景（M6+）：DBRepro 思路——只用元数据（列统计、直方图、行数）合成"代理数据库"，禁止拷贝生产数据。
- 边界数据工厂：NULL/极值/溢出/编码/时区/精度组合，LLM 负责枚举维度，采样留给随机。

### 4.4 覆盖引导（M2 起，A/B 验证）

- DBcover 方案（白盒）：覆盖率 + 调用图建索引，Agent 面向未覆盖函数生成定向用例。
- BuzzHouse 的反方观点：覆盖率引导在复杂 SQL 下性价比未必高。
- **做法**：两条路线各跑 2 周，用"确认 bug / 千用例"对比决定，不预设立场。

---

## 5. Oracle 方案（核心武器库）

### 5.1 实施顺序与 Checklist

| 优先级 | Oracle | 检出目标 | 实施要点 | 工期 |
|---|---|---|---|---|
| P0 | TLP（三值逻辑分区） | 谓词/过滤/聚合逻辑 bug | 复用 SQLancer 实现，适配方言 | 1 周 |
| P0 | NoREC | 优化器 bug | 同上 | 1 周 |
| P0 | 崩溃/内部错误 oracle | crash、assert、错误码越界 | expected-errors 白名单管理 | 3 天 |
| P1 | CODDTest（常量折叠/传播） | 表达式求值、子查询 | 会话内状态恒定假设 + 替换 | 2 周 |
| P1 | 执行模式差分 | 向量化 vs 标量、并行 vs 串行、不同 join 算法 | 同一查询多配置执行（抄 BuzzHouse） | 1 周 |
| P1 | DQP（强制不同计划） | 计划相关逻辑 bug | 用 hint / `pg_plan_advice` 类机制强制计划，比对结果 | 2 周 |
| P2 | ERIQ（VIEW ≡ CTE ≡ 临时表） | 中间结果表示一致性 | 2026 年新思路，实现量小、检出独到 | 1 周 |
| P2 | 数值语义 oracle | 浮点聚合假阳性治理 | 精确有理数真值；exact/bounded/indeterminate 三分类 | 2 周 |
| P2 | DQE（SELECT/UPDATE/DELETE 同谓词一致） | DML 逻辑 bug | 加追踪列 | 2 周 |
| P3 | LLM 发现的新 oracle | 长尾 | Agent 提案 → 人工评审 → 灰度（见 5.2） | 持续 |
| M4+ | Elle/历史检查（事务隔离） | G0/G1/G2 等 | 接 Elle 或自研 checker | 4 周 |
| M4+ | 集中式 vs 分布式模式差分 | 分布式执行/优化 bug | DistSQL 思路 | 3 周 |

### 5.2 LLM Oracle 提案管线（SIGMOD'26 思路的工程化）

1. Agent 输入：引擎文档章节 + 优化器规则列表 + 既有 oracle 样例。
2. 输出：oracle 候选（自然语言描述 + 变换函数伪代码 + 预期覆盖的 bug 类型）。
3. **三关验证**：
   - 静态关：评审 prompt 自检（是否可判定、是否有假阳性路径）；
   - 实证关：在历史版本（含已知 bug 的 commit）上回放，要求：已知 bug 命中 ≥1、正确版本零误报；
   - 灰度关：接入主流程但标记"观察中"，误报率超阈值自动下架。
4. 每个 oracle 入库必填：变换函数、适用方言面、已知盲区（如浮点、NULL 语义）、假阳性案例。

### 5.3 假阳性治理（硬门槛）

- 三重复现：干净容器、最新主干、关闭随机种子变化，三次一致才进人工队列。
- 浮点/时区/并行顺序类差异：一律走"数值语义 oracle"或显式豁免清单，不进人工。
- 每周误报复盘会（30 分钟）：新增 expected-errors / 修正 oracle。

---

## 6. 执行层方案

### 6.1 Runner

- DB 生命周期：容器镜像按 commit 构建（每日主干镜像）；`runner.py` 负责 init/seed/execute/snapshot/teardown。
- 并发：单机 N 个容器并行；任务队列（Redis/DB）分发；超时硬杀。
- 种子化：所有随机源（数据、查询、调度）记录 seed，失败可 100% 复演。

### 6.2 故障注入（M4 起，存储/事务）

- 进程级：SIGKILL/SIGSTOP 矩阵（BuzzHouse/strata 式崩溃矩阵）。
- 磁盘级：断电语义（fsync 丢失、写撕裂）、位腐化。
- 网络级：分区、延迟、丢包、乱序（tc/netem 或模拟网络）。
- 长期方向：**确定性仿真（DST）**——为存储/复制层建模拟 IO，使任何失败可逐字节复现（FoundationDB/TigerBeetle 模式，1 小时模拟 ≈ 1 个月真实）。M6 立项评估，M9 落地骨架。

### 6.3 与 CI 集成（Fuzz4DB 模式）

- PR 级 delta testing：只对变更代码定向生成（diff-cover 驱动），每 PR ≤ 15 分钟预算。
- Nightly：全量 oracle 组合 8 小时。
- Weekly：长程混沌（故障注入 + 并发）+ 性能基线。
- 报告门禁：确认级 bug 自动建 issue（带证据链）；观察级进看板。

---

## 7. Triage 与复现方案

### 7.1 自动复现管线

```text
差异事件 → 干净环境重放 → 二分版本(git bisect) → 语句级最小化(delta-debugging)
→ 计划 diff(新旧/跨模式) → 去重签名 → 证据链打包 → 人工队列
```

### 7.2 去重签名（多信号组合）

`hash(规范化AST骨架 | 栈帧 Top-k | 计划算子序列 | oracle 类型)`——四者至少两两相同才判重，防止误杀。

### 7.3 证据链 Schema（入库必填）

```json
{
  "id": "DBT-2026-xxxxx",
  "engine_commit": "abc123",
  "oracle": "TLP",
  "seed": 987654,
  "minimal_repro": ["..."],
  "expected_vs_actual": {"expected": "...", "actual": "...", "oracle_rationale": "..."},
  "plan_diff_url": "…",
  "coverage_delta": "…",
  "dedup_signature": "…",
  "environment": {"image": "…", "config": "…"},
  "reproduced": {"clean_env": true, "master": true, "times": 3},
  "status": "triage_pending"
}
```

### 7.4 Agent 在 triage 的角色

- 做：读日志/plan/栈 → 给出根因假设排序 → 建议相关源码位置（代码地图 RAG）→ 起草 issue。
- 不做：宣布 bug 成立（由 oracle + 复现决定）、改产品代码、关 issue。

---

## 8. Agent 层设计

### 8.1 角色分工

| Agent | 职责 | 模型档位 | 调用频率 |
|---|---|---|---|
| Generator-Evolver | 改生成器、写方言 adapter | 大模型 | 低频（特性级） |
| Oracle-Proposer | 提案新 oracle / 新等价变换 | 大模型 | 周级 |
| Triage | 根因分析、issue 草稿、去重判断 | 大模型 | 事件级 |
| Repro | 自动最小化编排、参数搜索 | 小模型 + 确定性算法 | 事件级 |
| Reviewer | 交叉质疑其他 Agent 的结论 | 不同模型/不同 prompt | 事件级 |
| 方言/文档助手 | 从 docs 提取语法规则 | 小模型（私有化） | 低频 |

> 关键：**Reviewer 用不同模型或至少不同上下文**，避免同一模型的系统性盲区自我确认。

### 8.2 工具与沙箱

- 工具白名单：SQL 客户端、EXPLAIN、git、代码搜索、日志读取、容器控制（受限）。
- 沙箱：独立容器/microVM、只读文件系统挂载源码、无生产网络、资源上限（CPU/内存/时长/token 预算）。
- 审计：所有工具调用落日志；所有产出带 witness；任何写操作可回滚。

### 8.3 成本预算（默认值，启动时校准）

| 项目 | 预算 |
|---|---|
| 大模型 token | ≤ 30 美元/日（Agent 事件级调用） |
| 小模型 | 私有化部署（单卡可跑 Qwen3-32B 级即可） |
| 执行算力 | 1000 容器·小时/日（约 40 台 16C 机器等价） |
| 每确认 bug 目标成本 | ≤ 50 美元 |

---

## 9. 评估基准（DB-Kernel-Test-Bench）

平台是否有效，用三条腿度量，**不采用"生成用例数/覆盖率"作为主指标**。

1. **历史 bug replay**：整理近 5 年 bug（含 CVE、issue、修复 commit），每个找"含 bug 的父 commit"；度量：T 小时内重发现比例。目标：M3 ≥ 30%，M12 ≥ 60%。
2. **Mutation 注入**：对优化器/执行器注入可控变异（删条件、翻比较符、错改基数估计路径等 50–100 个）；度量：变异杀死率。目标：M6 ≥ 60%。
3. **Held-out 特性**：每次新特性开发时留一个"不告诉平台"的特性；度量：自主覆盖时间。

每月出一次《平台有效性报告》，三条腿 + 北极星指标一起看。

---

## 10. 里程碑计划

### Phase 0（W1–W4）：基线与骨架

| 周 | 交付 | Exit criteria |
|---|---|---|
| W1 | 现状评估表（§1.1）填完；确定首发模块；搭好仓库/CI 骨架 | 评审通过 |
| W2 | SQLancer fork 跑通目标引擎；TLP/NoREC 上线 | 48h 长跑零语法错误 |
| W3 | expected-errors 收敛；自动最小化接好；第一张 bug 工单诞生 | 复现管线端到端通 |
| W4 | 基线度量（每周 bug、误报率、成本）；看板 v0 | 北极星基线写进本计划 §1.2 |

### Phase 1（W5–W12）：Oracle 组合 + 知识层

- W5–W6：CODDTest、执行模式差分、DQP 上线；方言 grammar 提取
- W7–W8：历史 bug 语料库（BugForge 模式：结构化 5 年 issue/commit/CVE）+ 代码地图
- W9–W10：Generator-Evolver（witness replay）上线；覆盖引导 A/B 开始
- W11–W12：数值语义 oracle、ERIQ；DB-Kernel-Test-Bench v1
- **Exit（M3）**：每周确认 bug ≥ 基线 ×1.5；误报率 ≤ 30%；历史 replay ≥ 30%

### Phase 2（W13–W24）：Agentic 闭环 + CI 化

- Triage Agent（根因/issue 草稿）上线；PR 级 delta testing 接 CI
- 数据合成（含 DBRepro 式元数据代理库）服务化
- Agent 治理：沙箱、配额、审计日志齐备
- **Exit（M6）**：time-to-repro -50%；确认 bug ≥ ×2；mutation 杀死率 ≥ 60%

### Phase 3（W25–W52）：存储/事务/分布式 + 自进化

- W25–W32：SIGKILL/磁盘故障矩阵；Elle 或自研历史检查；事务异常模式生成
- W33–W40：集中式 vs 分布式差分；DST 骨架（存储/复制模拟 IO）
- W41–W52：oracle 自进化管线全通；测试套件瘦身（set-cover）；跨引擎迁移 PoC
- **Exit（M12）**：北极星 ×3；误报率 ≤ 20%；历史 replay ≥ 60%；DST 复现 1 个真实分布式 bug

---

## 11. 人力与资源

| 角色 | 人数 | 职责 |
|---|---|---|
| 内核测试工程师（懂优化器/执行器） | 2 | oracle 实现、方言适配、误报复盘 |
| 内核开发工程师（懂存储/事务） | 1（M4 起） | 故障注入、DST、事务 checker |
| AI 工程师 | 1–2 | Agent、witness replay、RAG、成本治理 |
| SRE/平台（兼） | 0.5 | 容器、CI、看板 |

> 团队必须包含**至少 2 名读过被测引擎源码的人**——oracle 和 triage 的质量上限由内核理解力决定，不由模型决定。

---

## 12. 风险登记册

| # | 风险 | 概率 | 影响 | 应对 |
|---|---|---|---|---|
| R1 | 误报淹没，团队失去信任 | 中 | 高 | 三重复现门槛；误报率硬指标；每周复盘 |
| R2 | LLM 幻觉产出假 oracle/假特性 | 高 | 中 | witness replay；oracle 三关验证 |
| R3 | 回归测试固化错误行为 | 中 | 高 | mutation 校验；fault-revealing/enforcing 区分；生成测试进 review |
| R4 | 数据/代码泄露 | 低 | 高 | 私有模型；禁止生产数据；合成数据 |
| R5 | 成本失控 | 中 | 中 | 分层模型；日预算；每 bug 成本看板 |
| R6 | Agent 操作事故 | 低 | 高 | 沙箱、只读、配额、审计 |
| R7 | 覆盖率虚荣指标绑架方向 | 中 | 中 | 北极星=确认 bug/成本；覆盖率只作领先指标 |
| R8 | 引擎方言变化快于适配 | 中 | 中 | ParserFuzz 自动提取 + Generator-Evolver |
| R9 | 关键人流失 | 低 | 高 | 文档化；双人备份；知识入库 |

---

## 13. 启动第一周 Checklist

- [ ] 填写 §1.1 现状评估；确定首发模块（决策记录入库）
- [ ] 拉 SQLancer fork，`sqlite3 --oracle TLP` 本地跑通，熟悉 `--use-reducer`
- [ ] 建仓库骨架（§2.3）；CI 每日主干镜像 pipeline
- [ ] 选定指标采集方式（bug 工单打标：source=platform, oracle=?, status=confirmed/dup/fp）
- [ ] 梳理历史 bug 清单（近 5 年，导出 issue + 修复 commit 对应关系）
- [ ] 用 `/diverge-test-points` 对首发模块生成测试点清单，与现有用例做差距分析
- [ ] 确认 LLM 资源：私有模型可用性、大模型预算通道
- [ ] 排 W2–W4 的迭代计划与评审时间

---

## 附录 A：Oracle 实现速查

- **TLP**：谓词 p 拆为 p / NOT p / p IS NULL 三分区，三路查询并集应等于原查询。可推广到聚合（partition by）与窗口。
- **NoREC**：`SELECT ... WHERE p`（可优化）vs 强制逐行求值 p 的版本（不可优化），结果集应一致。
- **CODDTest**：会话内状态恒定 ⇒ 子查询/表达式的值可替换为字面量，替换前后结果一致。
- **DQP**：同一查询强制走 N 个不同计划（hint/开关/`pg_plan_advice`），结果一致。
- **执行模式差分**：线程数、向量化开关、外排开关、join 算法开关的全组合一致性（BuzzHouse）。
- **ERIQ**：同一中间结果用 VIEW / CTE / 临时表三种表示，结果一致。
- **数值语义**：浮点聚合以精确有理数为真值，按算法误差界分类 exact/bounded/indeterminate。
- **DQE**：SELECT/UPDATE/DELETE 同谓词操作的行集合一致（需追踪列）。
- **CERT（性能）**：更受限制询的估计行数不应更多；违例即基数估计问题。
- **集中式 vs 分布式**：同一 DBMS 两种部署模式结果一致（DistSQL）。
- **存储引擎轮换**：同一 DBMS 不同存储引擎，同一 SQL 结果一致（Thanos）。
- **事务历史检查（Elle）**：并发事务执行历史不得出现 G0/G1/G2 等异常。
- **崩溃一致性**：SIGKILL/断电后重启，已确认数据不丢、未确认数据不出现、校验全绿。

## 附录 B：Witness Replay 流程图

```text
Agent 提交: 代码 diff + witness(setup + 生成器选择路径 + 预期特性)
        │
        ▼
CI: checkout → build generator → replay witness
        │
   生成器产出含目标特性? ──No──→ 拒绝(记幻觉案例)
        │ Yes
        ▼
   引擎接受且解析树匹配? ──No──→ 拒绝
        │ Yes
        ▼
   跑 10 分钟冒烟(该特性 × 全部 oracle)
        │
   零误报? ──No──→ 打回修复
        │ Yes
        ▼
   合并 + 特性登记(谁引入/何时/覆盖哪些 oracle)
```

## 附录 C：关键参考（按主题）

**Oracle/逻辑 bug**：TLP(OOPSLA'20)、NoREC(FSE'20)、PQS(OSDI'20)、CERT(ICSE'24)、DQP(SIGMOD'24)、CODDTest(SIGMOD'25)、TQS(TODS'25)、集合论 join oracle(ICDE'26)、ERIQ(2026)、LLM oracle 发现(SIGMOD'26)、数值语义 oracle(2026)、EPSC(SIGMOD'26)

**生成器/Agent**：SQLancer++(ASPLOS'26)、ShQveL(SIGMOD'26)、Gungnir(2026)、ParserFuzz(2025)、DBcover(ICSE'26 工业)、MCTS(ICSE'26 工业)、DIRT(2026)、Fuzz4DB(ASE'26)

**分布式/事务/存储**：DistSQL(SIGMOD'26)、DAT(VLDB'27 TR)、Thanos(ICSE'25)、事务异常模式测试(2025)、OmniLink TLA+ 轨迹验证(2026)、Dinkel(VLDB'26)

**复现/数据**：DBRepro(ASE'26 工业)、BugForge(2026)、Test Data Generation(SIGMOD'26)

**工程实践**：SQLancer、SQLancer++、BuzzHouse(ClickHouse)、SQLite 测试体系、TigerBeetle VOPR、FoundationDB DST、Jepsen/Elle、Anthropic defending-code-reference-harness、Google Big Sleep(SQLite)、OpenAI Aardvark、DARPA AIxCC

**风险研究**：LLM 回归测试风险(2026)、工业 DBMS 的 LLM 单测评估(2026)

---

*本计划书中的所有论文结论与产业事件均以 2026-10-19 前公开可查的资料为准；落地实施时请以各项目最新版本为准。*
