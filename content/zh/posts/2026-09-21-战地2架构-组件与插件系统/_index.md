---
title: "战地 2 架构、组件与插件系统"
date: 2026-09-21
draft: false
math: true
layout: single
ShowToc: false
tags:
  - "战地 2"
  - "游戏架构"
  - "组件系统"
  - "插件架构"
  - "Unity"
categories:
  - "软件架构"
description: >-
  从 Battlefield 2 与 Refractor 2 的内容组织边界出发，逐步讨论 Unity 集成、组件语义、Runtime、Capability、Runtime View 和接口隔离。
---

本组笔记整理“分析战地 2 架构”这段讨论，保留原始问答结构。讨论从 Battlefield 2 / Refractor 2 的内容组织方式开始，逐步延伸到 Unity 集成、Component 语义、Hexagonal Architecture、DSH/Cordis 式 Runtime、Capability、Runtime View，以及 Interface Segregation Principle（ISP）。

每个问题单独成篇，尽量保留原始提问、回答中的代码和图示。为了避免把讨论过度概括，事实核验、概念修正和补充内容会放在对应文件的“整理说明”中。

## 文件目录

1. [战地 2 架构的核心边界](./01-战地2架构的核心边界.md)
2. [Unity 中的低成本实现路径](./02-Unity中的低成本实现路径.md)
3. [组件组合与依赖治理](./03-组件组合与依赖治理.md)
4. [Hexagonal Architecture 与组件语义](./04-Hexagonal-Architecture与组件语义.md)
5. [从 Plugin 到 Runtime 的组件简化](./05-从Plugin到Runtime的组件简化.md)
6. [Super Runtime 与 Runtime View](./06-Super-Runtime与Runtime-View.md)
7. [ISP 在游戏 Runtime 中的应用](./07-ISP在游戏Runtime中的应用.md)

## 整理顺序

本组 7 个问题已经按照原始对话的时间顺序整理完成。之后如果继续讨论相关问题，再将新增问答追加到对应章节，或创建新的问题文件。
