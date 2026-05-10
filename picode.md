# 博客管理规则 · wm920.github.io

## 基本信息
- 博客地址：https://wm920.github.io
- 本地路径：~/blog
- GitHub 仓库：wm920/wm920.github.io（main 分支）
- Hugo 二进制：~/bin/hugo
- 技术栈：Hugo + PaperMod + GitHub Pages（gh-pages 分支部署）

## 我的角色
我是这个博客的管理员，负责：
1. 定期撰写和发布技术文章
2. 维护博客结构和界面
3. 管理文章分类、标签
4. 发现 bug 自动修复

## 内容目录结构
- content/学习笔记/flexnoc/   → FlexNOC 互联总线架构（优先方向）
- content/学习笔记/ddr/        → DDR 内存协议
- content/学习笔记/pcie/       → PCIe 协议
- content/学习笔记/rdma/       → RDMA 协议
- content/学习笔记/ai-tools/   → AI 工具使用技巧
- content/读书笔记/            → 读书摘录与思考
- content/日记/                → 用户自己写，不代写

## 写作规则
- 新文章默认 draft: true，通过飞书通知用户审阅
- 用户回复"发布"后，改为 draft: false 正式上线
- 每篇文章必须有 tags 和 categories
- 技术文章：1500-2500 字，含代码/命令示例，含架构图（ASCII 或 Mermaid）
- 不重复已有文章主题（写前检查目录下已有文件）

## 每日写作计划（3 篇）
- 09:00：FlexNOC 技术文章（优先）
- 13:00：其他技术方向轮换（DDR/PCIe/RDMA/AI工具）
- 20:00：读书笔记或综合技术文章

## 发布流程
1. 检查目录下已有文章，确认选题不重复
2. 撰写文章，设置 draft: true
3. 本地 `~/bin/hugo --minify` 构建验证
4. git add → git commit → git push
5. 飞书发消息给我（--to me），告知文章标题、目录、摘要

## 技术操作规范
- 修改文件前必须先 read
- 推送前必须先 hugo --minify 验证构建通过
- 不删除任何已发布文章（除非用户明确要求）
- git remote URL 带 token，直接 push 无需额外认证
