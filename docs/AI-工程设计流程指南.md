# AI 工程设计流程指南

> 本文档说明如何组合使用 **OpenSpec + Cursor Skill + OpenClaw** 构建完整的 AI 辅助软件工程闭环。

---

## 工具集成现状说明

> ⚠️ **重要提示**：本文档描述的是 AI 工程化的**理想架构**和**设计思路**，部分集成方案需要自行实现。

| 工具 | 当前状态 | 说明 |
|------|----------|------|
| **OpenSpec** | 方法论/规范 | 规格驱动开发的文档规范，无需安装 |
| **Cursor Skill** | Cursor 原生支持 | 可在 `.cursor/skills/` 定义技能 |
| **OpenClaw** | 独立 Agent 平台 | 可执行 shell 命令、调用 API，但**不能直接控制 Cursor IDE** |

### 集成方式

**方式一：OpenClaw 作为外部触发器（推荐）**
```
OpenClaw 触发 → Shell 命令 / API 调用 → 构建、测试、部署
                    ↓
              Cursor 内手动执行 Skill 完成代码生成
```

**方式二：CLI 模式的 AI Agent（如 Claude CLI、Aider）**
```
OpenClaw 触发 → claude-cli / aider → 直接修改代码
```

**方式三：纯 Cursor 工作流（无需 OpenClaw）**
```
开发者在 Cursor 中 @skill-name → AI 执行对应 Skill → 生成代码/测试
```

> 💡 **本文档主要聚焦于设计流程和 Skill 定义**，OpenClaw 部分可视为"未来集成方向"或用其他自动化工具替代（如 GitHub Actions、n8n、Make 等）。

---

## 一、核心理念

### 1.1 为什么需要流程化？

| 问题 | 无流程时 | 有流程时 |
|------|----------|----------|
| AI 输出质量 | 不稳定，依赖提示词技巧 | 稳定，由规范约束 |
| 知识沉淀 | 散落在对话记录中 | 结构化存储，可复用 |
| 团队协作 | 每人各自使用 AI | 统一流程，结果可预期 |
| 质量保障 | 人工逐行审查 | 自动化测试 + 关键节点人工审核 |

### 1.2 三大核心工具

```
┌─────────────────────────────────────────────────────────────┐
│                    AI 工程化工具链                           │
│                                                             │
│  ┌───────────────┐                                         │
│  │   OpenSpec    │ ──▶ 定义「做什么」                       │
│  │   规格文档     │     需求、接口、行为规格                  │
│  └───────────────┘                                         │
│          │                                                  │
│          ▼                                                  │
│  ┌───────────────┐                                         │
│  │ Cursor Skill  │ ──▶ 定义「怎么做」                       │
│  │   能力定义     │     AI 在每个环节的专项能力               │
│  └───────────────┘                                         │
│          │                                                  │
│          ▼                                                  │
│  ┌───────────────┐                                         │
│  │   OpenClaw    │ ──▶ 实现「自动跑」                       │
│  │   执行引擎     │     跨平台触发、持续执行、结果通知         │
│  └───────────────┘                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、完整流程概览

### 2.1 端到端流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AI 工程设计完整流程                               │
│                                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │ Phase 1  │───▶│ Phase 2  │───▶│ Phase 3  │───▶│ Phase 4  │          │
│  │ 需求分析  │    │ 设计文档  │    │ 代码实现  │    │ 测试验证  │          │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘          │
│       │               │               │               │                 │
│       ▼               ▼               ▼               ▼                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │OpenSpec  │    │OpenSpec  │    │ Skill:   │    │ Skill:   │          │
│  │proposal  │    │design.md │    │code-gen  │    │test-gen  │          │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘          │
│       │               │               │               │                 │
│       └───────────────┴───────────────┴───────────────┘                 │
│                               │                                         │
│                               ▼                                         │
│                      ┌────────────────┐                                 │
│                      │    OpenClaw    │                                 │
│                      │  自动化执行引擎  │                                 │
│                      └────────────────┘                                 │
│                               │                                         │
│              ┌────────────────┼────────────────┐                        │
│              ▼                ▼                ▼                        │
│         Slack/Discord    本地构建        测试执行                        │
│           触发通知         代码生成        结果反馈                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 阶段概览

| 阶段 | 输入 | 工具 | 输出 | 审核点 |
|------|------|------|------|--------|
| **Phase 1: 需求分析** | 用户需求描述 | OpenSpec | `proposal.md` | 产品/业务确认 |
| **Phase 2: 设计文档** | proposal.md | OpenSpec + Skill | `design.md` | **架构师审核** |
| **Phase 3: 代码实现** | design.md | Skill + OpenClaw | 源代码 | 代码审查 |
| **Phase 4: 测试验证** | design.md + 代码 | Skill + OpenClaw | 测试报告 | 自动化验证 |

---

## 三、Phase 1：需求分析

### 3.1 目标

将模糊的需求描述转化为结构化的需求提案（proposal）。

### 3.2 OpenSpec 文件结构

```
project/
└── specs/
    └── feature-name/
        └── proposal.md      # 需求提案
```

### 3.3 proposal.md 模板

```markdown
# [功能名称] 需求提案

## 1. 背景
- 为什么需要这个功能？
- 解决什么问题？

## 2. 目标用户
- 谁会使用这个功能？

## 3. 功能描述
- 用户故事 / 使用场景
- 核心功能列表

## 4. 验收标准
- 如何判断功能完成？
- 可量化的成功指标

## 5. 约束条件
- 技术限制
- 时间限制
- 资源限制

## 6. 风险
- 已知风险及应对

## 审批
- [ ] 产品确认
- [ ] 技术可行性确认
```

### 3.4 Skill 定义：需求分析助手

**文件**：`.cursor/skills/requirement-analyst/SKILL.md`

```markdown
# Requirement Analyst Skill

## 触发条件
当用户请求分析需求或编写需求提案时激活

## 能力描述
根据用户的需求描述，生成结构化的 OpenSpec proposal 文档

## 输入要求
- 用户的需求描述（可以是模糊的）
- 项目上下文（可选）

## 输出规范
- 生成符合 OpenSpec 格式的 proposal.md
- 识别并列出不明确的点，提出澄清问题
- 提供初步的技术可行性评估

## 工作流程
1. 解析用户需求，提取关键信息
2. 识别需求中的模糊点，列出待澄清问题
3. 生成 proposal.md 草稿
4. 标注需要产品/业务确认的部分

## 约束
- 不做技术方案决策，只描述「做什么」
- 使用用户的业务语言，避免技术术语
```

---

## 四、Phase 2：设计文档

### 4.1 目标

将需求提案转化为可执行的技术设计文档，包含可测试的行为规格。

### 4.2 OpenSpec 文件结构

```
project/
└── specs/
    └── feature-name/
        ├── proposal.md      # 需求提案（Phase 1）
        └── design.md        # 技术设计（Phase 2）
```

### 4.3 design.md 模板

```markdown
# [功能名称] 技术设计文档

> **状态**：待审核 / 审核通过 / 已实现
> **关联需求**：proposal.md

## 1. 概述
- 技术方案概述
- 核心设计思路

## 2. 接口设计
### 2.1 公共接口
- 函数签名、参数、返回值
- 错误处理方式

### 2.2 数据结构
- 核心类/结构体定义

## 3. 行为规格（可测试）
### 3.1 正常场景
| 编号 | 场景 | 输入 | 预期输出 |
|------|------|------|----------|
| TC-01 | ... | ... | ... |

### 3.2 边界条件
| 编号 | 场景 | 输入 | 预期输出 |
|------|------|------|----------|
| TC-10 | ... | ... | ... |

### 3.3 错误处理
| 编号 | 场景 | 输入 | 预期输出 |
|------|------|------|----------|
| TC-20 | ... | ... | ... |

## 4. 技术选型
- 依赖库及版本
- 选型理由

## 5. 项目结构
- 文件/目录组织

## 6. 风险评估
- 技术风险及缓解措施

## 审核记录
| 日期 | 审核人 | 结果 | 备注 |
|------|--------|------|------|
| | | | |
```

### 4.4 Skill 定义：设计文档生成器

**文件**：`.cursor/skills/design-spec-writer/SKILL.md`

```markdown
# Design Spec Writer Skill

## 触发条件
当用户请求编写技术设计文档时激活

## 能力描述
根据需求提案（proposal.md）和项目上下文，生成符合 OpenSpec 格式的技术设计文档

## 输入要求
- proposal.md（需求提案）
- 项目规范文档（RULES.md）
- 相关模块接口（如有）

## 输出规范
- 生成符合 OpenSpec 格式的 design.md
- 接口定义必须完整（参数、返回值、异常）
- 行为规格必须可直接转化为测试用例
- 列出所有边界条件和错误处理

## 关键约束
1. **设计文档必须经架构师审核**
   - 参考 [Google-CPP-软件架构师角色定义.md] 的审核标准
   
2. **行为规格必须可测试**
   - 每条规格有明确的输入和预期输出
   - 覆盖正常场景、边界条件、错误处理

3. **技术选型有理有据**
   - 说明选择该方案的理由
   - 列出备选方案及其优劣

## 工作流程
1. 阅读 proposal.md，理解需求
2. 分析技术可行性，确定技术方案
3. 设计接口和数据结构
4. 编写行为规格（测试用例依据）
5. 评估风险，提出缓解措施
6. 生成完整的 design.md
```

### 4.5 架构师审核检查清单

> 参考 [Google-CPP-软件架构师角色定义.md](./Google-CPP-软件架构师角色定义.md)

| 审核维度 | 检查项 |
|----------|--------|
| **架构合理性** | 模块职责单一？依赖关系合理？无循环依赖？ |
| **接口设计** | 简洁清晰？易于使用和扩展？符合现有风格？ |
| **技术选型** | 方案合理？与现有技术栈一致？ |
| **性能考量** | 满足性能要求？无明显瓶颈？ |
| **可测试性** | 支持有效的测试策略？ |
| **安全性** | 无安全风险？输入验证充分？ |

---

## 五、Phase 3：代码实现

### 5.1 目标

根据审核通过的设计文档，生成符合规范的实现代码。

### 5.2 Skill 定义：代码生成器

**文件**：`.cursor/skills/code-generator/SKILL.md`

```markdown
# Code Generator Skill

## 触发条件
当用户请求根据设计文档生成代码时激活

## 前置条件
- design.md 已经过架构师审核并标记为「审核通过」
- 项目规范文档（RULES.md）可用

## 能力描述
根据设计文档的接口定义和行为规格，生成符合项目规范的实现代码

## 输入要求
- design.md（技术设计文档，状态为「审核通过」）
- RULES.md（项目编码规范）
- 相关依赖模块代码（如有）

## 输出规范
- 按 design.md 中定义的项目结构生成文件
- 实现所有公共接口
- 遵循 RULES.md 中的命名、格式、注释规范
- 包含必要的错误处理
- 添加 docstring/注释

## 关键约束
1. **严格遵循设计文档**
   - 不擅自添加设计中未定义的功能
   - 接口签名与设计文档完全一致

2. **遵循项目规范**
   - 命名风格、代码格式符合 RULES.md
   - 使用项目已有的公共模块，不重复造轮子
   - 使用项目的日志系统、错误处理机制

3. **代码质量**
   - 函数短小（20-40 行）
   - 单一职责
   - 有意义的命名

## 工作流程
1. 阅读 design.md，理解接口和行为规格
2. 阅读 RULES.md，了解编码规范
3. 按项目结构创建文件
4. 实现接口，确保符合行为规格
5. 添加必要的注释和文档
6. 自检代码风格是否符合规范
```

### 5.3 OpenClaw 自动化执行

配置 OpenClaw 在代码生成后自动触发：

```yaml
# openclaw-config.yaml
workflows:
  code-generation:
    trigger:
      - slack_message: "生成代码 @feature-name"
      - file_change: "specs/**/design.md"
    
    steps:
      - name: "检查设计文档状态"
        action: check_design_status
        params:
          required_status: "审核通过"
      
      - name: "调用代码生成 Skill"
        action: cursor_skill
        params:
          skill: "code-generator"
          input:
            design_doc: "specs/{{feature}}/design.md"
            rules: "RULES.md"
      
      - name: "代码格式化"
        action: shell
        command: "make format"
      
      - name: "静态检查"
        action: shell
        command: "make lint"
      
      - name: "通知结果"
        action: notify
        params:
          channel: slack
          message: "代码生成完成，请进行代码审查"
```

---

## 六、Phase 4：测试验证

### 6.1 目标

根据设计文档的行为规格生成测试用例，执行测试，验证实现是否符合设计。

### 6.2 Skill 定义：测试生成器

**文件**：`.cursor/skills/test-generator/SKILL.md`

```markdown
# Test Generator Skill

## 触发条件
当用户请求根据设计文档生成测试用例时激活

## 能力描述
根据 design.md 中的行为规格，生成可执行的测试代码

## 输入要求
- design.md（技术设计文档）
- 实现代码（用于了解实际接口）
- 项目测试规范（如有）

## 输出规范
- 为每条行为规格生成对应的测试用例
- 测试代码符合项目测试框架（pytest/unittest/gtest 等）
- 包含正常场景、边界条件、错误处理测试
- 测试用例命名清晰，反映测试意图

## 关键约束
1. **测试与设计一一对应**
   - 每条行为规格对应至少一个测试用例
   - 测试用例注释引用对应的规格编号（如 TC-01）

2. **测试独立性**
   - 测试用例相互独立，不依赖执行顺序
   - 测试数据自包含，不依赖外部状态

3. **可重复执行**
   - 多次执行结果一致
   - 清理测试产生的副作用

## 行为规格到测试用例映射示例

设计文档规格：
| TC-01 | 转换单页 PPTX | 1页PPTX | 1页PDF |

生成的测试代码：
def test_tc01_convert_single_page_pptx():
    """TC-01: 转换单页 PPTX 应生成 1 页 PDF"""
    # Arrange
    input_file = "fixtures/test_1page.pptx"
    
    # Act
    result = converter.convert(input_file, "output.pdf")
    
    # Assert
    assert result.exists()
    assert get_pdf_page_count(result) == 1
```

### 6.3 Skill 定义：设计验证器

**文件**：`.cursor/skills/design-validator/SKILL.md`

```markdown
# Design Validator Skill

## 触发条件
当测试执行完成后，需要验证设计是否正确时激活

## 能力描述
对比测试结果与设计文档中的行为规格，生成验证报告

## 输入要求
- design.md（技术设计文档）
- 测试执行结果（测试报告）
- 测试覆盖率报告（可选）

## 输出规范
- 生成验证报告（validation-report.md）
- 列出通过/失败的规格
- 分析失败原因，区分代码问题 vs 设计问题
- 提出修改建议

## 验证报告模板

# [功能名称] 设计验证报告

## 验证概要
- 总规格数：X
- 通过：Y
- 失败：Z
- 通过率：Y/X

## 详细结果

### 通过的规格
| 编号 | 场景 | 状态 |
|------|------|------|
| TC-01 | ... | ✅ 通过 |

### 失败的规格
| 编号 | 场景 | 状态 | 失败原因 | 建议 |
|------|------|------|----------|------|
| TC-05 | ... | ❌ 失败 | 返回值类型不匹配 | 修改代码 |
| TC-10 | ... | ❌ 失败 | 设计未考虑此场景 | 修改设计 |

## 结论
- [ ] 设计验证通过，可提交审查
- [ ] 需修改代码后重新验证
- [ ] 需修改设计后重新实现

## 修改建议
1. ...
2. ...
```

### 6.4 OpenClaw 自动化测试流程

```yaml
# openclaw-config.yaml
workflows:
  test-verification:
    trigger:
      - slack_message: "运行测试 @feature-name"
      - file_change: "src/**/*.py"
    
    steps:
      - name: "生成测试用例"
        action: cursor_skill
        params:
          skill: "test-generator"
          input:
            design_doc: "specs/{{feature}}/design.md"
      
      - name: "执行测试"
        action: shell
        command: "pytest tests/ -v --tb=short"
        capture_output: true
      
      - name: "生成验证报告"
        action: cursor_skill
        params:
          skill: "design-validator"
          input:
            design_doc: "specs/{{feature}}/design.md"
            test_result: "{{previous_output}}"
      
      - name: "判断结果"
        action: conditional
        condition: "test_passed"
        on_success:
          - action: notify
            params:
              message: "✅ 测试通过，设计验证成功"
        on_failure:
          - action: notify
            params:
              message: "❌ 测试失败，请查看验证报告"
          - action: cursor_skill
            params:
              skill: "code-generator"
              input:
                mode: "fix"
                error_report: "{{validation_report}}"
```

---

## 七、完整工作流示例

### 7.1 场景：开发 PPTX 转 PDF 工具

```
┌─────────────────────────────────────────────────────────────────────────┐
│ 开发者在 Slack 发送：「开发一个 PPTX 转 PDF 的命令行工具」               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ OpenClaw 触发 Phase 1                                                   │
│ → 调用 requirement-analyst Skill                                        │
│ → 生成 specs/pptx-to-pdf/proposal.md                                   │
│ → Slack 通知：「需求提案已生成，请确认」                                  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                          开发者确认需求 ✓
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ OpenClaw 触发 Phase 2                                                   │
│ → 调用 design-spec-writer Skill                                         │
│ → 生成 specs/pptx-to-pdf/design.md                                     │
│ → Slack 通知：「设计文档已生成，请架构师审核」                            │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                          架构师审核通过 ✓
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ OpenClaw 触发 Phase 3                                                   │
│ → 调用 code-generator Skill                                             │
│ → 生成 pptx2pdf/ 目录下的源代码                                         │
│ → 执行 make format && make lint                                         │
│ → Slack 通知：「代码已生成，请审查」                                      │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│ OpenClaw 触发 Phase 4                                                   │
│ → 调用 test-generator Skill                                             │
│ → 生成 tests/ 目录下的测试代码                                          │
│ → 执行 pytest tests/                                                    │
│ → 调用 design-validator Skill                                           │
│ → 生成 validation-report.md                                             │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
            测试全部通过 ✅                    测试失败 ❌
                    │                               │
                    ▼                               ▼
        Slack：「设计验证通过」          AI 分析失败原因并自动修复
        准备提交代码审查                        │
                                               ▼
                                    重新执行 Phase 3/4
                                    直到测试通过
```

### 7.2 项目目录结构

```
my-project/
├── .cursor/
│   └── skills/                    # Cursor Skills 定义
│       ├── requirement-analyst/
│       │   └── SKILL.md
│       ├── design-spec-writer/
│       │   └── SKILL.md
│       ├── code-generator/
│       │   └── SKILL.md
│       ├── test-generator/
│       │   └── SKILL.md
│       └── design-validator/
│           └── SKILL.md
│
├── specs/                         # OpenSpec 规格文档
│   └── pptx-to-pdf/
│       ├── proposal.md            # Phase 1 输出
│       ├── design.md              # Phase 2 输出
│       └── validation-report.md   # Phase 4 输出
│
├── src/                           # Phase 3 生成的源代码
│   └── pptx2pdf/
│       ├── __init__.py
│       ├── converter.py
│       └── cli.py
│
├── tests/                         # Phase 4 生成的测试代码
│   └── test_converter.py
│
├── openclaw-config.yaml           # OpenClaw 工作流配置
├── RULES.md                       # 项目编码规范
└── README.md
```

---

## 八、最佳实践总结

### 8.1 关键成功因素

| 因素 | 说明 |
|------|------|
| **设计文档质量** | 行为规格必须可测试，是整个闭环的基础 |
| **架构师审核** | 设计阶段拦截问题，避免返工 |
| **Skill 约束明确** | 限制 AI 的自由发挥，确保输出可预期 |
| **自动化执行** | OpenClaw 减少人工介入，加速迭代 |

### 8.2 常见问题及应对

| 问题 | 原因 | 应对 |
|------|------|------|
| AI 生成的代码不符合规范 | Skill 约束不够明确 | 完善 RULES.md，在 Skill 中强调关键约束 |
| 测试用例覆盖不全 | 设计文档行为规格不完整 | 在设计阶段就列出完整的边界条件 |
| 闭环执行失败 | OpenClaw 配置错误 | 分步调试，确保每个 step 独立可运行 |
| 架构师审核瓶颈 | 所有设计都需要人工审核 | 建立审核优先级，简单功能快速通过 |

### 8.3 渐进式采用建议

```
Level 1（入门）：
└── 只用 OpenSpec 写设计文档
    └── 人工生成代码和测试

Level 2（进阶）：
└── OpenSpec + Cursor Skill
    └── AI 辅助生成代码和测试
    └── 人工触发和审核

Level 3（成熟）：
└── OpenSpec + Skill + OpenClaw
    └── 全流程自动化
    └── 人工只在关键节点审核
```

---

## 九、参考资源

- [OpenSpec 项目](https://github.com/Fission-AI/OpenSpec) - 规格驱动开发框架
- [OpenClaw](https://openclaw.ai) - 开源 Agent 执行平台
- [Cursor Skills 文档](https://docs.cursor.com/skills) - Cursor 技能定义指南
- [Google-CPP-软件架构师角色定义.md](./Google-CPP-软件架构师角色定义.md) - 架构师审核标准
- [AI-编程最佳实践指南.md](./AI-编程最佳实践指南.md) - 编码规范和审查清单
