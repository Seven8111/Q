# Agent Compliance

基于多 Agent 协作的法律合规多层审查系统。通过 4 个专业化 Agent 和 1 个规则引擎，实现从合同文本到合规报告、风险矩阵、修订建议的全自动化审查流水线。

## 核心设计

### Pipeline 流程

```
合同文档
   │
   ▼
[ParserMatcher] ──→ 结构化条款 + 法规匹配结果
   │
   ▼
[ConsistencyChecker] ──→ 跨条款一致性检查（规则引擎，非 LLM）
   │
   ▼
[RiskAssessor] ──→ 风险矩阵 + 综合评估
   │
   ▼
[ComplianceReviewer] ──→ 审查报告 + 路由决策
   │
   ├── pass ────────────────→ [Advisor] ──→ 修订建议 ──→ 交付
   ├── fix_and_retry ───────→ 回到 ParserMatcher（不消耗回退次数）
   └── escalate_to_human ───→ 标记人工介入，输出当前最佳结果
```

### 四个 Agent + 一个规则引擎

| 组件 | 职责 | 类型 |
|------|------|------|
| **ParserMatcher** | 条款解析 + 法规匹配（合并执行） | LLM Agent |
| **ConsistencyChecker** | 跨条款一致性检查（金额、期限、义务对等性） | 规则引擎 |
| **RiskAssessor** | 综合风险评估 + 风险矩阵生成 | LLM Agent |
| **ComplianceReviewer** | 质量门控 + 路由决策 + 人工升级判断 | LLM Agent |
| **Advisor** | 修订建议 + 替代条款文本 + 风险缓解方案 | LLM Agent |

### 关键设计决策

**ParserMatcher 合并设计**：条款解析和法规匹配合并为一个 Agent，因为两者在实践中耦合——识别条款类型需要法规知识，匹配法规需要理解条款结构。

**规则引擎处理跨条款检查**：金额一致性、期限合理性、义务对等性等需要精确比较的任务用规则引擎而非 LLM，因为 LLM 在长上下文中做数值比较不可靠。

**三种路由决策**：`pass`（通过）、`fix_and_retry`（修复重试，不消耗回退配额）、`escalate_to_human`（人工介入，法律场景特有的退出路径）。

**版本化状态**：每次操作追加新版本，回退是切换版本指针，不删除历史。

**回退上限 2 次**：超过 2 次说明问题可能出在文档本身或法规覆盖不足，应升级到人工。

## 快速开始

### 环境要求

- Python 3.10+
- OpenAI 兼容的 API

### 安装

```bash
git clone https://github.com/your-username/multi-agent-compliance.git
cd multi-agent-compliance
pip install -r requirements.txt
```

### 配置

```bash
cp .env.example .env
# 编辑 .env 填入 API 配置
```

### 运行

**使用内置示例合同：**

```bash
python main.py
```

**审查自定义合同：**

```bash
python main.py --contract my_contract.txt
```

**指定合同类型和法规范围：**

```bash
python main.py \
  --contract labour_contract.txt \
  --contract-type labour \
  --regulations "《劳动法》" "《劳动合同法》" "《社会保险法》"
```

**使用本地模型：**

```bash
python main.py \
  --model qwen2.5:72b \
  --base-url http://localhost:11434/v1 \
  --contract contract.txt
```

### 参数说明

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--contract`, `-c` | - | 合同文件路径 |
| `--text`, `-t` | - | 直接输入合同文本 |
| `--contract-type` | 自动识别 | 合同类型（sales/labour/service/lease/technology） |
| `--regulations` | 民法典+劳动法+消保法 | 重点关注的法规列表 |
| `--model` | `gpt-4o` | LLM 模型名称 |
| `--base-url` | - | API 地址 |
| `--max-rollbacks` | `2` | 最大回退次数 |
| `--output`, `-o` | `output` | 输出目录 |
| `--verbose`, `-v` | - | 详细日志 |
| `--resume` | - | 从保存的状态恢复 |

## 输出结构

```
output/
├── state.json              # 完整版本化状态（可恢复）
├── review_report.md        # 合规审查报告
├── risk_matrix.json        # 风险矩阵
├── compliance_results.json # 逐条款合规评估
├── clauses.json            # 结构化条款
├── amendments.json         # 修订建议
└── escalations.json        # 需人工介入的事项（如有）
```

## 项目结构

```
multi-agent-compliance/
├── main.py                     # 入口文件
├── requirements.txt
├── .env.example
├── .gitignore
├── README.md
├── examples/
│   └── example_contract.txt    # 内置示例合同（技术服务合同）
└── src/
    ├── __init__.py
    ├── agents/
    │   ├── __init__.py
    │   ├── base.py
    │   ├── parser_matcher.py       # 条款解析 + 法规匹配
    │   ├── risk_assessor.py        # 风险评估
    │   ├── compliance_reviewer.py  # 合规审查 + 路由决策
    │   └── advisor.py              # 修订建议
    ├── scheduler/
    │   ├── __init__.py
    │   └── scheduler.py            # 规则调度器
    ├── state/
    │   ├── __init__.py
    │   └── contract_state.py       # 版本化状态管理
    ├── rules/
    │   ├── __init__.py
    │   └── consistency.py          # 跨条款一致性规则引擎
    └── llm/
        ├── __init__.py
        └── client.py               # LLM 客户端封装
```

## 规则引擎检查项

`ConsistencyChecker` 执行以下确定性检查：

- **金额一致性**：付款金额、违约金比例、定金金额之间的合理性
- **期限逻辑**：保修期是否超过合同期、付款期限是否合理
- **义务对等性**：甲乙方义务数量是否严重不对等
- **兜底条款**：识别"包括但不限于"等赋予过大裁量权的表述
- **术语冲突**：检测"排他"与"非排他"等根本性矛盾

## 与代码架构系统的区别

| 维度 | CodeArch（代码架构） | Compliance（法律合规） |
|------|---------------------|----------------------|
| Agent 数量 | 4 | 4 + 规则引擎 |
| 回退上限 | 2 次 | 2 次 |
| 人工介入 | 可选 | 必须（escalate_to_human） |
| 外部知识 | 无 | 法规知识库 |
| 可溯源性 | 次要 | 核心（必须引用法条） |
| 确定性检查 | 无 | 规则引擎做跨条款检查 |
| 输出验证 | 测试通过 | 人工最终确认 |

## 扩展方向

- **RAG 法规知识库**：将法规全文向量化，自动检索最相关条文
- **OCR 预处理**：支持扫描件 PDF 和图片格式合同
- **多法域支持**：扩展到不同国家和地区的法规体系
- **增量审查**：法规更新后自动重新审查已有合同
- **判例参考**：引入相关司法判例辅助灰色地带判定
- **人工审核工作流**：集成审批流程，支持律师在线审核和批注

## License


