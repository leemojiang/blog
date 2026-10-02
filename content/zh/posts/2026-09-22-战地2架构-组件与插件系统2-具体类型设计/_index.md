---
title: "战地 2 架构、组件与插件系统 2：具体类型设计"
date: 2026-09-22
draft: false
math: true
layout: single
ShowToc: false
tags:
  - "战地 2"
  - "游戏架构"
  - "组件系统"
  - "插件架构"
  - "UI"
categories:
  - "软件架构"
description: >-
  按讨论顺序整理具体类型设计：从输入、Control Surface、Camera 与 HUD，延伸到能力接口、事件流、UI 数据流和 Surface 生命周期。
---

本组笔记严格按照原始讨论的时间顺序，逐个整理后续 11 个问题。前半部分讨论 Input、Control Surface、Camera、HUD、Ammo 和 Event；最后一组连续整理 DSH UI、游戏 UI 数据流以及 Surface 的组装、数据和 Present。

## 文件目录

1. [输入、武器控制与其他特殊类型]({{< relref "01-输入-武器控制与特殊类型.md" >}})
2. [Control Surface 与 Aim 组合]({{< relref "02-Control-Surface与Aim组合.md" >}})
3. [Camera 与 Zoom 的单例 Runtime]({{< relref "03-Camera与Zoom的单例Runtime.md" >}})
4. [HUD 与动态 Runtime]({{< relref "04-HUD与动态Runtime.md" >}})
5. [IAmmoStatus 与能力接口]({{< relref "05-IAmmoStatus与能力接口.md" >}})
6. [事件推送、Pull 与能力接口]({{< relref "06-事件推送-Pull与能力接口.md" >}})
7. [DSH UI 调研问题]({{< relref "07-DSH-UI调研问题.md" >}})
8. [DSH UI Plugin 的调研结果]({{< relref "08-DSH-UI调研结果.md" >}})
9. [DSH UI 与游戏数据流]({{< relref "09-DSH-UI与游戏数据流.md" >}})
10. [Surface 的组装与动态生命周期]({{< relref "10-Surface的组装与动态生命周期.md" >}})
11. [Surface 的数据、逻辑与 Present]({{< relref "11-Surface的数据逻辑与Present.md" >}})

## 整理说明

本次重排不再把多个问题合并到同一篇文件。第 7～11 篇构成最后的 DSH UI → 游戏 UI → Surface 讨论链，并保留了原始追问没有独立回答的情况。
