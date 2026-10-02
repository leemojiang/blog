---
title: "战地 2 架构、组件与插件系统 2：具体类型设计"
date: 2026-09-22
draft: false
math: true
layout: single
ShowToc: true
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

1. [输入、武器控制与其他特殊类型](#chapter-01)
2. [Control Surface 与 Aim 组合](#chapter-02)
3. [Camera 与 Zoom 的单例 Runtime](#chapter-03)
4. [HUD 与动态 Runtime](#chapter-04)
5. [IAmmoStatus 与能力接口](#chapter-05)
6. [事件推送、Pull 与能力接口](#chapter-06)
7. [DSH UI 调研问题](#chapter-07)
8. [DSH UI Plugin 的调研结果](#chapter-08)
9. [DSH UI 与游戏数据流](#chapter-09)
10. [Surface 的组装与动态生命周期](#chapter-10)
11. [Surface 的数据、逻辑与 Present](#chapter-11)

{{< include-chapter "01-输入-武器控制与特殊类型.md" "chapter-01" "输入、武器控制与其他特殊类型" >}}

{{< include-chapter "02-Control-Surface与Aim组合.md" "chapter-02" "Control Surface 与 Aim 组合" >}}

{{< include-chapter "03-Camera与Zoom的单例Runtime.md" "chapter-03" "Camera 与 Zoom 的单例 Runtime" >}}

{{< include-chapter "04-HUD与动态Runtime.md" "chapter-04" "HUD 与动态 Runtime" >}}

{{< include-chapter "05-IAmmoStatus与能力接口.md" "chapter-05" "IAmmoStatus 与能力接口" >}}

{{< include-chapter "06-事件推送-Pull与能力接口.md" "chapter-06" "事件推送、Pull 与能力接口" >}}

{{< include-chapter "07-DSH-UI调研问题.md" "chapter-07" "DSH UI 调研问题" >}}

{{< include-chapter "08-DSH-UI调研结果.md" "chapter-08" "DSH UI Plugin 的调研结果" >}}

{{< include-chapter "09-DSH-UI与游戏数据流.md" "chapter-09" "DSH UI 与游戏数据流" >}}

{{< include-chapter "10-Surface的组装与动态生命周期.md" "chapter-10" "Surface 的组装与动态生命周期" >}}

{{< include-chapter "11-Surface的数据逻辑与Present.md" "chapter-11" "Surface 的数据、逻辑与 Present" >}}

## 整理说明

本次重排不再把多个问题合并到同一篇文件。第 7～11 篇构成最后的 DSH UI → 游戏 UI → Surface 讨论链，并保留了原始追问没有独立回答的情况。
