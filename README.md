# 智能教材 V1.0

一个基于 AI 的教材学习辅助平台，集教材阅读、思维导图、知识管理和 AI 助教于一体。采用纯前端单文件架构，无需后端服务，打开 HTML 即可使用。

## 在线访问

https://dreamgitdeep.github.io/-/

## 功能特性

- **教材管理**：上传 MD 格式教材，按二级/三级标题自动解析章节，支持多本教材切换
- **思维导图**：可视化笔记脑图，支持节点拖拽、增删、缩放平移，多种主题和框架模板
- **知识点管理**：AI 自动提取教材核心知识点，支持手动编辑和检索
- **AI 助教**：问答、引导、考核三种模式，基于教材内容进行 RAG 增强对话
- **模拟考核**：卡片式答题流程，AI 自动出题和评分，支持学习报告导出 PDF
- **主题定制**：多种主题和字号可选，数据本地持久化

## 技术架构

### 整体架构

采用**纯前端单文件（SPA）**架构，所有 HTML、CSS、JavaScript 集中在一个 `.html` 文件中，无构建工具、无框架依赖，打开即用。

```
教材智能体V1.0.html
├── <head>          CDN 引入外部库
├── <style>         全局样式（~350 行）
├── <body>          HTML 结构
│   ├── 顶部导航栏
│   ├── 三栏面板（教材 | 脑图 | AI助教）
│   ├── 模态框（框架选择、自定义框架、学习报告）
│   └── Toast 提示
└── <script>        JavaScript 逻辑（~2200 行）
    ├── 教材解析与管理
    ├── 脑图渲染与交互
    ├── AI 对话与 RAG
    ├── 考核系统
    ├── 学习报告
    └── 主题与设置
```

### 外部依赖（CDN）

| 库 | 版本 | 用途 |
|---|---|---|
| [Tailwind CSS](https://tailwindcss.com/) | CDN | 工具类 CSS，快速布局 |
| [Font Awesome](https://fontawesome.com/) | 6.0.0-beta3 | 图标库 |
| [Marked.js](https://marked.js.org/) | 9.1.6 | Markdown → HTML 渲染 |
| [MathJax](https://www.mathjax.org/) | 3.x | LaTeX 数学公式渲染 |
| [html2pdf.js](https://github.com/eKoopmans/html2pdf.js) | 0.10.1 | HTML → PDF 导出 |

### AI 接口

接入 [DeepSeek API](https://platform.deepseek.com/)（兼容 OpenAI Chat Completions 格式）：

- **接口地址**：`https://api.deepseek.com/v1/chat/completions`
- **模型**：`deepseek-chat`
- **请求方式**：同步请求（知识点提取、关键词检索）+ 流式请求（AI 对话、出题、评分）
- **流式解析**：逐行解析 SSE `data:` 前缀的 JSON 增量，实时渲染 Markdown

流式响应处理核心逻辑：
```javascript
// 逐行解析 Server-Sent Events
var lines=chunk.split('\n');
for(var i=0;i<lines.length;i++){
    var line=lines[i].trim();
    if(!line||!line.startsWith('data:'))continue;
    var json=line.substring(5).trim();
    if(json==='[DONE]')break;
    var d=JSON.parse(json);
    var delta=d.choices&&d.choices[0]&&d.choices[0].delta;
    if(delta&&delta.content) fullText+=delta.content;
}
```

## 核心模块详解

### 1. 教材管理模块

#### 数据结构

```javascript
TEXTBOOKS = [
    {
        id: "tb_xxx",           // 唯一 ID
        title: "教材名称",       // 文件名
        sections: [
            {
                id: "sec_xxx",
                title: "章节标题",
                content: "原始 Markdown 内容",
                knowledge: [    // AI 提取的知识点
                    { id: "k_xxx", point: "核心概念", detail: "详细说明" }
                ]
            }
        ]
    }
]
```

#### 解析逻辑

- 按 `## ` 和 `### ` 两级标题拆分 Markdown 文件为章节
- 每个章节保留原始 Markdown 内容，渲染时通过 Marked.js 转为 HTML
- 支持上传多个教材文件，通过顶部下拉切换

#### 持久化

所有教材数据通过 `localStorage` 持久化存储：
- `textbooks_v2`：教材数据 JSON
- `tb_current`：当前选中的教材 ID

### 2. 脑图模块

#### 数据结构

```javascript
mm = {
    id: "root",
    label: "学习笔记",
    content: "学习笔记",
    expanded: true,
    collapsed: false,
    children: [
        {
            id: "node_xxx",
            label: "节点文本",
            content: "节点文本",
            expanded: true,
            collapsed: false,
            children: [],
            x: 100, y: 200,    // 手动定位坐标
            w: 120, h: 36,     // 节点宽高
            lines: ["行1","行2"],// 自动换行结果
            icon: "📚"          // 节点图标
        }
    ]
}
```

#### SVG 渲染

脑图使用**纯 SVG** 渲染，非 Canvas：

- 每个节点是一个 `<g>` 元素，包含圆角矩形 `<rect>` + 多行文本 `<text>`
- 节点间连线使用三次贝塞尔曲线 `<path>`（`C` 命令）
- 支持 4 级层级深度，每级独立配色
- 节点自动换行：根据中英文字符权重计算行宽，遵守中文排版禁则（行首禁逗号、行尾禁左括号等）

#### 画布交互

- **缩放**：鼠标滚轮，以鼠标位置为中心点缩放（CSS `transform: scale() translate()`）
- **平移**：鼠标拖拽空白区域（`mousedown` → `mousemove` → `mouseup`）
- **节点拖拽**：通过节点左侧 `⋮⋮` 手柄拖拽到其他节点，成为其子节点
- **自由拖拽**：拖拽节点圆角矩形可自由调整位置（记录到 `x`, `y` 坐标）
- **教材文字拖入**：教材段落的拖拽手柄可拖入脑图节点，自动创建子分支

#### 主题系统

4 套内置主题，每套包含 4 级节点配色 + SVG 渐变定义：

| 主题 | 风格 | 配色 |
|---|---|---|
| 简约 | 黑白灰 | 纯色填充，干净利落 |
| 彩虹 | 柔和粉彩 | 玫瑰粉、天蓝、薄荷绿、暖杏 |
| 糖果 | 可爱风 | 粉色、薰衣草、薄荷、鹅黄 |
| 自然 | 大地暖色 | 卡其、驼色、橄榄绿、暖灰 |

#### 框架模板

6 种预设思考框架，一键生成脑图骨架：

- 自由框架、大纲式笔记、康奈尔笔记、五维思考法、三遍阅读法、空白框架
- 支持自定义框架（用户编辑节点名称后保存）

### 3. AI 对话模块

#### 三种对话模式

| 模式 | 系统提示词 | 用途 |
|---|---|---|
| 问答模式 | 直接回答问题 | 常规知识问答 |
| 引导模式 | 引导思考，不直接给答案 | 启发式学习 |
| 考核模式 | AI 出题 + 评分 | 模拟考试 |

#### RAG 检索增强

AI 对话前会自动检索教材知识点作为上下文：

```
学生提问 → AI 提取关键词（轻量级调用）
         → 关键词匹配所有知识点（字符串包含匹配）
         → 取 Top 5 相关知识点 + 对应教材原文段落
         → 注入 system prompt 作为上下文
         → AI 基于上下文回答
```

#### Markdown 渲染

AI 回答支持完整的 Markdown 渲染：
- 标题、列表、代码块、表格、引用
- LaTeX 数学公式（行内 `$...$` 和独立 `$$...$$`）
- 渲染后自动调用 `MathJax.typesetPromise()` 刷新公式

### 4. 考核系统

#### 流程

```
输入姓名学号 → 选择难度 → AI 出题（流式）
→ 卡片式逐题作答 → 提交答卷 → AI 评分（流式）
→ 生成学习报告（可导出 PDF）
```

#### 技术实现

- **卡片式 UI**：题目渲染为独立卡片，支持上一题/下一题切换
- **进度条**：顶部显示答题进度百分比
- **计时器**：记录答题总时长，实时显示 mm:ss 格式
- **状态管理**：`quizState` 对象管理考核全流程状态
- **退出保护**：考核进行中切换模式会弹出提示

#### 学习报告

- 显示学生姓名、学号、得分、用时、难度
- 展示 AI 评分详情（Markdown 渲染）
- 嵌入当前脑图快照（SVG 克隆 + viewBox 适配）
- 使用 html2pdf.js 导出为 A4 尺寸 PDF

### 5. PDF 导出

使用 html2pdf.js（内部基于 html2canvas + jsPDF）：

```javascript
var opt = {
    margin: [10, 10, 10, 10],
    filename: '学习报告.pdf',
    image: { type: 'jpeg', quality: 0.98 },
    html2canvas: { scale: 2, useCORS: true, logging: false },
    jsPDF: { unit: 'mm', format: 'a4', orientation: 'portrait' },
    pagebreak: { mode: ['css', 'legacy'] }
};
```

脑图嵌入 PDF 的处理：
- 克隆 SVG 节点，设置 `viewBox` 适配容器宽度
- 隐藏交互元素（工具栏、删除按钮、拖拽手柄等）
- 将 SVG gradient 内联到 `<style>` 标签中，确保 PDF 渲染正确

## 前端技术细节

### CSS 技术

| 技术 | 应用场景 |
|---|---|
| CSS Grid + Flexbox | 三栏面板布局 |
| CSS Transform | 脑图画布缩放/平移、节点高亮动画 |
| CSS Transition | 面板切换、节点悬停、主题过渡 |
| CSS Filter | `drop-shadow` 节点阴影、`backdrop-filter` 模态框毛玻璃 |
| CSS Animation | `fadeInUp` 弹窗动画、`pulse` 加载动画、`spin` 旋转动画 |
| CSS Custom Properties | 主题色通过 `body.className` 切换 |
| `user-select` | 教材区域允许选中文字，容器阻止浏览器原生拖拽搜索 |
| `backdrop-filter` | 模态框背景模糊效果 |
| `@keyframes` | 加载旋转、淡入上移、脉冲等动画 |
| 自定义滚动条 | `::-webkit-scrollbar` 细滚动条样式 |

### JavaScript API

| API | 用途 |
|---|---|
| `fetch()` + `ReadableStream` | DeepSeek API 流式请求 |
| `localStorage` | 教材、脑图、主题、字号、学生信息持久化 |
| Drag and Drop API | 教材文字拖入脑图节点 |
| Selection API | 选中文字弹出浮动操作按钮 |
| `Blob` + `URL.createObjectURL` | SVG 导出为图片 |
| `canvas.toBlob()` | SVG → PNG 转换 |
| `html2pdf.js` | HTML → PDF 导出 |
| `marked.parse()` | Markdown 渲染 |
| `MathJax.typesetPromise()` | LaTeX 公式渲染 |
| `document.createElementNS()` | SVG 元素动态创建 |
| `closest()` | 事件委托中查找最近祖先节点 |
| `getBoundingClientRect()` | 精确定位浮动按钮、计算拖拽坐标 |
| `setInterval` / `setTimeout` | 考核计时器、Toast 自动消失 |

### 拖拽系统

实现了三套独立的拖拽机制：

1. **教材 → 脑图**：教材段落的拖拽手柄（`draggable="true"`）→ 脑图节点 `drop` 事件，创建子分支
2. **节点排序**：脑图节点的 `⋮⋮` 手柄 → 拖拽到其他节点成为子节点
3. **自由拖拽**：直接拖拽节点圆角矩形自由调整位置

防闪烁方案：教材文字拖入脑图时，使用 `mmArea` 全局 `dragover` 事件 + `closest('.mm-g')` 找目标节点，避免子 SVG 元素的 `dragleave` 冒泡导致高亮闪烁。

### 面板拖拽调整

三个面板通过两个拖拽条（`.resize-bar`）调整宽度：

- 拖拽前冻结所有三个面板为像素宽度（防止其他面板联动）
- 拖拽中只调整相邻两个面板的宽度
- `document.addEventListener('mousemove/mouseup')` 全局监听

### 选中文字浮动按钮

教材区域选中文字后弹出浮动「问AI助教」按钮：

- `mouseup` 事件检测选区，通过 `getBoundingClientRect()` 定位
- `mousedown` 时自动移除浮动按钮
- 教材容器 `user-select:none` + 文本 `user-select:text` 阻止浏览器原生拖拽搜索

## 数据持久化

所有数据存储在浏览器 `localStorage` 中：

| Key | 内容 |
|---|---|
| `textbooks_v2` | 教材数据（章节、知识点） |
| `tb_current` | 当前教材 ID |
| `mm_v4` | 脑图数据（节点树、当前框架） |
| `mm_theme` | 脑图主题 ID |
| `theme_v2` | 页面主题 |
| `font_size_v2` | 字号设置 |
| `quizStudentInfo` | 学生姓名和学号 |

## 项目结构

```
教材智能体/
├── 教材智能体V1.0.html    # 主文件（开发用）
├── index.html              # GitHub Pages 部署文件（与主文件同步）
├── README.md               # 项目文档
└── *.md                    # 示例教材文件
```

## 部署到 GitHub Pages

1. 在 GitHub 上创建新仓库
2. 将 `教材智能体V1.0.html` 复制为 `index.html`
3. 推送代码到仓库
4. 进入仓库 Settings > Pages，Source 选择 `main` 分支
5. 等待几分钟后即可通过 `https://用户名.github.io/仓库名/` 访问

## 使用说明

1. 点击「上传教材」导入 MD 格式的教材文件
2. 在左侧教材面板阅读，点击「提取知识点」让 AI 分析章节
3. 在中间脑图面板记录学习笔记，可选择思考框架模板
4. 在右侧 AI 助教面板进行问答、引导学习或模拟考核
5. 点击「学习报告」查看学习统计并导出 PDF

## 许可

MIT License
