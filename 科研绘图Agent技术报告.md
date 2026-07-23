# 非生成式AI科研绘图Agent：开源项目技术报告

> 面向初学者的深度技术分析 | 2026-07-21
>
> 本报告覆盖 9 个开源项目/论文，按技术路线分为四大类：Draw.io XML 渲染、代码渲染（matplotlib/SVG）、GUI 操控、混合路线。每个项目均从架构、绘图原理、渲染管线、技术难点、对初学者的启示五个维度展开分析。

---

## 目录

- [第一部分：技术路线总览](#第一部分技术路线总览)
- [第二部分：Draw.io XML 渲染路线](#第二部分drawio-xml-渲染路线)
  - [项目1：next-ai-draw-io](#项目1next-ai-draw-io)
  - [项目2：drawio-mcp（官方）](#项目2drawio-mcp官方)
  - [项目3：Pegasus](#项目3pegasus)
- [第三部分：代码渲染路线](#第三部分代码渲染路线)
  - [项目4：scipilot-figure-skill](#项目4scipilot-figure-skill)
  - [项目5：NaturePanelForge](#项目5naturepanelforge)
  - [项目6：LLM4SVG](#项目6llm4svg)
- [第四部分：GUI 操控路线](#第四部分gui-操控路线)
  - [项目7：VisPainter](#项目7vispainter)
- [第五部分：混合路线](#第五部分混合路线)
  - [项目8：Paper2Any](#项目8paper2any)
  - [项目9：AutoFigure](#项目9autofigure)
- [第六部分：横向对比](#第六部分横向对比)
- [第七部分：初学者构建指南](#第七部分初学者构建指南)

---

## 第一部分：技术路线总览

在开始逐个分析之前，你需要先理解一个核心问题：**"不用生成式图像AI来绘图"到底有哪些可能的技术路径？**

目前开源社区探索出了四条主要路线：

### 路线一：生成 Draw.io XML → 渲染

LLM 输出 draw.io 的 XML 描述文件，由 draw.io 引擎渲染成矢量图。draw.io XML 本质上是一种**图形描述语言**——它不描述像素，而是描述"在坐标(100,200)处放一个宽120高60的圆角矩形，里面写'Frontend'"。

**核心优势**：输出天然可编辑（任何元素都可以单独修改）、矢量无损、draw.io 生态成熟。

**核心难点**：LLM 需要精确控制坐标和布局，但 LLM 对空间关系的理解很弱。

### 路线二：生成绘图代码（matplotlib / SVG）→ 执行渲染

LLM 输出 Python 绑图代码（matplotlib/seaborn）或 SVG 标记语言，由对应引擎执行渲染。

**核心优势**：matplotlib 生态极其成熟，出版级格式化有现成方案；SVG 是 W3C 标准，浏览器原生渲染。

**核心难点**：代码正确性要求高（一个语法错误就全崩）；SVG 的坐标系统对 LLM 不友好。

### 路线三：Agent 操控绘图软件 GUI

LLM 不直接生成图形描述，而是像人一样"操作"绘图软件——点击、拖拽、设置属性。通过 MCP 协议或 COM 自动化接口实现。

**核心优势**：可以利用绘图软件的全部功能（包括自动布局、对齐辅助等）；输出是原生可编辑文件。

**核心难点**：需要逐步操作+截图反馈的迭代循环，速度慢、成本高；依赖特定软件。

### 路线四：混合路线

组合以上多种方法。例如先用 LLM 生成 SVG 布局代码，再用生成式 AI 做最终渲染；或者先生成图再逆向工程为可编辑格式。

**核心优势**：取长补短。

**核心难点**：系统复杂度高，多个环节的误差会累积。

### 你需要理解的基础概念

**draw.io XML 是什么？**

draw.io（也叫 diagrams.net）是一个开源的在线绘图工具。它用 XML 格式存储图形。一个最简单的 draw.io 文件长这样：

```xml
<mxfile>
  <diagram name="Page-1">
    <mxGraphModel>
      <root>
        <mxCell id="0"/>                    <!-- 根节点，固定存在 -->
        <mxCell id="1" parent="0"/>         <!-- 默认图层，固定存在 -->
        <mxCell id="2" value="Hello"        <!-- 一个矩形 -->
                style="rounded=1;fillColor=#dae8fc;"
                vertex="1" parent="1">
          <mxGeometry x="100" y="100" width="120" height="60" as="geometry"/>
        </mxCell>
      </root>
    </mxGraphModel>
  </diagram>
</mxfile>
```

关键概念：
- `mxCell`：每个图形元素（矩形、圆、连线、文字）都是一个 mxCell
- `vertex="1"`：表示这是一个形状（矩形、圆等）
- `edge="1"`：表示这是一条连线
- `style`：用分号分隔的 key=value 对控制外观（颜色、圆角、字体等）
- `mxGeometry`：控制位置和大小（x, y, width, height）
- `parent`：层级关系，子元素的 parent 指向父容器的 id
- `source` / `target`：连线的起点和终点（指向其他 mxCell 的 id）

**SVG 是什么？**

SVG（Scalable Vector Graphics）是 W3C 标准的矢量图形格式，本质也是 XML：

```xml
<svg width="200" height="100" xmlns="http://www.w3.org/2000/svg">
  <rect x="10" y="10" width="80" height="40" fill="#4A90D9" rx="5"/>
  <text x="50" y="35" text-anchor="middle" fill="white">Hello</text>
  <line x1="90" y1="30" x2="150" y2="30" stroke="black" stroke-width="2"/>
</svg>
```

SVG 可以直接在浏览器中渲染，也可以被 CairoSVG、Inkscape 等工具转换为 PNG/PDF。

**MCP 是什么？**

MCP（Model Context Protocol）是 Anthropic 提出的一个开放协议，让 AI 模型能够调用外部工具。你可以把它理解为"AI 的 USB 接口"——任何实现了 MCP 协议的服务都可以被 AI agent 调用。draw.io 官方已经出了 MCP Server，意味着任何支持 MCP 的 AI（Claude、Cursor 等）都可以直接操作 draw.io。

**matplotlib 是什么？**

matplotlib 是 Python 最主流的绑图库。它用代码描述图形：

```python
import matplotlib.pyplot as plt
fig, ax = plt.subplots(figsize=(3.5, 2.5))  # 直接设论文实际尺寸
ax.bar(['A', 'B', 'C'], [3, 7, 5], color='#4A90D9')
ax.set_ylabel('Value')
fig.savefig('figure.pdf', bbox_inches='tight')  # 矢量导出
```

matplotlib 的输出是真正的矢量图（PDF/SVG/EPS），可以直接用于论文投稿。

---

## 第二部分：Draw.io XML 渲染路线

### 项目1：next-ai-draw-io

| 属性 | 值 |
|------|-----|
| GitHub | https://github.com/DayuanJiang/next-ai-draw-io |
| Stars | 33,700+ |
| 许可 | Apache-2.0 |
| 技术栈 | Next.js 16 + React 19 + Vercel AI SDK 6 + react-drawio |
| 创建时间 | 2025-03 |

#### 1.1 它是怎么工作的

next-ai-draw-io 的核心思路非常直接：**让 LLM 直接输出 draw.io XML，然后在浏览器里用 draw.io 的 iframe 渲染出来。**

完整数据流：

```
用户输入 "画一个三层神经网络架构图"
    ↓
前端组装请求（用户消息 + 当前画布XML + 历史消息）
    ↓
POST /api/chat → 服务端构建 system prompt → 调用 LLM（streamText）
    ↓
LLM 流式返回 tool_call：display_diagram(xml="...")
    ↓
前端拦截 tool_call → 验证XML → 包装成完整mxfile
    ↓
通过 postMessage 发送给 draw.io iframe
    ↓
draw.io 内部的 mxGraph 引擎渲染 → 用户看到图形
    ↓
（可选）VLM 验证：截图 → 视觉模型检查 → 不合格则重试
```

#### 1.2 System Prompt 的设计（核心学习点）

这是你构建自己的 agent 时最值得学习的部分。next-ai-draw-io 的 system prompt 分为几层：

**角色定义**：
```
You are an expert diagram creation assistant specializing in draw.io XML generation.
```

**工具选择规则**（告诉 LLM 什么时候用什么工具）：
- `display_diagram`：创建新图或大幅修改时使用
- `edit_diagram`：小幅修改时使用（按 ID 增删改元素）
- `append_diagram`：XML 太长被截断时续写
- `get_shape_library`：查询可用图标库

**XML 生成规则**（约束 LLM 的输出格式）：
- 只生成 mxCell 元素，不写包装标签（包装由前端自动完成）
- ID 从 "2" 开始，必须唯一
- 所有元素的 parent 为 "1"（默认图层）或容器 ID
- 连线（edge）必须指向已存在的节点
- 布局范围：x: 0-800, y: 0-600

**7 条连线避让规则**（Edge Routing）：
这是 prompt 中最精细的部分，包括：不同连线使用不同出口位置、双向连线用对侧、使用 waypoint 绕障、禁止从角落连接等。这些规则是大量试错后总结出来的。

**Few-shot 示例**：
prompt 中包含了泳道图+连线的完整 XML 示例、waypoint 绕障示例、双向边示例、edit_diagram 的 JSON 操作示例。

**动态上下文注入**：
每次请求时，系统会把当前画布的完整 XML 注入到 system message 中：
```
Current diagram XML (AUTHORITATIVE - the source of truth):
"""xml
<mxCell id="2" value="Input" .../>
<mxCell id="3" value="Hidden" .../>
...
"""
```
这样 LLM 就知道当前图上有什么，可以做增量修改。

#### 1.3 渲染管线

渲染完全在**前端**完成：

```
React 应用
    ↓ drawioRef.current.load({ xml })
react-drawio 组件（npm 包）
    ↓ window.postMessage
draw.io iframe（https://embed.diagrams.net）
    ↓ 内部 mxGraph 引擎
可见图形（SVG 渲染）
```

关键点：
- draw.io 的渲染引擎是 mxGraph（一个 JavaScript 图形库），它在 iframe 里运行
- React 应用和 iframe 之间通过 postMessage 通信
- 导出时调用 `drawioRef.current.exportDiagram({ format: "png" })`，由 iframe 内部渲染后返回 base64 图片
- 也可以自托管 draw.io（设置 `DRAWIO_BASE_URL` 环境变量）

#### 1.4 增量编辑机制

这是"在已有图上修改"而不是每次重新生成的关键：

LLM 返回 `edit_diagram` 工具调用，包含一个 operations 数组：

```json
{
  "operations": [
    {"type": "update", "cell_id": "5", "new_xml": "<mxCell id=\"5\" value=\"New Label\" .../>"},
    {"type": "add", "cell_id": "10", "new_xml": "<mxCell id=\"10\" .../>"},
    {"type": "delete", "cell_id": "3"}
  ]
}
```

前端收到后，对当前 XML 做 DOM 级操作：
- `update`：找到 id="5" 的 mxCell，整个替换
- `add`：在 root 下追加新 mxCell
- `delete`：级联删除（包括子节点和关联的连线）

**MCP Server 的编辑门控（Edit Gate）**：
防止 AI 覆盖用户的手动编辑。每次编辑前检查：模型上次看到的 XML 和当前浏览器中的 XML 是否一致。如果用户手动改了图，AI 的编辑会被拒绝，需要先 `get_diagram` 获取最新状态。

#### 1.5 XML 质量保证（三层防线）

1. **结构验证**：`validateAndFixXml()` 检查语法错误、重复 ID、非法字符、嵌套违规
2. **自动修复**：`autoFixXml()` 修复转义问题、重复属性、缺失闭合标签
3. **VLM 视觉验证**：渲染成 PNG → 发给视觉模型 → 检查重叠/穿越/截断 → 不合格则反馈给 LLM 重试（最多 3 次）

#### 1.6 技术难点

| 难点 | 具体表现 | 解决方案 |
|------|----------|----------|
| LLM 输出截断 | 复杂图的 XML 很长，LLM 可能写到一半就停了 | `append_diagram` 工具支持多次续写；`jsonrepair` 修复截断的 JSON |
| 布局质量 | LLM 对空间关系理解弱，元素容易重叠 | 7 条 Edge Routing 规则 + 布局范围约束 + VLM 验证兜底 |
| 模型能力要求高 | 弱模型生成的 XML 质量很差 | README 明确推荐 Claude Sonnet 4.5、GPT-5.1、Gemini 3 Pro |
| 渲染依赖外部服务 | 默认依赖 embed.diagrams.net | 支持自托管 |
| MCP 通信靠轮询 | 浏览器与 MCP Server 之间是 HTTP 轮询 | 有延迟但实现简单 |

#### 1.7 对你的启示

**如果你要构建一个 draw.io XML 生成 agent，next-ai-draw-io 告诉你：**

1. **Prompt 工程是核心**：不是简单地告诉 LLM "生成 draw.io XML"，而是需要详细的规则约束（ID 规则、布局范围、连线避让）和 few-shot 示例。这些规则是大量试错的结晶。

2. **增量编辑比全量重生成更实用**：用户说"把第三个框改成红色"时，不应该重新生成整张图。edit_diagram 的 update/add/delete 三种操作是必须设计的。

3. **XML 验证和修复不可少**：LLM 生成的 XML 经常有各种小问题（未闭合标签、重复 ID、转义错误），必须有自动修复机制。

4. **视觉验证是最后防线**：即使 XML 语法正确，渲染出来也可能很丑（重叠、穿越）。VLM 验证虽然增加成本，但能显著提升质量。

5. **draw.io iframe 是最简单的渲染方案**：不需要自己实现渲染引擎，直接嵌入 draw.io 的 iframe 即可。

---

### 项目2：drawio-mcp（官方）

| 属性 | 值 |
|------|-----|
| GitHub | https://github.com/jgraph/drawio-mcp |
| Stars | 4,900+ |
| 许可 | Apache-2.0 |
| 维护者 | JGraph（draw.io 官方团队） |
| 创建时间 | 2026-02 |

#### 2.1 它是什么

drawio-mcp 是 draw.io 官方出的 MCP Server，让任何支持 MCP 的 AI agent 都能操作 draw.io。它不是一个完整的"AI 绘图应用"，而是一个**基础设施层**——提供标准化的工具接口，上层的 AI agent（Claude、Cursor、你自己写的 agent）通过调用这些工具来创建和编辑图形。

#### 2.2 四种接入方式

| 方式 | 适用场景 | 安装方式 |
|------|----------|----------|
| MCP App Server | 聊天内嵌交互式预览 | 添加 `https://mcp.draw.io/mcp` 到 MCP 客户端 |
| MCP Tool Server | 本地桌面工作流 | `npx @drawio/mcp` |
| Claude Code Plugin | 开发工作流 | `/plugin install drawio@drawio` |
| Project Instructions | 零安装 | 把 xml-reference.md 粘贴到 Claude Project 指令中 |

#### 2.3 提供的工具

**App Server 工具**：
- `create_diagram(xml)`：创建图表，返回交互式查看器
- `search_shapes(query, limit)`：搜索 10,000+ 内置图形（支持语音模糊匹配）

**Tool Server 工具**：
- `open_drawio_xml(xml)`：在浏览器打开 draw.io 编辑器
- `open_drawio_csv(csv)`：从 CSV 导入
- `open_drawio_mermaid(mermaid)`：从 Mermaid 语法导入
- `search_shapes(query)`：搜索图形
- `list_pages(file)`：列出多页文件的页面
- `get_page(file, page)` / `set_page(file, page, xml)`：读写特定页面

#### 2.4 XML 格式规范（核心学习点）

drawio-mcp 的 `shared/xml-reference.md` 是目前最权威的 draw.io XML 参考文档。关键规范：

**标准尺寸和网格**：
- 矩形：140×60
- 菱形：140×80
- 圆形：60×60
- 列间距：x = col_index × 180 + 40
- 行间距：y = row_index × 120 + 40

**样式语法**：
```
style="rounded=1;fillColor=#dae8fc;strokeColor=#6c8ebf;fontSize=12;fontFamily=Arial;"
```
分号分隔的 key=value 对。常用属性：
- `rounded=1`：圆角
- `fillColor`：填充色
- `strokeColor`：边框色
- `fontSize`：字号
- `container=1`：标记为容器（子元素坐标相对于容器）
- `html=1`：允许 HTML 标签

**布局引擎**：
- **ELK**：自动层次布局（重新排列节点+路由边）
- **libavoid**：保持节点位置，仅重新路由连线
- 两者只能用其一

**XML 规范化**：
LLM 输出的 XML 经常被 JSON 包裹（最多 4 层），`normalize-diagram-xml.js` 会自动剥离。还会将相对图片路径转为绝对 URL。

#### 2.5 导出机制

**完全依赖本地 draw.io 桌面版 CLI**，不使用任何云端渲染服务：

```bash
# 检测是否安装了 draw.io 桌面版
which drawio

# 命令行导出
drawio --export --format png --output output.png input.drawio
drawio --export --format svg --output output.svg input.drawio
drawio --export --format pdf --output output.pdf input.drawio
```

draw.io Desktop 是基于 Electron 的应用，内嵌了完整的 mxGraph 渲染引擎。

#### 2.6 对你的启示

1. **MCP 是连接 AI 和绘图工具的标准接口**：如果你要构建 agent，MCP 是最值得学习的集成方式。它定义了工具发现、调用、返回的标准流程。

2. **Shape 搜索是被低估的能力**：draw.io 有 10,000+ 内置图形（AWS 图标、网络设备、化学结构等），通过 `search_shapes` 工具，LLM 可以找到合适的图标而不是自己画。

3. **XML 参考文档本身就是 prompt**：`xml-reference.md` 可以直接作为 system prompt 的一部分注入给 LLM，让它知道如何生成正确的 XML。

4. **导出依赖桌面版是一个限制**：如果你的 agent 需要在服务器端运行（没有 GUI），需要考虑替代方案（如 headless 渲染）。

---

### 项目3：Pegasus

| 属性 | 值 |
|------|-----|
| GitHub | https://github.com/HANsoA-KevinO/Pegasus |
| Stars | 2 |
| 许可 | MIT |
| 技术栈 | Next.js 16 + React 19 + MongoDB + Sharp + Puppeteer |
| 状态 | 早期（10 commits） |

#### 3.1 它是什么

Pegasus 是目前最贴合"科研绘图 agent"定义的项目。它的目标不是画通用流程图，而是**生成论文级别的科研配图**（如分子机制图、实验流程图、模型架构图）。

#### 3.2 七阶段流水线

Pegasus 的核心创新在于它的七阶段流水线：

```
阶段1: 输入分析
  → 分析用户文本，识别学科领域、目标期刊、逻辑结构
  → 工具：Claude Opus 4.6

阶段2: 逻辑结构设计
  → 从文本中提取论文配图的逻辑关系（流程、对比、层次）
  → 工具：Claude Opus 4.6

阶段3: 视觉规格书
  → 确定配色方案、期刊风格、尺寸规范、图标需求
  → 工具：Claude Opus 4.6 + 期刊风格知识库

阶段4: 绘图 Prompt 生成
  → 将逻辑结构和视觉规格转化为图像生成 prompt
  → 工具：Claude Opus 4.6

阶段5: 图像生成
  → 根据 prompt 生成科研配图
  → 工具：Gemini 3 Pro Image Preview（注意：这里用了生成式AI）

阶段6: 审核与修正
  → 多轮图像编辑、用户确认
  → 工具：Gemini 3 Pro + 用户交互

阶段7: 可编辑 XML 组装（核心创新）
  → 将 AI 图像逆向工程为可编辑 Draw.io XML
  → 子步骤：
    7a. 生成 Icons-only 图像（仅图标版本）
    7b. 去除白色背景（Sharp 库）
    7c. 检测并裁剪各个图标区域
    7d. 视觉模型逆向工程出 Draw.io XML 模板
    7e. 视觉一致性审核
    7f. 将裁剪的图标以 base64 data URI 嵌入 XML
```

**注意**：Pegasus 在阶段 5 使用了生成式 AI（Gemini 图像生成），但阶段 7 的"逆向工程为可编辑 XML"是它的核心创新。如果你要做纯非生成式 AI 的方案，可以跳过阶段 5-6，直接从文本生成 XML。

#### 3.3 视觉审核机制

Pegasus 的视觉审核是一个结构化循环：

```
XML文件 → Puppeteer 渲染 PNG 预览
    → 视觉模型对比审核（检查箭头方向、连线、标签、颜色、布局）
    → 发现问题 → 读取 XML → 修复 → 再渲染
    → 最多 2 轮 → 记录剩余缺陷 → 继续
```

**RenderSvg 工具**的实现：
- 启动 headless Puppeteer 浏览器
- 将 SVG 嵌入最小 HTML 模板
- 截取 PNG screenshot
- 裁剪到 SVG 精确尺寸

#### 3.4 领域特化

Pegasus 有一个 Skills 系统，按学科领域提供特化知识：

```
lib/skills/
├── scientific-drawing/SKILL.md   # 核心方法论
├── visual-review/SKILL.md        # 审核流程
├── biology/SKILL.md              # 生物学（细胞、DNA、蛋白质图标）
├── computer-science/SKILL.md     # 计算机科学（服务器、网络拓扑）
└── economics/SKILL.md            # 经济学（图表、货币符号）
```

期刊风格处理：读取目标期刊信息 → 提取排版规范（配色、字体、尺寸）→ 如果没有信息则通过 WebSearch 搜索。

#### 3.5 对你的启示

1. **"先生图再逆向"vs"直接生成 XML"**：Pegasus 选择了先生成图像再逆向为 XML 的路线。如果你不用生成式 AI，就需要直接让 LLM 生成 XML，这对 prompt 工程的要求更高。

2. **期刊风格知识库是有价值的**：不同期刊对配图有不同的风格要求（Nature 的配色 vs IEEE 的配色），把这些知识编码成 skill 是一个好思路。

3. **Puppeteer 是服务端渲染 SVG 的可行方案**：如果你需要在服务器端把 SVG 渲染成 PNG（用于预览或验证），Puppeteer（headless Chrome）是最简单的方案。

4. **项目太早期，代码参考价值有限**：只有 10 个 commit，很多功能可能还不完善。但它的架构设计思路值得参考。

---

## 第三部分：代码渲染路线

### 项目4：scipilot-figure-skill

| 属性 | 值 |
|------|-----|
| GitHub | https://github.com/Haojae/scipilot-figure-skill |
| Stars | 1,280+ |
| 许可 | MIT |
| 技术栈 | Python（matplotlib + seaborn + SciencePlots） |
| 创建时间 | 2026-05 |

#### 4.1 它是什么

scipilot-figure-skill 不是一个独立应用，而是一个 **Claude Code Skill**——一份结构化的指令文档（SKILL.md），告诉 Claude 如何像一个"可视化顾问"一样工作。它的核心理念是**"先思考后绘图"**：不是一收到请求就画图，而是先分析数据、选择图表类型、确认期刊规范，然后再画。

#### 4.2 Skill 的结构

```
├── SKILL.md              # 核心技能定义（~400行指令）
├── scripts/              # 6个Python工具脚本
│   ├── profile_data.py   # 数据剖析（EDA）
│   ├── setup_style.py    # 期刊样式配置
│   ├── export_figure.py  # 多格式导出
│   ├── check_figure.py   # 出版合规自检
│   ├── layout_tools.py   # 子图标签对齐
│   └── visual_qa.py      # 渲染预览+程序自检
└── references/           # 7份参考文档
    ├── chart_selection.md      # 选图决策框架
    ├── data_profiling.md       # 数据剖析报告解读
    ├── journal_specs.md        # 期刊规范
    ├── plot_recipes.md         # 9类图代码配方
    ├── publication_checklist.md # 投稿前合规清单
    ├── visual_review.md        # AI读图8项清单
    └── viz_pitfalls.md         # 18条科研画图禁忌
```

#### 4.3 八步工作流（核心学习点）

这是整个 Skill 的灵魂：

| 步骤 | 内容 | 关键动作 |
|------|------|----------|
| 第0步 | 理解任务 | 搞清楚"这张图要论证什么"，主动问用户 |
| 第1步 | 剖析数据 | 调用 `profile_data.py` 做 EDA（列类型、样本量、缺失率、偏度、异常值） |
| 第2步 | 选图 | 基于数据事实 + 论证目标，查决策框架 |
| 第3步 | 查期刊规范 | 确定栏宽、字号、DPI、格式 |
| 第4步 | 配环境 | `setup_style(journal, lang)` |
| 第5步 | 绘制 | 按配方画，强制最终尺寸 |
| 第6步 | 自检闭环 | 三层检查（语义+形式+视觉） |
| 第7步 | 导出 | 多格式 + 灰度预览 + 机器审计 |

**选图决策框架**（chart_selection.md 的核心）：

| 数据形态 | 推荐 | 不该用 |
|----------|------|--------|
| 1分类+1连续，n<10/组 | stripplot/dot plot | 均值柱（严禁） |
| 1分类+1连续，n≥10/组 | 箱线/小提琴+stripplot | 仅均值柱 |
| 2连续看关系 | 散点+回归+r值 | 折线 |
| 时间/剂量vs连续 | 折线+误差带 | 柱状 |
| 构成占比 | 堆叠柱/treemap | 饼图 |

#### 4.4 出版级格式化

**五条硬性原则**：

1. **按最终尺寸出图，不二次缩放**：`figsize=(3.5, 2.625)` 直接设论文实际尺寸（Nature 单栏 3.5 英寸），导出后绝不在 Word/LaTeX 里再缩放。

2. **矢量优先**：数据图→PDF/SVG/EPS；照片→TIFF/PNG(300-600DPI)；**绝对不用 JPEG**。

3. **色盲友好配色**：默认 Okabe-Ito 配色（`#E69F00, #56B4E9, #009E73, #F0E442, #0072B2, #D55E00, #CC79A7`）+ 冗余编码（线型/marker 区分）+ 灰度预览。

4. **字号可读**：最小字≥6pt。

5. **误差必有交代**：图注写清 SD/SEM/95%CI + n + 检验方法。

**期刊规范对比**（journal_specs.md）：

| 期刊 | 单栏宽 | 双栏宽 | 字号 | 字体 | 格式 |
|------|--------|--------|------|------|------|
| Nature | 3.5in | 7.2in | 7-9pt | Arial/Helvetica | PDF/EPS |
| IEEE | 3.5in | 7.16in | 7-9pt | Times New Roman | EPS |
| 中文核心 | 视期刊 | 视期刊 | ≥6pt | 宋体+TNR数字 | PDF |

#### 4.5 视觉自检循环（三层）

**第一层：语义层**
对照 18 条科研画图禁忌检查（如"不要用饼图"、"不要用均值柱代替分布图"）。

**第二层：形式层**
对照投稿清单检查尺寸、DPI、字号、误差交代。`check_figure.py` 会：
- 检查光栅图 DPI 和物理尺寸
- 拒绝 JPEG 用于线/文字图
- 检查 PDF 字体（检测 Type 3 字体和未嵌入字体）
- 检查 SVG 中是否嵌入了位图

**第三层：视觉层**
1. `visual_qa.render_preview(fig)` 渲染 PNG 预览
2. `visual_qa.audit_layout(fig)` 程序化检测确定性问题（缺字、文字裁切、刻度重叠）
3. 用 Claude 的 Read 工具读 PNG，对照 8 项清单核对感知问题（图例压数据、配色灰度可分等）
4. 发现问题 → 修改代码 → 重渲染 → 再读图 → 直到通过

#### 4.6 CJK 字体处理

matplotlib 默认字体不含中文字符，会显示为方框。`setup_style.py` 的解决方案：

```python
# 按优先级查找中文字体
font_candidates = ['Noto Sans CJK SC', 'Source Han Sans SC', 'SimHei', 'Microsoft YaHei']
# 修负号方框
plt.rcParams['axes.unicode_minus'] = False
# 找不到任何CJK字体 → 抛出清晰的安装提示
```

#### 4.7 对你的启示

1. **"先思考后绘图"是最重要的设计理念**：不要一收到请求就画图。先理解数据、选择图表类型、确认规范，然后再画。这比画完再改效率高得多。

2. **代码配方（plot_recipes）是核心资产**：把常见图表类型的代码模板化，LLM 只需要在模板基础上修改参数，而不是从零写代码。这大幅降低了出错概率。

3. **三层自检闭环是质量保证的关键**：语义检查（图型对不对）→ 形式检查（格式合不合规）→ 视觉检查（看起来好不好）。缺任何一层都可能"带病投稿"。

4. **Skill 的形式值得借鉴**：把领域知识（期刊规范、选图框架、禁忌清单）编码成结构化的参考文档，让 LLM 在工作流中按需查阅，比全部塞进 prompt 更高效。

5. **matplotlib 是科研数据图的最佳选择**：对于数据驱动的图表（柱状图、折线图、散点图、热力图等），matplotlib + seaborn 是最成熟、最可控的方案。

---

### 项目5：NaturePanelForge

| 属性 | 值 |
|------|-----|
| GitHub | https://github.com/littlepeachs/NaturePanelForge |
| Stars | 204 |
| 许可 | MIT |
| 技术栈 | Python + matplotlib + YOLOv12 + Qwen + Codex |
| 创建时间 | 2026-06 |

#### 5.1 它是什么

NaturePanelForge 做的是 scipilot-figure-skill 的**逆向**：不是"有数据→画图"，而是"看到 Nature 论文里的好图→逆向生成可运行的 matplotlib 代码"。它的目标是建立一个"Nature 级科研图表的代码库"。

#### 5.2 三阶段工作流

```
阶段1: Panel Split（拆分）
  输入：一张完整的论文 figure（含 a/b/c/d 多个 panel）
  输出：独立的 panel 图像
  方法：
    路径A: YOLOv12 目标检测（快但不够精确）
    路径B: Codex Agent 循环（慢但精确）
      → Split Agent 写裁剪脚本（bbox 定义在代码中）
      → 运行脚本生成裁剪结果
      → Review Agent 检查每个 panel 的四边完整性
      → 发现问题 → 修改 bbox → 重跑
      → 4 轮审核

阶段2: Code Reproduce（代码复现）
  输入：单个 panel 图像 + 元数据 + caption
  输出：可运行的 matplotlib 代码
  方法：Codex Agent 循环
    → 读图 + 元数据
    → 写 reproduce_panel.py
    → 运行生成 PNG/PDF
    → 对比输出与 target（9 个维度检查）
    → 不通过则修改代码重跑

阶段3: Final Refine（最终精修）
  输入：通过复现的代码 + 渲染结果
  输出：精修后的代码 + 最终图像
  方法：
    → 使用 Arial 字体
    → 检查所有文字元素防止重叠
    → 验证数学符号、希腊字母、单位
    → 手动调整边距
```

#### 5.3 Qwen 视觉评分（质量筛选）

不是所有 panel 都适合代码复现（比如电镜照片、化学结构式）。NaturePanelForge 用本地部署的 Qwen3.6-27B 视觉模型对每个 panel 打分：

**评分维度（0-10分）**：
- `clarity_integrity_score`：完整性、清晰度
- `data_purity_score`：是否纯净数据统计图
- `code_reproducibility_score`：能否用代码复现
- `aesthetic_score`：美学程度
- `overall_quality_score`：综合分

**筛选门槛**：
- `data_purity_score >= 10`（必须是纯数据图）
- `code_reproducibility_score >= 9`
- `aesthetic_score >= 8`

**分类体系**：5 大类（schematic / data_statistical / chemical_structure / characterization_photo / other），30+ 数据图子类型（bar, line, scatter, box, violin, heatmap, volcano_plot, umap_tsne_pca 等）。

#### 5.4 Codex Agent 循环的技术细节

```python
cmd = [codex, "exec",
    "--skip-git-repo-check",
    "--json",
    "--output-last-message", str(last_message),
    "--output-schema", str(schema_path),    # 强制结构化输出
    "--sandbox", "workspace-write",          # 允许写文件和运行代码
    "--cd", str(BASE_DIR),
    "--image", str(image_path),              # 图像作为输入
    "--model", model,
    prompt]
```

关键设计：
- 图像通过 `--image` 参数直接传给 Codex
- `--output-schema` 强制返回符合 JSON schema 的结构化响应
- 并发控制：限制同时运行的 Codex 进程数（默认 16）
- 超时 900s，最多 5 次重试

#### 5.5 对你的启示

1. **"图→代码"逆向是一个有价值的方向**：如果你能建立一个"Nature 级图表的代码库"，就可以用这些代码作为模板来画自己的图。这比让 LLM 从零生成代码可靠得多。

2. **Panel 拆分是预处理的关键**：论文 figure 通常是多 panel 组合图，需要先拆分才能单独处理。YOLOv12 检测 + Codex 代码循环是两条可行路径。

3. **VLM 评分的校准很难**：VLM 倾向于给高分（"分数膨胀"），prompt 中需要大量"严格打分"的校准指令。

4. **Code-in-the-Loop 是核心范式**：不是让 LLM 一次性生成完美代码，而是"写代码→运行→看结果→改代码→再运行"的迭代循环。这个范式适用于所有代码生成任务。

---

### 项目6：LLM4SVG

| 属性 | 值 |
|------|-----|
| GitHub | https://github.com/ximinng/LLM4SVG |
| Stars | 651 |
| 论文 | CVPR 2025 |
| 技术栈 | PyTorch + GPT-2 XL / Phi-2 / Qwen2.5-VL |

#### 6.1 它是什么

LLM4SVG 是一个学术研究项目，研究的核心问题是：**如何让 LLM 理解和生成复杂的 SVG 矢量图形？** 它不是直接让 LLM 写 SVG 代码，而是引入了 55 个专用的 SVG 语义 token，让 LLM "看懂" SVG 的结构化语法。

#### 6.2 核心创新：55 个 SVG 语义 Token

SVG 代码虽然本质是文本，但它的语法结构（标签、属性、路径命令）与自然语言差异极大。直接用 LLM 的原始 tokenizer 处理 SVG 会导致语义模糊和幻觉。

LLM4SVG 定义了 55 个专用 token：

| 类别 | 数量 | 示例 |
|------|------|------|
| Tag Tokens | 15 | `<path>`, `<circle>`, `<rect>`, `<ellipse>`, `<text>`, `<g>` |
| Attribute Tokens | 30 | `fill`, `stroke`, `d`, `cx`, `cy`, `r`, `width`, `height`, `transform` |
| Path Command Tokens | 10 | `M`(moveto), `L`(lineto), `C`(三次贝塞尔), `Q`(二次贝塞尔), `A`(圆弧), `Z`(闭合) |

**Token 嵌入初始化**：新 token 的嵌入向量通过对其简短文本描述（如 "move to command"）的已有嵌入取平均来初始化，而非随机初始化。

**生成示例**：
```
原始 SVG: <rect x="10" y="20" width="100" height="50" fill="#FF0000"/>
LLM4SVG:  [RECT] [X] 10 [Y] 20 [WIDTH] 100 [HEIGHT] 50 [FILL] #FF0000
```

#### 6.3 训练数据和策略

**SVGX 数据集**：
- 25 万个人工设计的彩色复杂 SVG 文件
- 清洗 → 归一化（128×128 画布）→ 光栅化 → 用 BLIP/GPT-4 生成描述
- 最终 58 万条指令样本（25 万文本→SVG + 25 万文本+图像→SVG + 6 万理解 + 2 万组级理解）

**两阶段训练**：
1. 特征对齐：冻结 LLM，仅训练 SVG token 嵌入
2. 监督微调：训练嵌入 + LLM 参数（LoRA/QLoRA 或全参数）

#### 6.4 关键发现

- **GPT-2 XL（1.5B）经过专门训练后超越 GPT-4o**：说明领域适配比模型规模更重要
- **GPT-2 的 tokenizer 对数字处理更好**：它将数字和小数点作为独立 token，有利于精确的坐标生成
- **生成速度**：18 秒/10 个 SVG，远快于优化方法（CLIPDraw 等需要数分钟）
- **人类评估**：提示对齐 0.89、视觉质量 0.92（人工设计 SVG 为 0.94/0.95）

#### 6.5 对你的启示

1. **SVG 不应被当作普通文本处理**：如果你要让 LLM 生成 SVG，考虑引入结构化的表示方式（专用 token 或至少是结构化的 prompt 约束）。

2. **小模型 + 领域适配 > 大模型 + 通用能力**：对于特定领域的代码生成，微调一个小模型可能比调用通用大模型效果更好。

3. **数据工程是基础**：58 万条高质量指令数据是成功的关键。如果你要训练自己的 SVG 生成模型，数据收集和质量控制是第一步。

4. **但这个项目偏学术**：它需要 GPU 训练和推理，不是一个开箱即用的工具。对于你的 agent 项目，更实际的方案可能是用通用 LLM + 精心设计的 prompt 来生成 SVG。

---

## 第四部分：GUI 操控路线

### 项目7：VisPainter

| 属性 | 值 |
|------|-----|
| 论文 | arXiv: 2510.27452 |
| GitHub | https://github.com/HerzogFL/VisPainter |
| 技术栈 | MCP + Cline + Visio COM 自动化 |

#### 7.1 它是什么

VisPainter 是一个多智能体框架，通过 MCP 协议操控 Microsoft Visio 来生成可编辑的科研矢量图。它不生成 XML 或代码，而是**像人一样操作绘图软件**——插入形状、设置颜色、对齐、连线。

#### 7.2 三角色架构

```
Manager（管理者）
  → 解析用户请求为结构化任务
  → 跟踪任务状态
  → 选择下一步工具操作
  → 将操作打包为 MCP 请求

Designer（设计者）
  → 强 VLM（如 Gemini-2.5-Pro）
  → 生成绘图计划（对象列表 + 位置）
  → 每次渲染后接收截图并修改布局
  → 迭代优化直到满意

Toolbox（工具箱）
  → 无状态 MCP 服务
  → 封装 30+ Visio 底层操作
  → 每次操作创建矢量对象并返回截图
```

**为什么分离 Manager 和 Designer？** 消融实验证明：合并两者会导致精度下降 23%，召回率下降 27%。"规划"和"执行"必须解耦。

#### 7.3 截图引导的迭代编辑

这是 VisPainter 最核心的机制：

```
Designer 生成绘图计划
    ↓
Manager 调用 Toolbox 执行原子操作（如"在(100,200)插入一个矩形"）
    ↓
Toolbox 执行操作 → 自动截图 → 返回截图给 Designer
    ↓
Designer 看截图 → 发现问题（重叠、缺失、超出画布）
    ↓
Designer 生成修正指令 → 继续迭代
    ↓
直到满意或达到步数上限
```

**步数粒度的甜蜜点**：

| 每轮放置元素数 | 平均步数 | 效果 |
|---------------|---------|------|
| 1 | ~38 | 质量最高但最慢 |
| 2-4 | ~10.5 | **最佳平衡点** |
| 8 | - | 布局错误开始出现 |
| 16 | - | 性能崩溃（"认知拥堵"） |

#### 7.4 VisBench 评测基准

360 个密集科研图表，七维评估：

| 维度 | 权重 | 含义 |
|------|------|------|
| Precision | 0.20 | 必需文本中正确的比例 |
| Recall | 0.20 | 必需文本中实际出现的比例 |
| Design-error | 0.20 | 连接线/重叠/溢出缺陷 |
| Blank-space | 0.05 | 过多空白 |
| Readability | 0.25 | 文本可见可读的比例 |
| Alignment | 0.10 | 行列规整度 |
| Steps | - | MCP 原子命令数量 |

**实验结果**（T2I 任务 DQS 排名）：
1. Gemini-2.5-Pro: 0.85
2. GPT-5: 0.84
3. GPT-o3: 0.82
4. Claude-Opus-4: 0.78

#### 7.5 对你的启示

1. **"原子操作 + 视觉反馈"是 GUI 操控的核心范式**：每步一个小操作 + 截图确认，比一次性生成整个图更可靠。这个思路也可以应用到 draw.io XML 生成中——不是一次生成完整 XML，而是逐步添加元素并验证。

2. **角色分离带来显著质量提升**：规划（Designer）和执行（Toolbox）分离，比让一个 agent 同时做两件事效果好得多。

3. **步数粒度是甜蜜点问题**：每轮 2-4 个元素是最佳平衡。太多会"认知拥堵"，太少则效率低。

4. **但 GUI 操控路线的部署成本很高**：需要 Windows + Visio + COM 自动化。对于你的项目，draw.io XML 路线可能是更实际的选择。

5. **可编辑性是核心优势**：输出是真正的矢量文件，任何元素都可以后续修改。这是光栅图生成无法比拟的。

---

## 第五部分：混合路线

### 项目8：Paper2Any

| 属性 | 值 |
|------|-----|
| GitHub | https://github.com/OpenDCAI/Paper2Any |
| Stars | 2,700+ |
| 许可 | Apache-2.0 |
| 技术栈 | Python + LangGraph + python-pptx + draw.io |

#### 8.1 它是什么

Paper2Any 是一个"论文→任何格式"的全栈系统，包含十几个模块：Paper2Figure、Paper2Diagram、Paper2PPT、Paper2Poster、Paper2Video 等。它的核心是基于 LangGraph StateGraph 编排的 40+ Agent 工作流。

#### 8.2 Paper2Diagram 模块

```
论文文本 → DiagramPlanner（规划图表结构）
    → DrawioXmlGenerator（生成 draw.io XML）
    → VLM 验证（检查渲染结果）
    → 对话编辑（用户修改）
    → 输出 .drawio / .png / .svg
```

#### 8.3 Paper2Figure 模块

两条路线：
- **SVG 代码渲染**：黑白线稿 → 上色 → 图标嵌入（纯代码，无生成式 AI）
- **Matplotlib 代码执行**：生成 Python 代码 → 执行 → 输出图表

#### 8.4 PPTX 输出

使用 `python-pptx` 直接构建 PPTX 文件。有一个 `PptTextFitter` 组件做真实渲染驱动的字号自适应——先渲染文字，如果超出框就缩小字号，直到合适。

#### 8.5 对你的启示

1. **LangGraph 是编排多 Agent 工作流的成熟框架**：如果你要构建复杂的多步骤管线，LangGraph 的 StateGraph 是值得学习的工具。

2. **python-pptx 是程序化生成 PPT 的标准方案**：如果你需要把生成的图插入 PPT，python-pptx 是最直接的选择。

3. **但系统复杂度很高**：40+ Agent 的编排、多模块协调，对于初学者来说可能过于复杂。建议先从单一功能开始。

---

### 项目9：AutoFigure

| 属性 | 值 |
|------|-----|
| GitHub | https://github.com/ResearAI/AutoFigure |
| Stars | 1,760+ |
| 许可 | MIT |
| 论文 | ICLR 2026 |

#### 9.1 它是什么

AutoFigure 的核心是"Reasoned Rendering"：LLM 从论文文本中提炼方法结构，生成 SVG/HTML 布局代码，经 critic/designer 循环修正后，再引导图像生成模型出最终图，最后用 OCR 擦除修正文字。

#### 9.2 迭代循环

```
Generate（LLM 生成 SVG/HTML 布局代码）
    ↓
Render（CairoSVG 或 Playwright+draw.io 渲染为 PNG）
    ↓
Evaluate（VLMJudgeEvaluator 从 5 个维度打分）
    ↓
分数 < 9.0 → Improve（LLM 根据反馈修改代码）→ 回到 Render
分数 ≥ 9.0 → 完成
最多 5 轮
```

#### 9.3 对你的启示

1. **Generate → Render → Evaluate → Improve 是一个通用范式**：这个循环适用于任何代码生成任务。关键是 Evaluate 环节——如何自动判断生成结果的质量。

2. **CairoSVG 是服务端渲染 SVG 的轻量方案**：不需要浏览器，纯 Python 就能把 SVG 转成 PNG。

3. **但 AutoFigure 的最终渲染依赖生成式 AI**：它的布局是代码驱动的，但最终出图用了图像生成模型。如果你要纯代码路线，可以只取它的布局生成和迭代修正部分。

---

## 第六部分：横向对比

### 6.1 技术路线对比

| 维度 | Draw.io XML | matplotlib 代码 | SVG 代码 | GUI 操控 |
|------|-------------|----------------|----------|----------|
| 输出格式 | .drawio（矢量） | PDF/SVG/EPS（矢量） | SVG（矢量） | .vsdx（矢量） |
| 可编辑性 | 高（draw.io 打开即编辑） | 中（改代码重跑） | 中（改代码重渲染） | 高（Visio 打开即编辑） |
| 渲染依赖 | draw.io 引擎 | matplotlib | 浏览器/CairoSVG | Visio |
| 适合场景 | 流程图、架构图、示意图 | 数据图（柱/线/散点/热力） | 任意图形 | 复杂科研图 |
| LLM 难度 | 中（XML 结构固定） | 低（代码模板成熟） | 高（坐标系统复杂） | 中（原子操作简单） |
| 布局控制 | 手动坐标 | 自动（matplotlib 排版） | 手动坐标 | 手动+软件辅助 |
| 部署复杂度 | 低 | 低 | 低 | 高（需 Windows+Visio） |

### 6.2 项目成熟度对比

| 项目 | Stars | 活跃度 | 可直接使用 | 学习价值 |
|------|-------|--------|-----------|----------|
| next-ai-draw-io | 33.7k | 非常活跃 | 是 | 高（prompt 设计、XML 处理） |
| drawio-mcp | 4.9k | 活跃 | 是 | 高（MCP 集成、XML 规范） |
| scipilot-figure-skill | 1.3k | 活跃 | 是（需 Claude Code） | 极高（工作流设计、质量保证） |
| Paper2Any | 2.7k | 活跃 | 部分 | 中（系统复杂） |
| AutoFigure | 1.8k | 活跃 | 部分 | 中（迭代循环设计） |
| NaturePanelForge | 204 | 活跃 | 部分 | 高（Code-in-the-Loop） |
| LLM4SVG | 651 | 学术 | 否（需训练） | 高（SVG tokenization） |
| VisPainter | 论文 | 学术 | 否（需 Visio） | 高（多 Agent 架构） |
| Pegasus | 2 | 早期 | 否 | 中（架构思路） |

### 6.3 质量保证机制对比

| 项目 | 验证层数 | 验证方式 |
|------|---------|----------|
| next-ai-draw-io | 3 层 | XML 结构验证 → 自动修复 → VLM 视觉验证 |
| scipilot-figure-skill | 3 层 | 语义检查（禁忌）→ 形式检查（合规）→ 视觉检查（AI 读图） |
| NaturePanelForge | 3 层 | Qwen 评分筛选 → Codex Review Agent → 最终精修审核 |
| VisPainter | 2 层 | 截图视觉反馈 → VLM 评判 |
| AutoFigure | 2 层 | VLM 5 维评分 → 迭代修正 |
| Pegasus | 2 层 | Puppeteer 渲染 → 视觉模型审核（最多 2 轮） |

---

## 第七部分：初学者构建指南

### 7.1 你应该从哪条路线开始？

**如果你的目标是画数据图（柱状图、折线图、散点图、热力图等）**：
→ 从 **matplotlib 代码路线**开始。参考 scipilot-figure-skill 的设计，建立代码模板库 + 选图决策框架 + 自检闭环。这是最成熟、最可控的路线。

**如果你的目标是画示意图（模型架构图、反应路径图、实验流程图等）**：
→ 从 **draw.io XML 路线**开始。参考 next-ai-draw-io 的 prompt 设计和 drawio-mcp 的 XML 规范。这是目前生态最完善的路线。

**如果你想两者都做**：
→ 先做 matplotlib 路线（更简单），再做 draw.io 路线。

### 7.2 最小可行产品（MVP）的设计

**阶段一：matplotlib 数据图 agent**

```
输入：用户描述 + 数据文件（CSV）
    ↓
步骤1：数据剖析（列类型、样本量、分布）
    ↓
步骤2：选图（基于数据形态 + 用户意图）
    ↓
步骤3：生成 matplotlib 代码（基于模板）
    ↓
步骤4：执行代码 → 渲染 PNG
    ↓
步骤5：自检（程序化检查 + 视觉检查）
    ↓
步骤6：导出 PDF/SVG/EPS
```

你需要：
- 一个 LLM API（如 Claude、GPT、Qwen）
- Python + matplotlib + seaborn
- 一套代码模板（plot_recipes）
- 一个选图决策框架
- 一个自检脚本

**阶段二：draw.io 示意图 agent**

```
输入：用户描述
    ↓
步骤1：分析结构（有哪些元素、什么关系）
    ↓
步骤2：生成 draw.io XML（基于 prompt 约束）
    ↓
步骤3：验证 XML（结构检查 + 自动修复）
    ↓
步骤4：渲染（draw.io iframe 或 headless）
    ↓
步骤5：视觉验证（可选）
    ↓
步骤6：导出 .drawio / .png / .svg
```

你需要：
- 一个 LLM API
- draw.io XML 参考文档（作为 prompt 的一部分）
- XML 验证和修复逻辑
- 渲染方案（前端 iframe 或 Puppeteer）

### 7.3 关键技术难点及应对

| 难点 | 原因 | 应对策略 |
|------|------|----------|
| LLM 生成的代码/XML 有语法错误 | LLM 不是编译器，经常犯小错 | 自动修复 + 重试机制（参考 next-ai-draw-io 的 validateAndFixXml） |
| 布局丑（重叠、间距不均） | LLM 对空间关系理解弱 | 约束布局范围 + 网格对齐 + 视觉验证（参考 VisPainter 的原子操作+截图反馈） |
| 复杂图超出 LLM 上下文窗口 | XML/代码太长 | 分块生成 + 增量编辑（参考 next-ai-draw-io 的 append_diagram） |
| 风格不一致 | 每次生成独立决策 | 统一 style 模板 + 设计系统约束（参考 scipilot 的 setup_style） |
| 不知道画什么图 | 缺乏领域知识 | 选图决策框架 + 数据剖析（参考 scipilot 的 chart_selection.md） |
| 渲染环境依赖 | 需要特定库/服务 | 容器化部署（Docker）或选择轻量渲染方案（CairoSVG） |

### 7.4 推荐的学习路径

1. **先玩 next-ai-draw-io**：本地部署（`npm install && npm run dev`），用自然语言画几张图，观察它生成的 XML 长什么样。

2. **读 drawio-mcp 的 xml-reference.md**：这是最权威的 draw.io XML 参考，理解 mxCell、style、geometry 的关系。

3. **读 scipilot-figure-skill 的 SKILL.md**：理解"先思考后绘图"的工作流设计，学习代码模板和自检闭环。

4. **用 draw.io 手动画几张图**：打开 https://app.diagrams.net，手动画一个流程图，然后"Extras → Edit Diagram"看它的 XML。这是理解 draw.io XML 最快的方式。

5. **写一个最小 agent**：用你熟悉的 LLM API，写一个简单的 prompt 让它生成 draw.io XML，然后用 draw.io 打开看效果。迭代改进 prompt。

6. **加入验证和修复**：参考 next-ai-draw-io 的 validateAndFixXml，加入 XML 验证和自动修复。

7. **加入视觉验证**：用 Puppeteer 或 CairoSVG 把 SVG 渲染成 PNG，发给 VLM 检查质量。

### 7.5 不用多模态模型的约束下，你需要注意什么

你提到"不采用多模态模型"，这意味着：

1. **没有视觉验证**：你无法用 VLM 看渲染结果来判断质量。替代方案：
   - 程序化检查（元素是否重叠、是否超出画布、连线是否指向存在的节点）
   - 基于规则的布局评分（对齐度、间距均匀性、对称性）
   - 用户手动确认

2. **没有图像输入**：你无法让用户上传一张参考图然后"照着画"。替代方案：
   - 用户用文字描述
   - 用户提供结构化的 JSON/YAML 描述
   - 从论文文本中自动提取结构

3. **纯文本 LLM 生成 XML/代码**：这完全可行（next-ai-draw-io 的核心就是文本 LLM 生成 XML），但需要更精细的 prompt 约束和更多的 few-shot 示例。

4. **matplotlib 路线受影响最小**：matplotlib 代码生成完全不需要多模态能力，纯文本 LLM 就能做好。

### 7.6 值得深入研究的子问题

1. **布局算法**：如何让 LLM 生成的元素不重叠？可以考虑：
   - 约束生成（在 prompt 中规定网格）
   - 后处理（生成后用算法调整位置）
   - 引入 ELK 等自动布局引擎

2. **设计系统**：如何保证风格一致？
   - 定义一套固定的色板、字体、间距规范
   - 在 prompt 中注入设计系统约束
   - 用模板而非自由生成

3. **增量编辑**：如何支持"改一下这个框的颜色"？
   - 基于 ID 的增删改操作（参考 next-ai-draw-io 的 edit_diagram）
   - 维护图的当前状态作为上下文

4. **领域特化**：如何画好特定领域的图（如化学反应路径、神经网络架构）？
   - 领域图标库
   - 领域特定的模板
   - 领域知识编码为 skill

---

## 附录：项目链接汇总

| 项目 | GitHub | 类型 |
|------|--------|------|
| next-ai-draw-io | https://github.com/DayuanJiang/next-ai-draw-io | Draw.io XML |
| drawio-mcp | https://github.com/jgraph/drawio-mcp | Draw.io MCP |
| Pegasus | https://github.com/HANsoA-KevinO/Pegasus | Draw.io 科研绘图 |
| scipilot-figure-skill | https://github.com/Haojae/scipilot-figure-skill | matplotlib Skill |
| NaturePanelForge | https://github.com/littlepeachs/NaturePanelForge | 图→代码逆向 |
| LLM4SVG | https://github.com/ximinng/LLM4SVG | SVG 生成（学术） |
| VisPainter | https://github.com/HerzogFL/VisPainter | Visio GUI 操控 |
| Paper2Any | https://github.com/OpenDCAI/Paper2Any | 多格式平台 |
| AutoFigure | https://github.com/ResearAI/AutoFigure | SVG/HTML 布局 |
| w-next-ai-drawio | https://github.com/wangfenghuan/w-next-ai-drawio | 国内二开版 |
| StarVector | https://github.com/joanrod/star-vector | SVG 基础模型 |
