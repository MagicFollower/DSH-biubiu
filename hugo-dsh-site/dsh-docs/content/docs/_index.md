---
title: 文档
linkTitle: 文档
description: 三章认知 + 一章实操：DeepSeek 模型底座、终端 Agent 工具浪潮、DSH 概念指南，以及 Windows 本机实测教程。
type: docs
icon: fa-solid fa-book
sidebar_root_for: self
sidebar_root_link_self: true
menus:
  main:
    identifier: docs
    weight: 20
cascade:
  type: docs
  footer_style: slim
---

本站围绕 **DeepSeek Harness（DSH）** 展开，前三章按读者认知顺序讲清来路与机制：**DeepSeek 模型底座**从何而来、DSH 诞生前的 **终端 Agent 工具浪潮**、以及 **DSH 指南**本体的架构、CLI、Web GUI 与插件生态。第四章是 2026-10-24 的 **Windows 本机实测教程**，把安装、配置、插件、Jet Hub、排障与升级按操作顺序串起来。

- [认识 DeepSeek](/docs/01-deepseek/)
- [终端 Agent 工具演进](/docs/02-agent-tools/)
- [DeepSeek Harness 指南](/docs/03-dsh/)
- [DSH 本机实操教程](/docs/04-hands-on/)
{.cards}

<!--
落地清单：
1. 链接条目数必须等于 content/docs 下一级分类目录数，顺序与 weight 一致；
2. 每行形如 `- [显示名](/docs/NN-目录名/)`，末尾 `{.cards}` 决定渲染成卡片而非列表，别删；
3. menus.main 让「文档」出现在顶栏，identifier 全站唯一。
-->
