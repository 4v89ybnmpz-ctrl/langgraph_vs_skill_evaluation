# 开发者体验评估方案对比：Cogito LangGraph 原生编排 vs Claude Skill 复刻编排

> **版本**: v1.0 | **日期**: 2026-08-14
> **背景**: 本报告对比 OSS-Compass Cogito 的两种开发者体验(DevX)评估执行方式——① 项目原生的 LangGraph 编排引擎(`devx_main.py`);② 本次新实现的 Claude Code Skill 复刻编排(`devx-orchestration`)。
> **数据来源**:
> - 方案① 实测报告:`/root/cogito_run_0804_no_toolkit_master_ml_google_baidu/reports/Report-cann_graph-autofusion_20260813_1432.{md,json}`(LangGraph 原生运行)
> - 方案② 实测报告:`reports/Report-cann_ops-math_20260814_0405.{md,json}` + `logs/json/log_cann_ops-math_20260814_0405.json`(Skill 复刻运行)

---

## 一、两种方案的定义

| 维度 | 方案① LangGraph 原生编排 | 方案② Skill 复刻编排 |
|------|------------------------|---------------------|
| **定位** | 项目自带的自动化评估引擎 | 用 Claude Code Skill 复刻同一套编排流程 |
| **编排大脑** | Python LangGraph 显式状态机(Planner/Executor/Evaluator/Reporter 四节点 + 条件路由) | Claude 按 `SKILL.md` 指令扮演四节点 + 路由决策 |
| **执行手脚** | MCP 工具层:Playwright 真浏览器 / 持久 Shell 会话 / 真 Git 客户端 | `docker exec` 在昇腾 NPU 容器内执行 shell/git/编译/测试 |
| **报告生成** | `schema.py`(13 dataclass) + `sdx_metrics.py`(4256 行公式) + `report_template.md`(Jinja2) **代码确定性生成** | Claude 按 `schema.py` 结构 **手写**(LLM 估算) |
| **部署依赖** | 完整源码 + `uv sync` + LLM Key + 镜像 | 一个 `SKILL.md` + docker(零项目源码依赖) |

---

## 二、架构机制对比

### 2.1 方案① LangGraph 原生

```
devx_main.py (CLI 入口)
  └─ LangGraph StateGraph (显式状态机)
      ├─ planner   → 生成 evaluation_plan + pending_tasks(六阶段任务队列)
      ├─ executor  → 内含 advisor(LLM 决策),循环调用工具直到任务 success/failure
      ├─ tools     → MCP 工具:web_search / browser_navigate / shell_exec / git_clone ...
      ├─ evaluator → 逐任务提炼 observations/pain_points + 规则评分(0-100) + 推进阶段
      └─ reporter  → 渲染 Jinja2 模板生成 MD + schema.py 导出 JSON
```

关键特性:
- **显式状态机**:节点只读 state、返回增量,LangGraph 框架合并,流程天然可回放、可断点续跑
- **真环境执行**:Playwright 真浏览器(自动截图)、`PersistentShellSession` 持久会话(env/cwd 跨命令延续)
- **裁判系统(Judge Governance)**:`judge_pre`(事前拦截)/`judge`(事后归因)/`pain_review`(痛点质控)/`pain_repro`(确定性复现)四接入点,确保痛点来自项目而非 Agent 失误
- **上下文保护**:单次 prompt 上限 200K 字符,超限滑动窗口缩减历史重试

### 2.2 方案② Skill 复刻

```
Claude (编排大脑,按 SKILL.md 执行)
  ├─ 阶段0 planner   → 解析项目 + 生成 pending_tasks
  ├─ 阶段1-6 executor → 规划工具调用 → docker exec 执行 → 判读任务成败
  ├─ 阶段间 evaluator → 提炼痛点 + 估算评分(0-100) + 推进阶段
  └─ 阶段7 reporter   → 按 schema 手写 13 键 JSON + 六阶段 MD + log JSON
```

关键特性:
- **编排逻辑复刻**:六阶段旅程、四节点职责、路由决策表、工具循环、早退机制均按项目源码语义复刻
- **docker 隔离执行**:任务实际操作(克隆/装环境/编译/测试/跑示例)在昇腾 NPU 容器内真实执行
- **自愈能力**:遇到问题由 Claude 自主诊断、修复、重跑(本次实测 7 处)
- **报告三件套**:`Report-*.md` + `Report-*.json`(13 键) + `logs/json/log_*.json` 结构对齐项目

---

## 三、实测产出对比(核心数据)

### 3.1 结构与指标富集度

| 维度 | 方案① LangGraph(graph-autofusion) | 方案② Skill(ops-math) | 差异 |
|------|------|------|------|
| JSON 顶层键 | 13 键 | 13 键 | ✅ 完全一致 |
| `journey_steps` 数量 | 6 | 6 | ✅ 一致 |
| **SDX 指标实例数** | **40 个** | **12 个** | ❌ 差 3.3 倍 |
| SDX 唯一指标数 | 29 个 | 12 个 | ❌ |
| **E2E 指标** | **3 个**(总耗时/成功率/Token) | 1 个(达成率) | ❌ |
| recommendations | 8 条 | 3 条 | 部分差距 |
| 评分精度 | 63.2(小数) | 66(整数) | ❌ |
| 任务达成率 | 80.0%(精确计算) | 100.0%(估算) | ❌ |

> 注:两报告不是同一项目(graph-autofusion vs ops-math),但反映的是**同一维度上的产出能力差异**,与项目无关。

### 3.2 SDX 指标清单对比

**方案① 覆盖的 29 个唯一指标**(代码 4256 行公式计算):

| 阶段 | LangGraph 覆盖的 SDX 指标 |
|------|--------------------------|
| S0 发现 | `SDX_SEARCH_ROUNDS`、`SDX_SEARCH_TIME_SEC`、`SDX_SEARCH_DIRECT_HIT_RATE`、`SDX_DOC_JUMPS`、`SDX_DOC_SELF_CONTAINED_RATIO`、`SDX_DOC_DEAD_LINKS`、`SDX_NAMING_CONFUSION_COUNT`、`SDX_PLATFORM_MIGRATION_FRICTION` |
| S1 准备 | `SDX_ENV_INIT_TIME_SEC`、`SDX_ENV_PERSISTENCE`、`SDX_ENV_HARDWARE_COST_USD`、`SDX_ENV_STEP_COUNT`、`SDX_ENV_CLOUD_SESSION_LIMIT_SEC`、`SDX_CLONE_SUCCESS_RATE`、`SDX_CLONE_TIME_SEC`、`SDX_DEPENDENCY_TRANSPARENCY` |
| S2 快速体验 | `SDX_BUILD_TIME_SEC`、`SDX_BUILD_PARAM_COUNT`、`SDX_BUILD_ERROR_DIAGNOSABILITY`、`SDX_BUILD_FIRST_TRY_SUCCESS`、`SDX_QUICKSTART_RUN_SUCCESS` |
| S3 开发 | `SDX_BUILD_*`(同上)、`SDX_DEPLOY_VERIFY_AVAILABLE`、`SDX_DEPLOY_IDEMPOTENT`、`SDX_RUN_MODE_COMBINATIONS`、`SDX_RUN_OUTPUT_READABILITY` |
| S4 测试 | `SDX_RUN_OUTPUT_READABILITY`、`SDX_RUN_MODE_COMBINATIONS`、`SDX_TEST_SUITE_PASS_RATE`、`SDX_TEST_EXECUTION_TIME_SEC`、`SDX_TEST_COVERAGE_AVAILABLE` |
| 端到端 | `SDX_E2E_TOTAL_TIME_SEC`、`SDX_E2E_SUCCESS_RATE`、`SDX_E2E_TOKEN_USAGE`、`SDX_TASK_ACHIEVEMENT_RATE` |

**方案② 实际覆盖的 12 个指标**(Claude 估算):
`SDX_SEARCH_ROUNDS`、`SDX_SEARCH_DIRECT_HIT_RATE`、`SDX_ENV_INIT_TIME_SEC`、`SDX_CLONE_SUCCESS_RATE`、`SDX_DEPENDENCY_TRANSPARENCY`、`SDX_BUILD_TIME_SEC`、`SDX_BUILD_FIRST_TRY_SUCCESS`、`SDX_QUICKSTART_RUN_SUCCESS`、`SDX_RUN_OUTPUT_READABILITY`、`SDX_TEST_SUITE_PASS_RATE`、`SDX_TEST_EXECUTION_TIME_SEC`、`SDX_TASK_ACHIEVEMENT_RATE`

> 方案② 缺的 17 个指标多是"隐性/间接"指标:`SDX_DOC_JUMPS`(文档跳转数)、`SDX_DOC_DEAD_LINKS`(死链)、`SDX_ENV_HARDWARE_COST_USD`(硬件成本)、`SDX_ENV_PERSISTENCE`(环境持久化)、`SDX_BUILD_PARAM_COUNT`(编译参数数)、`SDX_BUILD_ERROR_DIAGNOSABILITY`(错误可诊断性)等——这些需要从工具调用历史中做**结构化统计**,Claude 手写时易遗漏。

### 3.3 E2E 端到端指标对比

| E2E 指标 | 方案① | 方案② |
|----------|------|------|
| `SDX_E2E_TOTAL_TIME_SEC`(总耗时) | ✅ 532.2s | ❌ 无 |
| `SDX_E2E_SUCCESS_RATE`(成功率) | ✅ 80% | ❌ 无 |
| `SDX_E2E_TOKEN_USAGE`(Token 消耗) | ✅ 5,377,059 | ❌ 无 |

> 方案② 无法自动统计 Token 消耗(Claude 不暴露自身 token 计数给报告),也无法精确计时——这是 LLM 手写报告的**结构性盲区**。

---

## 四、优缺点逐维度对比

### 4.1 报告确定性

| 项 | 方案① | 方案② |
|----|------|------|
| JSON/MD 结构 | ✅ 100% 逐字段固定(dataclass + Jinja2) | ⚠️ 键能对齐,但指标数量/精度近似 |
| 评分精度 | ✅ 小数(63.2) | ⚠️ 整数估算(66) |
| 可复现性 | ✅ 同输入同输出 | ❌ LLM 每次可能不同 |
| 可回放 | ✅ 显式状态机断点续跑 | ❌ 无 |

### 4.2 指标可信度

| 项 | 方案① | 方案② |
|----|------|------|
| SDX 数值来源 | ✅ 4256 行确定性公式 | ❌ LLM 估算 |
| 指标覆盖 | ✅ 29 唯一 / 40 实例 | ⚠️ 12 个核心 |
| E2E 成本指标 | ✅ Token/耗时/成功率 | ❌ 缺失 |
| 横向可比性 | ✅ 多项目精确对比 | ❌ 估算不可横向比 |

### 4.3 执行能力

| 项 | 方案① | 方案② |
|----|------|------|
| 真浏览器 | ✅ Playwright + 截图 | ⚠️ 用 Claude WebSearch/WebFetch 替代 |
| 真持久 Shell | ✅ PersistentShellSession | ✅ docker exec(持久容器) |
| 真 Git/Issue | ✅ GitMCPClient | ✅ docker exec / gh CLI |
| **自愈能力** | ❌ 卡住即失败,如实记 early_exit | ✅ **自主诊断+修复+重跑** |

### 4.4 质控体系

| 项 | 方案① | 方案② |
|----|------|------|
| 裁判系统 | ✅ judge_pre/judge/pain_review/pain_repro | ❌ 无(靠 Claude 自觉归因) |
| 痛点归因 | ✅ PROJECT/AGENT/ENVIRONMENT 硬分类 | ⚠️ 人工判断 |
| 反爬拦截检测 | ✅ anti_crawler 结构化标识 | ⚠️ 靠 Claude 判断 |

### 4.5 部署与维护

| 项 | 方案① | 方案② |
|----|------|------|
| 部署复杂度 | ❌ 源码 + uv sync + 镜像 + 多依赖 | ✅ 一个 SKILL.md + docker |
| 维护成本 | ❌ reporter 3000+ 行 + sdx 4256 行 | ✅ 改 SKILL.md 即可 |
| 规模化 | ✅ 服务化集群(API + Celery + ES) | ⚠️ 每次需 Claude 会话 |
| 成本 | ✅ Token 可量化 | ⚠️ 上下文窗口约束 |

---

## 五、本次实测的关键实证(skill 自愈能力)

方案② 在评估 ops-math 时,**自主发现并修复了 7 处障碍**才跑通 S0–S4,这 7 处恰恰是方案①(LangGraph)会在第一处就 `early_exit` 记"失败 0 分"的地方:

| # | 障碍 | 根因 | Skill 的修复 |
|---|------|------|-------------|
| 1 | `install_deps.sh` 非交互环境 exit=1 | `read -p` 读 EOF 触发 `set -e` | `yes \|` 喂 Y |
| 2 | 编译 wheel 报 `No module named pip` | 镜像 `/app/.venv` 缺 pip | `ensurepip --upgrade` |
| 3 | `-Werror=array-bounds` 编译失败 | GCC12 对 memcpy 假阳性 | `dependencies.cmake` 加 `-Wno-error` |
| 4 | bisheng `cstdint not found` | clang 找不到 libstdc++ 路径 | 设 `CPLUS_INCLUDE_PATH` |
| 5 | AICPU `is_copy_assignable` 崩 | GCC12 + 旧 Eigen 兼容 bug | `--noaicpu` 跳过 |
| 6 | `--run_example` 打印 help | 漏 op_type/mode 位置参数 | 补 `add_example eager` |
| 7 | 运行示例 `EZ1013` 找不到 kernel | 覆盖 `ASCEND_OPP_PATH` + abs 910C not supported | 不覆盖 opp 路径 |

> **这 7 处是"新手按文档会遇到的真实阻碍",其发现过程本身就是最有价值的体验信号**。方案② 能把"踩坑-修复"的完整挣扎记录下来;方案① 则如实记录为"编译失败",但不展示修复路径——**两者反映的"体验真相"不同维度**。

---

## 六、适用场景建议

| 场景 | 推荐方案 | 理由 |
|------|---------|------|
| **探索性踩坑发现**(找项目真实的体验痛点) | ② Skill | 自愈能力强,能把踩坑过程完整记录,不会半途而废 |
| **定量的规模化复测**(多项目横向对比、打分) | ① LangGraph | 指标精确、可复现、有裁判质控 |
| **生产化持续评估**(每日定时跑批量项目) | ① LangGraph | 服务化集群、确定性、Token 可量化 |
| **快速单项目体检**(无源码环境、快速上手) | ② Skill | 零依赖、部署极简、灵活变通 |
| **报告结构严格一致**(对接下游系统/数据库) | ① LangGraph | 逐字段固定、schema 确定性 |
| **发现新痛点后再精确定量** | ①+② 组合 | 先用 ② 探路找痛点,再用 ① 精确复测 |

---

## 七、结论

两种方案**不是替代关系,而是互补关系**:

- **方案① LangGraph 是"确定性测量仪"**:指标精确(40 个 SDX + 3 个 E2E,小数精度)、结构逐字段固定、可回放可规模化、有裁判质控;但脆弱——遇坑即停,无自愈,部署维护成本高。

- **方案② Skill 是"带判断力的探路者"**:零源码依赖、部署极简、自愈能力强(本次实测 7 处自主修复)、能把踩坑挣扎完整记录;但指标是估算、不可复现、缺 E2E 成本指标和裁判质控。

**最理想的组合用法**:先用 Skill(方案②)对项目做探索性评估,挖掘真实体验痛点;再用 LangGraph(方案①)对已发现的问题做精确、可横向对比的定量复测。前者负责"发现问题",后者负责"精确测量"。

---

## 附:报告结构一致性说明

两种方案产出的 **JSON 顶层结构完全一致(13 键)**:`meta` / `project` / `targets` / `overall_scores` / `e2e_metrics` / `journey_steps` / `journey_map` / `phase_analysis` / `recommendations` / `divergence_alerts` / `key_insight` / `doc_analysis` / `anti_crawler`。

差异在于**数值的富集度与精度**,而非结构骨架——即方案② 能"对齐结构",但无法"对齐指标精确性"。这也是方案② 报告必须标注"评分为 LLM 估算口径"的原因。
