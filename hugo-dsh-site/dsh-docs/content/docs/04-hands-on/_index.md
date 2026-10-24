---
title: DSH 本机实操教程
linkTitle: 本机实操教程
description: 2026-10-24 Windows 本机实测：从 Node 安装到 DSH Web UI、配置与插件、Jet Hub、排障升级与验收。
type: docs
icon: fa-solid fa-screwdriver-wrench
weight: 4
sidebar_root_for: self
sidebar_root_link_self: true
cascade:
  type: docs
  footer_style: slim
---

本分类来自根目录 `README.md` 的 2026-10-24 本机实测教程，按主题拆成四篇。四篇合起来覆盖从「新机器装 Node」到「升级 Jet Hub 并验收」的完整链路；概念与架构细节仍看 [DeepSeek Harness 指南](/docs/03-dsh/)，这里只保留可复现的操作与判据。

> 原始教程标题：**Windows 本地 DSH 启动指南**（适用 Windows 10/11；命令默认在 PowerShell 中执行，Git Bash 差异单独标注）。原文的目录表由本页的四篇导航替代，正文按原章节顺序保留。

## 原文目录与新位置

| 原文章节 | 现在读哪篇 |
|----------|------------|
| 0. 新人小结：3 分钟跑起来 | [安装与首次启动](/docs/04-hands-on/安装启动/安装与首次启动/) |
| 1. 前置条件与版本基线 | [安装与首次启动](/docs/04-hands-on/安装启动/安装与首次启动/) |
| 2. 安装 Node.js 与 npm | [安装与首次启动](/docs/04-hands-on/安装启动/安装与首次启动/) |
| 3. 安装 DSH | [安装与首次启动](/docs/04-hands-on/安装启动/安装与首次启动/) |
| 4. 首次启动与验收 | [安装与首次启动](/docs/04-hands-on/安装启动/安装与首次启动/) |
| 5. 配置面 | [配置、插件与日常运维](/docs/04-hands-on/配置与日常/配置插件与日常运维/) |
| 6. 插件面 | [配置、插件与日常运维](/docs/04-hands-on/配置与日常/配置插件与日常运维/) |
| 7. 典型场景（第一批 3 个） | [配置、插件与日常运维](/docs/04-hands-on/配置与日常/配置插件与日常运维/) |
| 8. 日常运维 | [配置、插件与日常运维](/docs/04-hands-on/配置与日常/配置插件与日常运维/) |
| 9. 排障手册 | [排障、验收与附录](/docs/04-hands-on/排障与验收/排障验收与附录/) |
| 10. 验收清单与未验证项 | [排障、验收与附录](/docs/04-hands-on/排障与验收/排障验收与附录/) |
| 11. Jet Hub（账号枢纽）与安装方案 | [Jet Hub 安装与升级](/docs/04-hands-on/JetHub与升级/JetHub安装与升级/) |
| 12. 升级到最新版（DSH + Jet Hub） | [Jet Hub 安装与升级](/docs/04-hands-on/JetHub与升级/JetHub安装与升级/) |
| 附录 A. 命令速查 | [排障、验收与附录](/docs/04-hands-on/排障与验收/排障验收与附录/) |
| 附录 B. 追加内容约定 | [排障、验收与附录](/docs/04-hands-on/排障与验收/排障验收与附录/) |
| FAQ | [排障、验收与附录](/docs/04-hands-on/排障与验收/排障验收与附录/) |
| 变更记录 | [排障、验收与附录](/docs/04-hands-on/排障与验收/排障验收与附录/) |

## 知识板块

- **安装启动** — 前置条件、Node/npm、npx 与全局安装、首次启动、token URL 验收与停止。
- **配置与日常** — DSH_HOME、模型与凭据、插件安装与 allowBuilds、三个典型场景、停止备份与日志。
- **排障与验收** — 症状 → 原因 → 处理、机器可检清单、命令速查、FAQ、追加约定与变更记录。
- **JetHub 与升级** — Jet Hub 与 `dsh-providers-hub` 的关系、三种安装方案、真实报错、allowBuilds 键轮换、DSH 与插件升级回滚。

> 教程中的版本、端口、路径都是 2026-10-24 的本机快照；升级后先以 `dsh --version` 与当前 profile 文件为准。
