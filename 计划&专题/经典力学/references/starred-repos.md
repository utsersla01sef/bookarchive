# 星标清单 · Feynman 学习法相关开源项目

> 检索日期：2026-08-01
> 用途：为 cm-tutor skill 提供方法论参考与特性借鉴

---

## ★ 推荐借鉴

### 1. suvojit-0x55aa / skill.md — Feynman method to teach deep understanding

- **链接**：https://gist.github.com/suvojit-0x55aa/61053bd8c0b79a1b4f208272778e0247
- **形态**：Claude Code Skill (Markdown)
- **创建时间**：2026-02-19

**核心方法论：4-Pass 渐进式学习**

| Pass | 目标 | 产出 |
|------|------|------|
| Pass 1 概览 | 能用一句话解释核心思想 | 现实类比 + 学习目标确认 |
| Pass 2 架构 | 能凭记忆画出结构图 | 目录/概念关系地图 |
| Pass 3 细节 | 理解关键机制 | 逐模块深讲 + 检查点 |
| Pass 4 迁移 | 能应用到新场景 | 扩展练习 + 教学挑战 |

**7 条教学原则**
1. 术语首现必定义
2. 类比优先
3. 先讲为什么再讲是什么
4. 具体先于抽象
5. 以提问代替灌输
6. 渐进复杂度
7. "从零构建"框架

**反模式清单**：代码墙、随意用术语、一次性倾倒、假设背景知识、无互动灌输、跳过"为什么"

**借鉴要点** → cm-tutor 的检查点门控机制 + 反模式约束

---

## 其他参考项目

### 2. chr1stophe / learn SKILL.md — 结构化自学 skill

- **链接**：https://gist.github.com/chr1stophe/22c62c92a0129a5f8f0c72263e3674ea
- **形态**：Claude Code Skill (Markdown)
- **Stars**：3

**方法论**：6 阶段 14 步，基于循证学习科学
- 间隔检索 (Roediger & Karpicke, 2006)
- 生成效应 (Slamecka & Graf, 1978)
- 刻意练习 (Ericsson et al., 1993)
- 交错练习 (Rohrer & Taylor, 2007)
- 精细审问 (Pressley et al., 1987)

**文件结构**（每个学习轨道独立目录）：
```
personal/learning/{topic-slug}/
├── progress.md      # 阶段状态 + 会话日志
├── sources.md       # 资料清单
├── guidebook.md     # 综合参考
├── concept-map.md   # 核心概念与争议
├── builds/          # 实践项目
└── artifacts/       # 产出文件
```

**路由模式**：无参数=列表 / 匹配=恢复 / 新主题=新建

**借鉴要点** → cm-tutor 的 progress.md 会话日志结构 + 间隔重复调度

---

### 3. hemanth / feynman-learning — AI 费曼学习 CLI

- **链接**：https://github.com/hemanth/feynman-learning
- **PyPI**：https://pypi.org/project/feynman-learning/
- **形态**：Python 包 (MIT, Python ≥3.7)
- **发布**：2025-06-14

**5 阶段费曼流程**：Study → Explain → Identify Gaps → Simplify → Review

**CLI 命令**：
```bash
feynman-learning learn                # 交互式学习
feynman-learning stats                # 统计
feynman-learning progress "概念"       # 单概念进度
feynman-learning export -o report.json # 导出报告
feynman-learning config show          # 配置
```

**特性**：支持 OpenAI 及兼容端点（vLLM/Ollama 本地模型），配置存于 `~/.feynman.conf`

**借鉴要点** → cm-tutor 的统计/导出能力 + 多模型兼容思路

---

### 4. jialingxiao / obsidian-feynman-learning — Obsidian 插件

- **链接**：https://github.com/jialingxiao/obsidian-feynman-learning
- **形态**：Obsidian 社区插件 (MIT, TypeScript)
- **下载量**：496

**4 步引导工作流**：选概念 → 简单解释 → 识别缺口 → 简化与类比

**3 维 AI 评分体系**：
| 维度 | 说明 |
|------|------|
| 语言简洁 | 能否用通俗语言表达 |
| 核心机制 | 是否抓住本质机制 |
| 举例说明 | 能否给出恰当类比 |

**四级掌握度**：初识 → 理解 → 掌握 → 精通（仅 AI 评估通过才晋级）

**间隔重复**：
- 默认间隔 1 / 7 / 30 天
- 全 ✓ → 间隔 ×1.3（延长）
- 有 △ → 间隔 ×0.6（缩短）
- 失败 → 次日重试

**仪表盘**：连续打卡、概念总数、待复习数、掌握度分布、活动热力图、周报

**借鉴要点** → cm-tutor 的三维评分升级 + 间隔重复算法 + 可视化仪表盘

---

## cm-tutor 借鉴融合方案

| 当前 cm-tutor 特性 | 可增强方向 | 来源项目 |
|-------------------|-----------|---------|
| correct/partial/incorrect 评分 | 升级为三维评分（直觉/推导/图像） | obsidian 插件 |
| mistakes.json 错题分类 | 加入间隔重复调度（1/7/30天） | chr1stophe + obsidian |
| progress.json 进度 | 增加 会话日志 + 概念地图 | chr1stophe |
| 课时模板输出 | 加入检查点门控 + 反模式约束 | suvojit |
| 无统计导出 | 增加 export 能力 | feynman-learning |

---

## 其他相关资源（未深入）

- **NathBabs / Feynman_Prompt.md**：费曼学习教练 prompt 模板
  - https://gist.sharingeye.com/NathBabs/1ea2dce77cd4c287f253302e550a858e
- **skills.rest / feynman**：社区 skill，结构化费曼学习循环
  - https://skills.rest/skill/feynman （源码 https://github.com/Abkr1/liza/tree/main/skills/feynman）
- **Lee985-cmd / AI-30-Day-Challenge**：AI 入门 30 天挑战（含费曼输出环节）
  - https://github.com/Lee985-cmd/AI-30-Day-Challenge
- **Lee985-cmd / algorithm-30days**：算法 30 天（费曼学习法风格）
  - https://github.com/Lee985-cmd/algorithm-30days
