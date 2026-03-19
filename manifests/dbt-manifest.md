# 🗂️ Master dbt Architect: Skill Manifest

This manifest maps the 4-Stage Agentic Framework to specific logic modules.
**Architectural Goal:** 100% Idempotency, Audited SCD, and Contract-First Modeling.

## 🧠 Core Agent Logic
| Component | Path | Purpose |
| :--- | :--- | :--- |
| **Identity** | `agents/dbt-architect.md` | Persona, Tone, and Core 15 Mastery Concepts. |
| **Orchestrator** | `orchestrator/project-manager.md` | Stage-Gate sequencing (Rule 01-05). |

## 🛠️ Skill Micro-services
| ID | Skill Name | Logic File | Capability |
| :--- | :--- | :--- | :--- |
| **SKL-01** | Data DNA Profiler | `skills/SKL-PROFILE-01-dna.md` | Statistical audit of seeds/sources. |
| **SKL-02** | Synthetic History Factory | `skills/SKL-MOCK-02-history.md` | Multi-state SCD stress-testing. |
| **SKL-03** | Gherkin-to-SQL | `skills/SKL-BDD-03-gherkin.md` | BDD Unit Test generation (Given/When/Then). |
| **SKL-04** | Temporal Architect | `skills/SKL-SCD-04-temporal.md` | SCD Type 2 & PIT Join implementation. |
| **SKL-05** | Governance Enforcer | `skills/SKL-CONTRACT-05-gov.md` | YAML Contracts & Schema Enforcement. |

---
**Usage Note:** To invoke a skill, reference its path using `#file` in Copilot Chat or allow the Orchestrator to auto-select based on the task context.