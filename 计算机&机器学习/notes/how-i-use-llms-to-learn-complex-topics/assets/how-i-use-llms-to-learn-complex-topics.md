# How I use LLMs to learn complex topics

> 作者：Laurentiu Raducu · 2026-08-09
> 原文：https://laurentiugabriel.github.io/blog/articles/how-i-use-llms-to-learn/
> HN 讨论：834 pts / 548 comments

Many engineers I know use generative AI for many functions, like building PoCs, internal tools or dashboards, or even learning new stuff. I personally find the style used by LLMs to explain things difficult to follow. It's just too simplistic and depending on the number of emojis used, a bit annoying too.

While I was analyzing new AI bottlenecks that might slow down data center buildup, I realized there are many aspects of chip production that I do not know. Surfing the web, I asked myself what if there would be a game to get you through the process of building a chip at a fab? For sure learning this way will stick, since you can map concepts with objects within the game. This is when I decided to try it, and it actually turned out really well.

## The flow

Instead of just asking AI to explain a topic, I use the following flow:

- In plan mode (using CC, or OpenCode) I ask a model to build the foundational knowledge for X topic.
- I ask it to review the accuracy of the knowledge base it built in the previous step.
- I proceed asking it to build a simulation of that topic in a low-poly, Rollercoaster Tycoon-like animation. I add some UX elements as well, like the page needs to be visible on both large and small screens, have controls to stop the flow whenever I want etc.
- I then push it to a new repo and enable GitHub Pages for it.

## The result

What you get is a beautiful animation that is 100% accurate and free of hallucinations. For me, this method works a lot better than just reading endless materials that I find on Google, or trying to digest a bulleted list that is spat by a language model.

I've done this specifically for learning chip building and launch it under this website: [ChipTycoon](https://laurentiugabriel.github.io/ChipTycoon/). You get to follow a cart from the moment when sand is collected, to the moment when a chip is finalized and delivered to a data center.

Visually, you can follow the cart and see how it changes too. Since it's low-poly, the details might be missing, but it's still a good indicator for showing how the product changes once it goes through the many steps required in the manufacturing process.

## How to improve it further

Let's say that the low-poly design requires too much imagination to actually visualize what happened to the quartz sand pile after it left the furnace. To transform this into a more realistic representation, you can use my [skill for transforming pictures into 3d objects](https://github.com/LaurentiuGabriel/unreal-game-assets-creation-skill), and map the resulting objects to your simulation. This way you get more accurate design.

Also, you can add challenges to your simulation too. Trying to answer questions about a previous step in the chip manufacturing process will help you retain the knowledge tremendously. Add intuitive puzzles too that will help you learn even better.

Check out what other pages I created:

- [How rocket engines are made](https://laurentiugabriel.github.io/rocket-engine/)
- [How LLMs work](https://laurentiugabriel.github.io/token-town/)
- [How F1 engines are built](https://laurentiugabriel.github.io/engineworks/)
- [How an EUV machine is built](https://laurentiugabriel.github.io/euv-lithography/)

After reading the feedback on the HN post, I decided to create [an awesome Skill.MD file for creating cool animations to learn complex topics](https://github.com/LaurentiuGabriel/learnscape) (learnscape).

---

## 中文摘要

**问题**：LLM 的讲解方式太简化、难以消化，纯读资料记不住。

**他的学习方法（四步流程）**：
1. 用 AI（Claude Code / OpenCode 的 plan 模式）搭建某主题的基础知识库
2. 让 AI 自查知识库的准确性
3. 让 AI 基于知识库做一个低多边形、过山车大亨风格的动画模拟（含响应式布局、暂停控制等 UX 细节）
4. 推到 GitHub Pages 发布

**效果**：得到 100% 准确、无幻觉的可视化动画，比读文档或看 AI 列的要点清单记得牢。实例 [ChipTycoon](https://laurentiugabriel.github.io/ChipTycoon/)——跟着小车从采砂到芯片交付数据中心，全程可视化芯片制造流程。

**进阶**：用图片转 3D 的 skill 提升真实感；给模拟加问答和谜题挑战以强化记忆。

他还做了火箭发动机、LLM 原理（Token Town）、F1 发动机、EUV 光刻机等同类页面，并开源了 [learnscape](https://github.com/LaurentiuGabriel/learnscape)——专门用于"生成学习动画"的 SKILL.md。
