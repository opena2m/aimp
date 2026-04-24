# AIMP — AI-to-Machine Protocol

> **Version:** 1.0.0-draft
> **Status:** Draft Specification
> **Date:** 2026-04-24

A standardized protocol set for bridging AI-generated intent and physical-world execution.
The "last mile" that turns AI outputs into tangible, real-world products — safely, cost-aware,
and with closed-loop auditing.

一套用于连接 AI 意图与物理世界执行的标准化协议集合。
让 AI 输出的数字成果安全、可控、带闭环审核地变为现实产品的"最后一公里"。

---

## Repository Layout | 目录结构

```
AIMP/
├── README.md                           This file
├── en/                                 English specification
│   ├── 00-overview.md                  Architecture & design rationale
│   ├── 01-core-protocol.md             Core kernel: lifecycle, APIs, state machine
│   ├── 02-domain-extensions.md         Domain namespaces & extension schemas
│   ├── 03-ai-protocol-integration.md   MCP, A2A, Function Calling bindings
│   ├── 04-security-and-cost.md         Budget control, auth, human-in-the-loop
│   ├── 05-audit-and-telemetry.md       Multimodal feedback & vision audit
│   ├── 06-error-codes.md               Standard error codes & recovery
│   └── 07-scenarios.md                 Reference scenarios
├── zh/                                 中文规范
│   ├── 00-概述.md
│   ├── 01-核心协议.md
│   ├── 02-领域扩展.md
│   ├── 03-AI协议集成.md
│   ├── 04-安全与成本.md
│   ├── 05-审核与遥测.md
│   ├── 06-错误码.md
│   └── 07-场景案例.md
├── schemas/                            Machine-readable JSON Schemas
│   ├── core/
│   │   ├── job-request.schema.json
│   │   ├── quote-response.schema.json
│   │   ├── telemetry.schema.json
│   │   └── capabilities.schema.json
│   └── domains/
│       ├── manufacturing.additive.fdm.v1.schema.json
│       ├── manufacturing.print.2d.v1.schema.json
│       ├── kinematics.robotic_arm.v1.schema.json
│       └── thermodynamics.cooking.v1.schema.json
└── examples/
    ├── 01-cloud-print-poster.json      End-to-end 2D print example
    ├── 02-3d-print-gear.json           FDM 3D print with audit
    └── 03-lab-pipette.json             Laboratory liquid handling
```

## Design Principles | 设计原则

1. **Micro-kernel + plugins** — a tiny, domain-agnostic core plus extensible domain schemas.
   微内核 + 插件化扩展 — 与领域无关的小核心加可扩展的领域 Schema。
2. **Trust nothing by default** — every physical action is quoted, budgeted, approved, audited.
   默认零信任 — 每一次物理动作都必须先询价、控预算、经审批、可审核。
3. **Asynchronous and observable** — physical time ≫ compute time; treat every job as a long-running observable.
   异步可观测 — 物理时间远长于计算时间，任何任务都是可观测的长时间作业。
4. **Ride existing rails** — embed into MCP / Function Calling / A2A; do not reinvent.
   嵌入现有协议栈 — 依托 MCP / Function Calling / A2A，不重造轮子。
5. **Closed-loop sensing** — AI sees the physical output through cameras and sensors and may intervene.
   闭环感知 — AI 通过摄像头和传感器看到物理结果，并可介入干预。

## Reading Order | 阅读顺序

| If you are... | Start with |
|---|---|
| Building an AI agent that manufactures things | `en/00-overview.md` → `en/03-ai-protocol-integration.md` → `examples/` |
| A hardware vendor writing an AIMP adapter | `en/01-core-protocol.md` → `en/02-domain-extensions.md` → `schemas/` |
| A security / compliance reviewer | `en/04-security-and-cost.md` → `en/05-audit-and-telemetry.md` |
| 中文读者 | `zh/00-概述.md` 开始 |

## Conformance Language | 规范性语言

This specification uses the key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, and **OPTIONAL** as defined in IETF RFC 2119 and RFC 8174. When these words appear
in lowercase, they carry their ordinary English meaning.

本规范中的关键词 **MUST（必须）**、**MUST NOT（禁止）**、**SHOULD（应当）**、
**SHOULD NOT（不应）**、**MAY（可以）**、**OPTIONAL（可选）** 的含义遵循 IETF RFC 2119
与 RFC 8174。小写形式时仅表达普通英语含义。

## License | 许可

This specification is released for open review. Reference implementations and test suites
are tracked separately.

本规范以开放评议形式发布。参考实现与测试套件另行维护。
