# Codex 学习笔记

## 结构

```
skeleton/       骨架层：一页纸，只有结构，用于快速定位和导航
  project.md        项目整体结构
  agent-loop.md     agent loop 骨架 + 扩展点地图
  customization.md  定制点一览表
  configuration.md  配置分类速查

deep-dive/      细节层：展开某个具体点
  project-overview.md           项目详细概览（crate 表格）
  agent-loop-skeleton-expanded.md  loop 骨架的逐层展开（含代码）
  agent-loop.md                 loop 深度解析文章（含源码分析）
  customization.md              定制机制详解
  customization-article.md      定制指南文章
  configuration.md              完整配置参考
  memory.md                     Memory 功能详解
```

## 分析基准

基于 openai/codex commit `7880414a2`（2026 年 3 月）。
源码在 `codex/` submodule 下（`git submodule update --init` 拉取）。

## 阅读路径

**快速了解**：skeleton/ 全部（4 个文件，每个一页）

**深入某个主题**：skeleton/ 对应文件 → deep-dive/ 对应文件

**写文章**：deep-dive/agent-loop.md 和 deep-dive/customization-article.md 是已成形的文章
