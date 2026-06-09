<p align="center">
  <h1 align="center">🧭 Mentor Skills</h1>
  <p align="center">想清楚 · 做清楚 · 说清楚<br>Think Clearly · Execute Clearly · Communicate Clearly</p>
</p>

<p align="center">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT License" /></a>
  <img src="https://img.shields.io/badge/status-active-success.svg" alt="Active" />
  <img src="https://img.shields.io/badge/PRs-welcome-brightgreen.svg" alt="PRs Welcome" />
</p>

<p align="center">
  <a href="#-中文">🇨🇳 中文</a> · <a href="#-english">🇬🇧 English</a>
</p>

---

# 🇨🇳 中文

## 📋 目录

- [🎯 理念](#-理念)
- [🧩 技能速览](#-技能速览)
- [🚀 快速使用](#-快速使用)
- [📁 项目结构](#-项目结构)
- [📐 设计原则](#-设计原则)
- [📄 许可证](#-许可证)
- [🙏 致谢](#-致谢)

---

## 🎯 理念

技术人员成长中常见的三个瓶颈：**想不清楚、做不清楚、说不清楚**。

Mentor Skills 围绕这三个痛点，提供一套结构化、可执行的技能框架，帮助技术人员建立系统化的思维和工作方式。

| 阶段 | 目标 | 对应技能 |
|------|------|----------|
| 🧠 **想清楚** | 需求评审、方案拷问，暴露设计盲点 | mentor-challenge |
| 🔧 **做清楚** | 有信源支撑的最佳实践和问题排查 | mentor-guide |
| 📝 **说清楚** | 结构化复盘，通过追问把经验转化为可复用知识 | mentor-reflect |

---

## 🧩 技能速览

### 1. mentor-challenge — 方案拷问

当需要评审需求、PRD 或技术方案时，通过 **6D 拷问框架** 逐一追问目标、逻辑、方案、风险、成本和替代方案，帮你发现被忽略的关键问题。

> **一句话**：给你的方案做一次全面体检。

### 2. mentor-guide — 技术指南

当遇到 Bug、报错、性能问题或技术选型困惑时，遵循 **先搜索再回答、先诊断再开方** 的原则，提供有信源可追溯的技术建议和最佳实践。

> **一句话**：有据可查的技术导航。

### 3. mentor-reflect — 复盘提炼

当一个技术决策、问题处理或项目经验需要沉淀时，使用 **BSSR 四阶段模型**（背景 → 挣扎 → 解决方案 → 反思提炼），把零散经验抽象为通用原则。

> **一句话**：每一次经历都不白过。

---

## 🚀 快速使用

在支持 Skill 系统的 AI 编程助手中启用本技能包，然后直接向 AI 描述你的需求：

| 场景 | 示例话术 |
|------|----------|
| 方案评审 | "帮我 review 一下这个设计方案" |
| 问题排查 | "服务启动报错，排查一下" |
| 技术选型 | "FlinkSQL 调优最佳实践？" |
| 复盘总结 | "帮我复盘一下这次日常事故" |

AI 会自动根据你的描述路由到合适的子技能，无需手动切换。

---

## 📁 项目结构

```
mentor-skills/
├── SKILL.md                          # 根技能定义（路由入口）
├── mentor-challenge/                 # 方案拷问子技能
│   ├── SKILL.md
│   └── references/
│       └── questioning-framework.md
├── mentor-guide/                     # 技术指南子技能
│   ├── SKILL.md
│   ├── references/                   # 调试框架、搜索策略、信源分级
│   ├── examples/                     # 服务发现选型、OOM 排查等示例
│   └── templates/                    # 最佳实践模板、调试模板
└── mentor-reflect/                   # 复盘提炼子技能
    ├── SKILL.md
    ├── references/                   # BSSR 模型、引导问题库、原则提炼指南
    ├── examples/                     # Redis 大 Key 事故复盘示例
    └── templates/                    # 复盘报告模板
```

---

## 📐 设计原则

1. **先问再答** — 理解场景后再输出，不盲目回答
2. **结构为先** — 所有输出遵循既定框架，保持一致性
3. **信源可溯** — 每条结论标注可信度等级和来源
4. **授人以渔** — 不仅给答案，更给方法和框架
5. **只读不写** — 不主动修改用户代码（方案拷问模式）

---

## 📄 许可证

本项目基于 [MIT License](LICENSE) 开源。

Copyright © 2026 mentor-skills

---

## 🙏 致谢

本项目的设计和实现受到以下开源技能包的启发，感谢所有开源社区的贡献者：

- [nature-skills](https://github.com/nature-skill)
- [myyyyyyz/master-skill](https://github.com/myyyyyyz/master-skill)
- [byted-volcengine-flink](https://github.com/byted-volcengine-flink)
- [grill-me](https://github.com/mattpocock/skills)

---

# 🇬🇧 English

## 📋 Table of Contents

- [🎯 Philosophy](#-philosophy)
- [🧩 Skills at a Glance](#-skills-at-a-glance)
- [🚀 Quick Start](#-quick-start)
- [📁 Project Structure](#-project-structure)
- [📐 Design Principles](#-design-principles)
- [📄 License](#-license)
- [🙏 Acknowledgements](#-acknowledgements)

---

## 🎯 Philosophy

Three common bottlenecks in a developer's growth: **unclear thinking, unclear execution, unclear communication**.

Mentor Skills revolves around these three pain points, providing a structured, actionable skill framework that helps developers build systematic ways of thinking and working.

| Stage | Goal | Skill |
|-------|------|-------|
| 🧠 **Think Clearly** | Review requirements & designs, expose blind spots | mentor-challenge |
| 🔧 **Execute Clearly** | Best practices & debugging with credible sources | mentor-guide |
| 📝 **Communicate Clearly** | Structured retrospectives, turn experience into knowledge | mentor-reflect |

---

## 🧩 Skills at a Glance

### 1. mentor-challenge — Design Review

When you need to review requirements, PRDs, or technical designs, the **6D Questioning Framework** probes goals, logic, design, risks, costs, and alternatives — one by one — to uncover overlooked issues.

> **In a word**: A full health check for your design.

### 2. mentor-guide — Technical Guidance

When you encounter bugs, errors, performance issues, or technology choices, follow the principle of **research first, diagnose before prescribe** — delivering traceable technical advice backed by credible sources.

> **In a word**: A reliable technical compass.

### 3. mentor-reflect — Retrospective & Refinement

When a technical decision, incident, or project experience needs to be consolidated, use the **BSSR four-stage model** (Background → Struggle → Solution → Reflection) to distill scattered experiences into reusable principles.

> **In a word**: Make every experience count.

---

## 🚀 Quick Start

Enable this skill pack in an AI coding assistant that supports the Skill system (e.g., Trae IDE), then simply describe your needs:

| Scenario | Example Prompt |
|----------|---------------|
| Design Review | "Review this design proposal for me" |
| Debugging | "My service fails to start, help me debug" |
| Tech Selection | "Best practices for microservice gateway?" |
| Retrospective | "Help me retrospect this incident" |

The AI will automatically route your request to the appropriate sub-skill — no manual switching needed.

---

## 📁 Project Structure

```
mentor-skills/
├── SKILL.md                          # Root skill definition (router)
├── mentor-challenge/                 # Design review sub-skill
│   ├── SKILL.md
│   └── references/
│       └── questioning-framework.md
├── mentor-guide/                     # Technical guidance sub-skill
│   ├── SKILL.md
│   ├── references/                   # Debugging framework, search strategies, source credibility
│   ├── examples/                     # Service discovery, OOM debugging, etc.
│   └── templates/                    # Best practice & debugging templates
└── mentor-reflect/                   # Retrospective sub-skill
    ├── SKILL.md
    ├── references/                   # BSSR model, guiding questions, principle refinement
    ├── examples/                     # Redis big-key incident review
    └── templates/                    # Review report template
```

---

## 📐 Design Principles

1. **Ask before answer** — Understand the context before responding
2. **Structure first** — All outputs follow defined frameworks for consistency
3. **Source traceability** — Every conclusion is tagged with credibility level and source
4. **Teach to fish** — Provide not just answers, but methods and frameworks
5. **Read-only mode** — Never modify user code without explicit request (challenge mode)

---

## 📄 License

This project is open-sourced under the [MIT License](LICENSE).

Copyright © 2026 mentor-skills

---

## 🙏 Acknowledgements

The design and implementation of this project were inspired by the following open-source skill packs. Thanks to all open-source contributors:

- [nature-skills](https://github.com/nature-skill)
- [myyyyyyz/master-skill](https://github.com/myyyyyyz/master-skill)
- [byted-volcengine-flink](https://github.com/byted-volcengine-flink)
- [grill-me](https://github.com/mattpocock/skills)

---

<p align="center">
  Made by <a href="https://melodylx666.github.io/lx-bigdata/">lixiao</a>
</p>
