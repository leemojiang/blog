---
title: "渲染管线学习：从概念到 Unity URP 实践"
date: 2026-10-20
draft: false
math: true
layout: single
ShowToc: true
tags:
  - "渲染管线"
  - "Unity"
  - "Shader"
  - "URP"
categories:
  - "图形学"
description: >-
  从实时渲染的基础流程、光照与阴影方案讲起，逐步映射到 Unity 渲染管线，并通过 URP 示例项目拆解 Shader、材质和渲染资源的协作方式。
---
本文按学习顺序分为三章：先建立实时渲染的整体认识，再看 Unity 如何组织渲染流程，最后通过 URP 示例项目把概念落实到资源与 Shader。

## 目录

1. [渲染管线概念：从模型数据到屏幕像素](#chapter-01)
2. [Unity 渲染管线：概念如何映射到 Unity](#chapter-02)
3. [URP 示例项目拆解：把渲染概念落到 Unity 资源](#chapter-03)

{{< include-chapter "01-渲染管线概念.md" "chapter-01" "渲染管线概念：从模型数据到屏幕像素" >}}

{{< include-chapter "02-Unity渲染管线.md" "chapter-02" "Unity 渲染管线：概念如何映射到 Unity" >}}

{{< include-chapter "03-URP示例项目拆解.md" "chapter-03" "URP 示例项目拆解：把渲染概念落到 Unity 资源" >}}

{{< mermaid-init >}}
