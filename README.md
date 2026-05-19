# 教材智能体 V1.0

一个基于 AI 的教材学习辅助平台，集教材阅读、思维导图、知识管理和 AI 对话于一体。

## 功能特性

- **教材管理**：上传 MD 格式教材，自动解析章节，支持多本教材切换
- **思维导图**：可视化笔记脑图，支持节点拖拽、增删、多种框架模板
- **知识点管理**：AI 自动提取教材核心知识点，支持手动编辑和检索
- **AI 助教**：问答、引导、考核三种模式，基于教材内容进行 RAG 增强对话
- **模拟考核**：卡片式答题流程，AI 自动出题和评分，支持学习报告导出 PDF
- **主题定制**：多种主题和字号可选，数据本地持久化

## 在线使用

直接打开 `教材智能体V1.0.html` 即可使用，无需安装任何依赖。

## 部署到 GitHub Pages

1. 在 GitHub 上创建新仓库（如 `teaching-agent`）
2. 将 `教材智能体V1.0.html` 重命名为 `index.html`
3. 推送代码到仓库：
   ```bash
   git init
   git add index.html README.md
   git commit -m "初始提交"
   git branch -M main
   git remote add origin https://github.com/你的用户名/teaching-agent.git
   git push -u origin main
   ```
4. 进入仓库 Settings > Pages，Source 选择 `main` 分支，点击 Save
5. 等待几分钟后即可通过 `https://你的用户名.github.io/teaching-agent/` 访问

## 使用说明

1. 点击「上传教材」导入 MD 格式的教材文件
2. 在左侧教材面板阅读，点击「提取知识点」让 AI 分析章节
3. 在中间脑图面板记录学习笔记，可选择思考框架模板
4. 在右侧 AI 助教面板进行问答、引导学习或模拟考核
5. 点击「学习报告」查看学习统计并导出 PDF

## 技术栈

- 纯前端单文件架构（HTML + CSS + JavaScript）
- DeepSeek API 提供 AI 能力
- Marked.js 渲染 Markdown
- MathJax 渲染数学公式
- html2pdf.js 导出 PDF

## 许可

MIT License
