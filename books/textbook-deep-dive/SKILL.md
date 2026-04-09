---
name: textbook-deep-dive
description: 将教材章节转换为全面的深度学习材料，包括知识图谱、证明和详细解释
license: MIT
metadata:
  author: Claude Code
  version: 1.0.0
  category: education
  tags: [textbook, deep-dive, learning, academic]
args:
  book_directory:
    type: string
    description: "书籍目录路径，包含各章节的 md 文件"
    required: true
  output_directory:
    type: string
    description: "输出目录路径，默认为书籍目录下的 deep-dive/ 子目录"
    required: false
    default: null
  chapters:
    type: string
    description: "要处理的章节范围，如 '1-5' 或 'all'，默认 'all'"
    required: false
    default: "all"
---

# 教科书深度解析 SKILL

## 工作流程

### 第一阶段：扫描章节

读取当前 skill 所在目录下的 `references/agent-prompts/chapter-scanner.md` 文件，调用 Agent 扫描书籍目录。

- name: "扫描章节"
  agent:
    model: "claude-opus-4-6"
    prompt: |
      {{read "references/agent-prompts/chapter-scanner.md"}}

      ## 当前任务
      请扫描以下书籍目录，识别所有章节文件：
      - 书籍目录: {{book_directory}}
      - 处理范围: {{chapters}}

      使用 Bash 工具列出目录内容，然后解析所有章节文件信息。
      返回符合指定格式的 JSON 结果。

### 第二阶段：创建输出目录

如果 `output_directory` 未指定，使用默认路径：`{{book_directory}}/deep-dive/`

使用 Bash 工具创建目录结构：
- `{{output_directory}}/`
- `{{output_directory}}/章节/` - 存放各章节的详细分析

### 第三阶段：并行处理章节（关键优化）

**重要：为提高效率，所有章节必须采用并行处理策略。**

**并行处理原则：**
- 章节之间相互独立，可以并行分析
- 使用多个 Agent 同时处理不同章节
- 每个 Agent 独立完成一个小节分析任务

#### 3.1 读取所有章节原始内容

使用 Read 工具并行读取所有章节 markdown 文件内容。

#### 3.2 并行分析章节小节

读取 `references/agent-prompts/section-analyzer.md` 和 `references/section-template.md`，**并行调用多个 Agent 同时分析不同章节**：

```yaml
# 为每个章节创建一个独立的 Agent 任务，然后并行执行
agents:
  - name: "分析-章节-{{chapter.num}}"
    model: "claude-opus-4-6"
    prompt: |
      {{read "references/agent-prompts/section-analyzer.md"}}

      ## 当前任务
      请分析以下章节内容，识别所有小节并生成详细分析。

      ### 章节信息
      - 章节编号: {{chapter.num}}
      - 章节标题: {{chapter.title}}

      ### 小节模板
      {{read "references/section-template.md"}}

      ### 章节原始内容
      ```markdown
      {{chapter.content}}
      ```

      请识别所有小节，为每个小节生成完整的 Deep Dive 分析。
      返回符合模板格式的完整 markdown 内容。

# 并行执行所有 Agent
parallel: true
```

**注意：** 每个章节的分析是独立的，多个章节的 Agent 可以同时运行。

#### 3.3 并行生成章节概览

读取 `references/chapter-overview-template.md`，**并行为所有章节生成学习概览**：

```yaml
agents:
  - name: "概览-章节-{{chapter.num}}"
    model: "claude-opus-4-6"
    prompt: |
      你是一个专业的学习材料设计专家。请基于章节内容生成学习概览。

      ### 概览模板
      {{read "references/chapter-overview-template.md"}}

      ### 章节信息
      - 章节编号: {{chapter.num}}
      - 章节标题: {{chapter.title}}

      ### 已分析的小节
      {{chapter.sections_list}}

      请根据模板生成完整的学习概览，包括学习目标、难度警告、节依赖图、定理清单和学习建议。

parallel: true
```

#### 3.4 并行生成章节总结

读取 `references/agent-prompts/chapter-summarizer.md` 和 `references/chapter-summary-template.md`，**并行为所有章节生成总结**：

```yaml
agents:
  - name: "总结-章节-{{chapter.num}}"
    model: "claude-opus-4-6"
    prompt: |
      {{read "references/agent-prompts/chapter-summarizer.md"}}

      ### 总结模板
      {{read "references/chapter-summary-template.md"}}

      ### 章节信息
      - 章节编号: {{chapter.num}}
      - 章节标题: {{chapter.title}}

      ### 小节分析内容
      {{chapter.sections_content}}

      请生成完整的章节总结 markdown 内容。

parallel: true
```

#### 3.5 写入章节文件

将所有并行生成的内容写入输出目录：

1. `{{output_directory}}/章节/第{{chapter.num}}章_{{chapter.title}}/`
2. `00_概览.md` - 章节学习概览
3. `{{section.num}}_{{section.title}}.md` - 各小节详细分析
4. `99_总结.md` - 章节总结与回顾

#### 3.6 内容质量检查（并行执行）

**关键步骤：所有章节处理完成后，必须并行调用 Agent 检查生成文件的质量：**

```yaml
agents:
  - name: "检查-章节-{{chapter.num}}"
    model: "claude-opus-4-6"
    prompt: |
      {{read "references/agent-prompts/content-checker.md"}}

      ## 检查目标
      章节: 第{{chapter.num}}章_{{chapter.title}}
      文件路径: {{output_directory}}/章节/第{{chapter.num}}章_{{chapter.title}}/

      ## 检查任务
      1. 使用 Read 工具读取所有生成的文件
      2. 按照检查清单逐项验证
      3. 输出详细的检查结果

parallel: true
```

**检查清单（Content Checker 必须执行）：**

1. **文件结构完整性**
   - [ ] 概览文件 (00_概览.md) 存在且内容完整
   - [ ] 所有小节文件存在且命名规范
   - [ ] 总结文件 (99_总结.md) 存在且内容完整

2. **数学公式格式**
   - [ ] 行内公式使用 `$...$` 格式
   - [ ] 独立公式使用 `$$...$$` 格式
   - [ ] LaTeX 语法正确，能正确渲染

3. **模板符合度**
   - [ ] 包含所有必需的章节（背景、概念、定理、示例等）
   - [ ] 内容深度达标（每小节不少于 2000 字）
   - [ ] 包含至少一个具体的数值示例

4. **图表语法**
   - [ ] Mermaid 图表语法正确
   - [ ] 知识图谱清晰展示概念关系
   - [ ] 定理依赖图正确反映逻辑关系

5. **术语一致性**
   - [ ] 核心术语在全章中保持一致
   - [ ] 中英文术语对照准确
   - [ ] 符号定义前后统一

6. **内部链接**
   - [ ] 章节内引用正确
   - [ ] 跨章节引用（如有）有效

**检查结果处理：**
- `passed`: 质量检查通过，继续下一步
- `warning`: 存在小问题，记录警告但继续
- `failed`: 存在严重问题，需要修复后重新检查

**修复流程：**
```yaml
- name: "修复质量问题"
  condition: "check.status == 'failed'"
  agent:
    model: "claude-opus-4-6"
    prompt: |
      根据质量检查报告修复以下问题：
      {{check.issues}}

      请修复后重新生成受影响的文件。
```

### 第四阶段：生成书籍总览

所有章节处理并检查通过后，读取 `references/agent-prompts/book-indexer.md` 和 `references/book-overview-template.md`，调用 Agent 生成书籍总览：

- name: "生成书籍总览"
  agent:
    model: "claude-opus-4-6"
    prompt: |
      {{read "references/agent-prompts/book-indexer.md"}}

      ### 总览模板
      {{read "references/book-overview-template.md"}}

      ### 书籍信息
      - 书籍目录: {{book_directory}}
      - 书籍名称: {{book_title}}

      ### 章节摘要
      {{chapters_summary}}

      请生成完整的书籍总览和索引。

写入文件：
- `{{output_directory}}/00_书籍总览.md`
- `{{output_directory}}/索引.md`

### 第五阶段：最终质量验证

**最终检查：对生成的完整输出进行最终验证**

```yaml
- name: "最终质量验证"
  agent:
    model: "claude-opus-4-6"
    prompt: |
      请对以下完整输出进行最终质量验证：
      - 输出目录: {{output_directory}}

      ## 验证清单

      ### 全局结构
      - [ ] 书籍总览文件存在且内容完整
      - [ ] 导航文件存在且所有链接正确
      - [ ] 索引文件存在且格式规范
      - [ ] 章节目录结构正确

      ### 章节一致性
      - [ ] 各章节编号连续无遗漏
      - [ ] 章节间引用关系正确
      - [ ] 术语在全书中保持一致

      ### 格式统一性
      - [ ] 所有文件使用统一的标题层级
      - [ ] 表格格式统一
      - [ ] 代码块和公式样式一致

      ## 输出
      返回验证报告，包含：
      1. 验证结果（通过/失败）
      2. 发现的问题列表
      3. 修复建议
```

### 第六阶段：生成导航索引

创建一个导航文件，方便在章节间跳转：

```markdown
# 深度学习材料导航

## 书籍信息
- **书名**: {{book_title}}
- **处理时间**: {{timestamp}}
- **章节数**: {{total_chapters}}
- **质量状态**: {{quality_status}}

## 章节列表
{{chapter_nav_list}}

## 快速链接
- [书籍总览](00_书籍总览.md)
- [完整索引](索引.md)
- [质量报告](质量报告.md)

## 生成统计
- 总文件数: {{total_files}}
- 总字符数: {{total_chars}}
- 公式数量: {{formula_count}}
- 图表数量: {{diagram_count}}
```

写入：`{{output_directory}}/导航.md`

## 输出结构

最终输出目录结构：

```
{{output_directory}}/
├── 00_书籍总览.md
├── 导航.md
├── 索引.md
├── 质量报告.md
└── 章节/
    ├── 第1章_XXX/
    │   ├── 00_概览.md
    │   ├── 1.1_XXX.md
    │   ├── 1.2_XXX.md
    │   └── 99_总结.md
    ├── 第2章_XXX/
    │   └── ...
    └── ...
```

## 性能优化策略

### 并行度控制
- 默认同时运行最多 **5-10 个** Agent（根据模型限制调整）
- 如果章节数量超过并行度，分批处理

### 错误恢复
- 单个章节处理失败不影响其他章节
- 记录失败的章节，最后统一报告
- 支持对失败章节单独重试

## 注意事项

1. **并行安全**: 各 Agent 只写入自己的章节目录，避免文件冲突
2. **资源控制**: 监控 API 调用频率，避免触发限制
3. **文件命名**: 章节目录和文件使用中文命名，保持与原始文件一致
4. **公式格式**: 所有数学公式使用 LaTeX 格式
5. **图片处理**: 保留原始内容中的图片引用，确保相对路径正确
6. **增量处理**: 如果输出目录已存在，询问用户是否覆盖或跳过已处理章节
7. **质量优先**: 宁可少处理几个章节，也要确保生成内容的质量
