# My Claude Code Skills Collection

这是我个人的 Claude Code Skills 集合，包含了我在使用 Claude Code 过程中积累的各种专业技能和工作流。

## 📦 包含的 Skills

### 🌐 dynamic-web-extractor
通用的动态网页内容提取工具，支持：
- Twitter/X 推文提取
- Reddit 帖子提取
- Medium 文章提取
- LinkedIn 动态提取
- 任何 JavaScript 驱动的 SPA 网站

**触发条件**：当 WebFetch 失败或用户提供动态网站 URL 时自动使用

### 📄 其他 Skills
- `pdf` - PDF 文件处理（提取、合并、创建）
- `xlsx` - Excel 电子表格处理
- `csv-data-summarizer` - CSV 数据分析和可视化
- `test-driven-development` - TDD 开发工作流
- `root-cause-tracing` - 错误根因追踪
- `finishing-a-development-branch` - 开发分支完成工作流
- `using-git-worktrees` - Git worktree 隔离工作空间
- `skill-creator` - 创建新 Skill 的指南
- `artifacts-builder` - 复杂 React 组件构建
- `canvas-design` - 视觉设计和艺术创作
- `theme-factory` - Artifact 主题样式

## 🚀 如何使用

### 在新电脑上安装

```bash
# 克隆这个仓库到 Claude Code 配置目录
git clone <你的仓库URL> ~/.config/claude-code
```

### 更新 Skills

```bash
cd ~/.config/claude-code
git pull
```

### 贡献新 Skill

```bash
cd ~/.config/claude-code
# 创建新 Skill 目录
mkdir skills/my-new-skill
# 编写 SKILL.md
# ...
git add .
git commit -m "Add: my-new-skill"
git push
```

## 📖 Skills 工作原理

Skills 是 Claude Code 的知识增强机制：

1. **待机状态**：Claude 读取所有 Skills 的 `description`，记住触发条件
2. **激活状态**：当任务匹配某个 Skill 的描述时，完整加载该 Skill 内容
3. **执行指导**：Claude 按照 Skill 中的最佳实践和代码模板执行任务

## 🔧 自定义配置

如果你 fork 了这个仓库，可以：
- 删除不需要的 Skills
- 修改 Skill 内容以适应你的工作流
- 添加你自己的专业领域 Skills

## 📝 许可证

部分 Skills 来自官方或第三方，具有各自的许可证（见各 Skill 目录下的 LICENSE.txt）。

自定义 Skills（如 `dynamic-web-extractor`）为个人创建，可自由使用和修改。

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

---

**创建日期**：2025-11-19
**最后更新**：2025-11-19
**Skills 数量**：12
**总大小**：~6.1 MB
