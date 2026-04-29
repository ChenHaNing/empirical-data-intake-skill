# Empirical Data Intake Skill · 实证研究数据入场 Skill

> A structured pre-analysis triage layer for empirical research, providing a 5-slot conditional Q&A interface, deterministic 80% mechanical cleaning, and a verified machine-readable data contract that routes the cleaned dataset to the correct downstream estimation pipeline.
>
> 本 skill 在原始数据与下游全流程估计 pipeline 之间提供一层结构化的预分析分诊（pre-analysis triage），通过 5 槽位条件式问答（5-slot conditional Q&A）、确定性的 80% 机械清洗，以及一份机器可读的数据合同，将清洗后的数据集精确路由到匹配的下游 pipeline。

[![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-v0.2-blue.svg)](#版本与-roadmap)
[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-skill-orange.svg)](https://claude.com/claude-code)
[![Discipline: econ + epi](https://img.shields.io/badge/discipline-econ%20%2B%20epi-green.svg)](#mode-a公共卫生--流行病学专项)

Repository: <https://github.com/ChenHaNing/empirical-data-intake-skill>

---

## 1. 摘要 (Overview)

本 skill 解决实证研究中一个长期被低估的问题：**从原始数据文件到 analysis-ready 数据集的过渡环节，目前缺乏一个结构化、可复现、跨语言的标准操作流程**。

现有的全流程实证 pipeline（如 StatsPAI、Stata `reghdfe`-based、R `fixest`-based、Python `pyfixest`-based）默认其输入已经是 analysis-ready 数据集，将"如何回答研究设计相关问题"和"如何识别清洗优先级"两件事完全留给研究者。本 skill 在此处插入一层**决策层**：通过对原始数据的静态检查（static inspection），结合学科感知的条件式问答（conditional questioning），自动完成约 80% 的机械清洗，并将其余 20% 的设计性决策以 `unresolved_decisions` 字段显式上交至下游 pipeline。

设计原则：
- **描述而非决断**：暴露数据缺陷与待决决策，不替研究者做研究设计判断
- **可复现性优先**：所有操作不联网、不抓取外部数据，保证 same input 永远 same output
- **学科双轨**：默认轨适用于经济学 / 计量 / 金融 / 政治学等社科研究；Mode A 适用于公共卫生 / 流行病学 / 临床研究
- **边界清晰**：与下游 4 个 flagship pipeline 严格 80/20 分工，不重复其能力

---

## 2. 设计动机 (Motivation)

在审计现有 4 个 flagship pipeline 的 Step 1 (Data Cleaning) 内容（参见 `audit-flagship-cleaning.md`）后，识别出三个共性缺口：

1. **缺乏学科感知的 Q&A 决策层** —— 4 个 pipeline 的 reference 文档均为参考手册式（reference manual），描述"如何做"但不指导"何时做、做不做"
2. **缺乏数据驱动的自适应** —— 4 个 pipeline 不会先 inspect 数据再决定提问内容
3. **公共卫生 / 流行病学专项内容缺位** —— 4 个 pipeline 的 description 均声明"Mode A 复用 Step 1 清洗"，但实际 reference 文档均未覆盖 epi 专属内容（index date 对齐、time-zero 选择、删失与缺失区分、person-time 构造、washout 期、immortal time bias 检测、ICD/ATC/CPT 代码标准化）

本 skill 精准填补这三处空白，且不与现有 pipeline 重复任何已实现能力。

---

## 3. 在实证研究全流程中的定位 (Pipeline Position)

```
原始数据文件 (.csv / .dta / .xlsx / .sav / .parquet / .sas7bdat)
        |
        v
+-----------------------------------------------+
|  empirical-data-intake-skill (本仓库)          |
|  - 5-slot conditional Q&A                     |
|  - 80% 自动机械清洗                            |
|  - Mode A epi 专项 (条件触发)                  |
|  - 4 件套合同输出                              |
+-----------------------------------------------+
        |
        v
干净数据集 (.dta / .parquet / .rds + 永久 .xlsx 视察版)
+ data_contract.yaml (机器可读)
+ routing_recommendation.md (人类可读)
        |
        v
+-----------------------------------------------+
|  下游全流程 pipeline (任选其一)                 |
|  - 00 StatsPAI (一站式 Python)                 |
|  - 00.1 Python (pandas + pyfixest 等)          |
|  - 00.2 Stata (reghdfe / csdid / ivreg2)       |
|  - 00.3 R (tidyverse + fixest)                 |
+-----------------------------------------------+
```

本仓库是**实证研究全流程 skill 体系**的第一个模块。后续模块（基线回归、稳健性、异质性、制表绘图等）将以独立仓库形式发布，并由总入口仓库（umbrella）统一索引。

---

## 4. 安装与调用 (Installation & Invocation)

### 4.1 安装

```bash
cd ~/.claude/skills/
git clone https://github.com/ChenHaNing/empirical-data-intake-skill.git
```

重启 Claude Code 即可识别该 skill。无须手动注册。

### 4.2 触发条件

下列任意一项满足即可激活本 skill：

- 用户描述刚收到原始数据，询问从何处开始（如"从哪开始""怎么处理这个数据"）
- 用户不确定 4 个 flagship pipeline 中应使用哪一个
- 用户开展公共卫生 / 流行病学研究（Mode A 触发）
- 用户提及 "原始数据""数据清洗""data intake""data triage""raw data""cohort 数据""index date""流行病学数据"等关键词
- 数据存在明显缺陷（缺失、重复、类型混乱）但用户未明确诊断

不应激活本 skill 的情况：

- 数据已经是 analysis-ready，研究者只想运行回归 → 直接调用 flagship
- 研究者询问单一具体清洗操作（如"如何 winsorize""如何运行 MICE"）→ 由 flagship reference 直接处理
- 研究者已确信数据干净，仅需描述性统计 → 由 flagship Step 3 处理

---

## 5. 标准操作流程 (Standard Operating Procedure, SOP)

本节描述 skill 在任意原始数据上的端到端执行流程。流程为**确定性**——同一输入始终产生同一输出（不联网、不依赖外部状态）。

### 5.1 流程总览

```
+---------+    +-----------+    +-------------+    +--------+    +----------+    +----------+    +--------+    +---------+
| Phase 0 | -> |  Phase 1  | -> |  Phase 2    | -> | Phase 3| -> | Phase 4  | -> | Phase 5  | -> |Phase 6 | -> | Phase 7 |
| 触发    |    | Skill加载 |    | 静态检查    |    |槽位预  |    | 用户问答 |    | 自动清洗 |    |Mode A  |    | 输出 +  |
|         |    |           |    | (silent)    |    |计算    |    | (5-slot) |    | (80%)    |    |(条件)  |    | 路由    |
+---------+    +-----------+    +-------------+    +--------+    +----------+    +----------+    +--------+    +---------+
```

### 5.2 Phase 0 — 触发 (Activation)

| 字段 | 值 |
|---|---|
| 输入 | 用户自然语言请求 或 一个数据文件路径 |
| 操作者 | 用户 |
| 输出 | Claude Code 完成 skill 描述匹配 |
| 退出条件 | 描述命中 → 进入 Phase 1；否则不激活 |

### 5.3 Phase 1 — Skill 加载 (Skill Loading)

| 字段 | 值 |
|---|---|
| 输入 | Phase 0 命中 |
| 操作 | Claude Code 加载 SKILL.md 至上下文 |
| 输出 | Skill 进入待运行状态 |

### 5.4 Phase 2 — 静态检查 (Static Inspection, Silent)

不向用户暴露中间计算过程。

| 字段 | 值 |
|---|---|
| 输入 | 数据文件路径 |
| 操作 | 1. 由扩展名识别格式（.csv / .dta / .xlsx / .sav / .sas7bdat / .parquet / .tsv）<br>2. 用对应 reader 加载（多 sheet 时打印 warning 并默认第一张）<br>3. 计算 12 维 inspection 字典 |
| 12 维 inspection 字段 | n_rows, n_cols, dtypes, missing_rate, n_unique, single_pkey_candidates, composite_pkey, candidate_id_cols, candidate_time_cols, binary_01_cols, binary_text_cols, epi_signal_cols, string_cols |
| 输出 | inspection 字典（驻留工作内存）|
| 失败模式 | 文件不存在 / 格式不支持 / 行数为零 / 列数为零 → 即停 |

### 5.5 Phase 3 — 槽位模式预计算 (Slot Mode Pre-computation)

对 5 个槽位分别根据 inspection 结果计算 AUTO / CONFIRM / ASK 三种模式之一。

| 槽位 | 名称 | 默认模式 | 决定模式的信号 |
|---|---|---|---|
| Slot 1 | 学科 | 始终 ASK | 不可由数据推断 |
| Slot 2 | 研究设计 | 多数 CONFIRM | 检测到 (id, time) 复合主键 → 面板；无时间列 → 截面；含 index_date 类列 → 队列 |
| Slot 3 | 观测单位 | CONFIRM 或 ASK | 由 candidate_id_cols 命名启发推断 |
| Slot 4 | 焦点变量 | ASK（措辞条件依赖 Slot 1, 2） | 由 binary_01_cols / candidate_outcome 命名预填候选 |
| Slot 5 | 软件目标 | `.dta` AUTO；其他 ASK | 由文件扩展名推断 |

### 5.6 Phase 4 — 用户问答循环 (User Q&A Loop)

按 Slot 1 → 5 顺序遍历，每个槽位单独一轮（**禁止批量提问**）。

```
for slot in [1, 2, 3, 4, 5]:
    mode = pre_computed[slot]
    if mode == AUTO:
        continue
    elif mode == CONFIRM:
        ask_yes_no_with_default()
    elif mode == ASK:
        ask_multiple_choice()
    record_answer()
```

熟练用户被问 2–3 题；新手用户最多 5–6 题。

### 5.7 Phase 5 — 自动机械清洗 (Auto Cleaning, 80%)

按下列顺序执行 9 个子步骤，每步印一行 `[intake]` 日志。

| 步骤 | 操作 | 是否可能修改行数 |
|---|---|---|
| 5.7.1 | 列名规范化（ASCII snake_case；Stata 目标时强制 ≤32 char + 非保留字） | 否 |
| 5.7.2 | 字符串列首尾空白剥离 | 否 |
| 5.7.3 | 明确情形的 dtype 强转（数值字符串 → 数值；ISO 日期字符串 → datetime） | 否 |
| 5.7.4 | 主键唯一性硬校验（违反则即停，不写合同） | 否 |
| 5.7.5 | 面板结构推断（n_units, n_periods, coverage, gaps, balanced） | 否 |
| 5.7.6 | 缺失率清单（按列汇总；零缺失列省略） | 否 |
| 5.7.7 | 焦点变量缺失的 MCAR hint（Welch t-test on 协变量；scipy 缺失时降级 numpy） | 否 |
| 5.7.8 | Outlier 标记（z-score |z|>4 与 IQR 1.5×；**仅 flag 不 winsorize**） | 否 |
| 5.7.9 | sample_log 初始化为 `[("raw", n_rows)]` | 否 |

**关键不变量**：v0.2 中此阶段不删除任何行（除非主键违规即停）；任何行级筛除均归属 Phase 6 或 flagship。

### 5.8 Phase 6 — Mode A 增强 (Mode A Augmentation, Conditional)

触发条件：`Slot 1 == "epi"`。

| 步骤 | 操作 | 失败时行为 |
|---|---|---|
| 6.1 | 验证 index date（非空、可解析、范围合理） | raise，不写合同 |
| 6.2 | 检测 time-zero 对齐与 immortal time bias（事件早于 t0 → 排除并 sample_log）| 删除非法行 + 日志 |
| 6.3 | 验证生存结构 `(event, event_date, censor_date)`，构造统一 `(follow_time, status)` | raise |
| 6.4 | 应用 washout 期（用户在 Slot 4 声明时） | 删除前导用户 + 日志 |
| 6.5 | ICD-10 / CPT / ATC 编码标准化（去点号、大写化、INVALID 前缀标记） | 不删除，仅标记 |

详细规则与代码模板见 `references/01-mode-a-epi-patterns.md`。

### 5.9 Phase 7 — 输出生成 (Output Generation)

写入 `<parent-of-data>/intake/` 目录（即数据文件父目录的 `intake/` 子目录）。

| 文件 | 内容 |
|---|---|
| `cleaned_dataset.{dta\|parquet\|rds}` | 清洗后数据集，格式匹配 Slot 5 |
| `cleaned_dataset.xlsx` | 7-sheet 视察工作簿（**永远写出**，与 Slot 5 无关）：cleaned_data、outlier_flags、rename_map、missing_inventory、outlier_summary、unresolved、contract_summary |
| `data_contract.yaml` | 机器可读合同（schema 详见 SKILL.md "Output artifacts"）|
| `routing_recommendation.md` | 人类可读下一步指引，含可粘贴的下游 pipeline 代码 |

### 5.10 Phase 8 — 路由交接 (Routing Handoff)

| 字段 | 值 |
|---|---|
| 输出 | 一行明确的下一步消息（例："Now invoke flagship 00.X (mode: Y) — see intake/routing_recommendation.md"） |
| 状态 | Skill 退出，下游 flagship pipeline 接手 |

---

## 6. 决策机制：5-slot Conditional Q&A

每个槽位的运行模式根据 inspection 结果动态决定：

| 模式 | 含义 | 用户体验 |
|---|---|---|
| AUTO | 数据已能直接回答 | 该槽位不向用户暴露 |
| CONFIRM | 数据强暗示某一答案 | 一次 yes/no 确认 |
| ASK | 数据无法回答 | 一道多选题 |

| Slot | 内容 | 不可由数据推断的部分 |
|---|---|---|
| 1 | 学科：经济社科默认轨 / 公共卫生流病 Mode A | 全部 |
| 2 | 研究设计：截面 / 面板 / 时序 / 队列+index date / 重复截面 / 描述性 / 探索 | 当数据信号弱时 |
| 3 | 观测单位（每行代表谁） | 当 id 命名启发失败时 |
| 4 | 焦点变量：outcome / treatment（econ）或 outcome / exposure（epi） | 永远不可推断（措辞条件依赖 Slot 1, 2） |
| 5 | 软件目标：Python / StatsPAI / Stata / R | 仅 `.dta` 可由扩展名 AUTO |

Slot 4 的措辞与候选根据 Slot 1 + 2 的答案动态变化（详见 SKILL.md 中"Slot 4 — Focal variables (CONDITIONAL on Slot 2 + 1)"小节的全分支表）。

---

## 7. 输出规范 (Output Specification)

### 7.1 cleaned_dataset.{dta|parquet|rds}

- 所有原始列（重命名为 ASCII snake_case，Stata 目标时强制满足列名约束）
- 自动强转后的 dtype
- Outlier flag 列（每个连续数值变量产出 `*_outlier_z4` 和 `*_outlier_iqr` 两列，int8）
- 焦点变量的缺失 flag 列（如 `{focal}_missing`）
- Mode A 时附 `index_date`、`follow_time`、`status` 等已验证字段

### 7.2 cleaned_dataset.xlsx (always written)

7 张工作表：cleaned_data、outlier_flags、rename_map、missing_inventory、outlier_summary、unresolved、contract_summary。该文件**与 Slot 5 选择无关，永远写出**，用于研究者在交付下游之前进行视觉核对。

### 7.3 data_contract.yaml

机器可读的"数据合同"，跨语言泛化自 StatsPAI 的 `data_contract()` 概念。完整 schema 见 SKILL.md。关键不变量：

- `key_uniqueness` 必须等于 1.0（否则 Skill 拒绝写出）
- `n_rows_raw - n_rows_clean` 必须等于 sample_log 中所有 drop 条目之和
- `discipline == "epi"` 时 `epi_checks` 段必须存在
- `software_target == "stata"` 时 `renames_applied.values()` 中所有列名必须 ASCII 且 ≤32 字符
- `missing_pattern` 仅列出非零缺失的列

### 7.4 routing_recommendation.md

人类可读的下一步指引文件，包含：

- 推荐的 flagship pipeline 与 mode（含理由）
- 已由 intake 完成的步骤清单（避免下游重做）
- 下游 pipeline 应执行的步骤（按编号给出可粘贴代码）
- 待研究者决策的开放问题（unresolved_decisions 的人类可读复述）
- sample_log 全文

---

## 8. Mode A：公共卫生 / 流行病学专项

当 Slot 1 = epi 时自动激活的 7 项检查：

| 检查 | 防范的方法学错误 |
|---|---|
| Index date 解析与硬校验 | t0 缺失或不可解析导致所有时间计算错误 |
| Time-zero 对齐 | Immortal time bias |
| Censoring 与 missing 区分 | 把删失误当数据缺失，错误施加多重插补 |
| Person-time 长格式校验 | (id, start, end) 区间重叠或断裂导致 Cox / KM 风险集错乱 |
| Washout 期实施 | 既往用户污染 |
| ICD-10 / CPT / ATC 标准化 | 同一诊断的多种编码形态导致频次错误 |
| Multiple cohort entries | 隐式破坏"id 唯一"假设 |

详细实现与代码模板见 `references/01-mode-a-epi-patterns.md`。

---

## 9. 与 Flagship Pipeline 的边界协议 (Boundary Contract)

| 由 Intake 完成（约 80%） | 由 Flagship 完成（约 20%） |
|---|---|
| 列名规范化 | 多重插补（MICE / `mi` / `mice`）|
| 明确情形的 dtype 强转 | Winsorize / 截尾 |
| 空白与编码清理 | Heckman 选择 / IPW for MNAR |
| 重复检测 | 主分析数据外的辅助 merge |
| 主键唯一性校验 | Event-study 时间对齐 |
| 面板结构推断 | Within-group outlier 检测 |
| 缺失率清单 | 调查权重的回归级应用 |
| Outlier 标记（仅 flag）| Winsorize / 截尾决策 |
| Sample log 初始化 | Sample log 后续追加 |
| Mode A epi 7 项专属检查 | 全部下游估计 |

**核心契约**：Intake 描述并暴露缺失，**不替研究者做研究设计判断**。研究设计相关的所有决策（缺失插补方法、winsorize 阈值、样本范围限制、index date 的具体定义等）打包为 `unresolved_decisions` 字段，交由 flagship pipeline + 研究者共同决定。

---

## 10. 不变量 (Invariants)

每次执行均成立的契约：

1. 原始数据文件全程只读
2. 所有输出写至 `<parent-of-data>/intake/`，从不修改 `data/`
3. 写入合同前永远先校验；硬错误立即停止，不出 contract
4. Sample log 自洽：`n_rows_raw - n_rows_clean` 必须等于 sample_log 后续 drop 项之和
5. Stata 目标列名守护：ASCII / ≤32 字符 / 非保留字，**绝不静默截断**
6. MCAR hint 永远计算（scipy 缺失时降级 numpy 实现）
7. Excel 视察工作簿永远写出（与 Slot 5 选择无关）
8. routing_recommendation.md 永远附可粘贴的下游 pipeline 代码

---

## 11. Prerequisites

**必需依赖**（任一缺失即停）：
- `pandas >= 2.0`
- `numpy`
- `pyyaml`
- `openpyxl`

**条件可选依赖**：
- `pyreadstat` —— 仅当源文件为 `.sav` (SPSS) 或 `.sas7bdat` (SAS)
- `scipy` —— 仅用于 MCAR Welch t-test；缺失时自动降级为 numpy 大样本近似
- `pyarrow` —— 仅当 Slot 5 = Python 且需 `.parquet` 原生输出

---

## 12. 文件结构

```
empirical-data-intake-skill/
├── SKILL.md                            主指令文件（Claude 读取）
├── README.md                           本文件
├── audit-flagship-cleaning.md          设计前对 4 个 flagship pipeline 的 Step 1 审计报告
└── references/
    └── 01-mode-a-epi-patterns.md       Mode A 公共卫生 / 流行病学专章
```

---

## 13. 版本与 Roadmap

### v0.2 (2026-04-29) — 当前版本

经真实数据测试后的稳定版。修复 v0.1 的 5 个 critical bug 与 6 个文档不一致问题，新增以下能力：

- 永远写出的 `cleaned_dataset.xlsx` 7-sheet 视察工作簿
- 复合主键检测（v0.1 误将连续数值变量识别为单列主键）
- pandas 2.x 字符串 dtype 兼容（v0.1 用 `dtype == object` 漏检）
- 文本二值列单独标记（如汉字两值分类）
- xlsx 多 sheet 警告
- Stata 列名硬约束（32 字符 + 保留字）
- 输出目录约定 `<parent-of-data>/intake/`
- scipy fallback 文档化

### v0.1 (2026-04-29) — 初版

5-slot conditional Q&A 框架、80% 自动清洗、Mode A epi 7 项检查、3 件套输出。

### v0.3 — 计划中

- `references/02-qa-decision-graph.md` —— 完整 Q&A 决策图（当前嵌于 SKILL.md）
- `references/03-data-contract-schema.md` —— YAML 字段字典
- `references/04-routing-rules.md` —— 完整路由规则表
- 调查权重的 intake 阶段标记（NHANES / CHARLS / HRS）
- 行业 / 地域代码的格式校验提示（NAICS / ISIC / FIPS / GB/T 4754）

---

## 14. 贡献 (Contributing)

欢迎 issue 与 pull request。**贡献前请先阅读 `audit-flagship-cleaning.md`**，理解本 skill 与下游 4 个 flagship pipeline 的边界。任何会让 intake 越界做研究设计判断的提案将被拒绝（具体边界见 SKILL.md 中"What intake does NOT do"小节）。

提交 PR 时请：
- 在描述中说明该改动属于 `decision layer / mechanical execution / output schema / Mode A` 中的哪一类
- 若涉及 schema 变更，附 contract YAML 的 before/after 对比
- 若新增依赖，说明是否为必需依赖或条件可选

---

## 15. License

[CC BY-SA 4.0](LICENSE)。允许商业使用、修改、分发，但需署名并以相同协议分发衍生作品。

---

## 16. 引用 (Citation)

学术使用时，请在脚注或 acknowledgments 中说明：

> Data preparation was assisted by the Empirical Data Intake Skill (v0.2, 2026), a Claude Code skill providing structured pre-analysis triage for empirical research. The skill produced a `data_contract.yaml` documenting the verified panel structure, missingness pattern, and outlier flags of the analysis dataset, together with a sample log recording all sample-construction steps. All cleaning decisions and downstream estimation were performed by the human author.

BibTeX：

```bibtex
@misc{empirical_data_intake_skill_2026,
  title  = {Empirical Data Intake Skill: Pre-Analysis Triage for Empirical Research},
  author = {{Chen, Haning}},
  year   = {2026},
  note   = {v0.2, Claude Code skill},
  url    = {https://github.com/ChenHaNing/empirical-data-intake-skill}
}
```

---

## 17. 致谢 (Acknowledgments)

- 数据合同的 YAML 概念跨语言泛化自 [`brycewang-stanford/StatsPAI`](https://github.com/brycewang-stanford/StatsPAI) 的 `data_contract()` 实现
- 8 步数据清洗范式（rename / dtype / missing / outlier / dedup / merge / panel / labels）参考自 [`Awesome-Agent-Skills-for-Empirical-Research`](https://github.com/brycewang-stanford/Awesome-Agent-Skills-for-Empirical-Research) 中 4 个 flagship pipeline 的 `references/01-data-cleaning.md`
- 设计前的 flagship 审计方法论参考实证研究中 pre-registration 的契约式工作流
