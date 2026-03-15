# NCP-AAI Study Course

A self-study course for the **NVIDIA Certified Professional — Agentic AI** certification exam.

You will build real agent systems using NVIDIA's stack: NeMo Agent Toolkit, NIM, NeMo Guardrails, and NVIDIA Blueprints. This is not a reading course — you write code, run agents, configure guardrails, and produce artifacts against real NVIDIA services.

## Who This Is For

- You know Python (OOP, async, packaging)
- You know basic ML/LLM concepts (transformers, tokenization, inference)
- You want to pass the NCP-AAI exam or build production agentic AI on NVIDIA's stack
- You do NOT need a GPU to start — the first 10 days run on a CPU laptop with hosted NVIDIA APIs

## What You Need

- Python 3.10+
- An NVIDIA account at [build.nvidia.com](https://build.nvidia.com) (free)
- Docker (for later labs)
- No GPU required until optional advanced sections

See [course/platform_setup/feature_to_install_matrix.md](course/platform_setup/feature_to_install_matrix.md) for exactly what to install and when.

## How to Start

**Do these 4 things first, in order:**

1. [Create your NVIDIA account and API key](course/platform_setup/build_nvidia_account_and_api_key.md)
2. [Make your first NIM API call](course/platform_setup/first_nim_api_call.md)
3. [Run your first NeMo Agent Toolkit workflow](course/platform_setup/first_nat_workflow.md)
4. [Configure your first NeMo Guardrails rail](course/platform_setup/first_guardrails_run.md)

Then start Module 1. Follow the module order — they build on each other.

## How to Do the Course

Each module is a `.md` file. Read it, do the hands-on exercise inside it, then do the corresponding lab. The code in the files is real — copy it, run it, modify it. You build your own project repo as you go.

If you want the fastest path to hands-on NVIDIA exposure, see the [3-day / 7-day / 14-day accelerated paths](course/minimum_nvidia_path.md).

For the full 8-10 week study plan, see [course/01_learning_path.md](course/01_learning_path.md).

## Course Structure

```
course/
├── platform_setup/          <- Do this first
│   ├── build_nvidia_account_and_api_key.md
│   ├── first_nim_api_call.md
│   ├── first_nat_workflow.md
│   ├── first_guardrails_run.md
│   ├── blueprint_walkthrough.md
│   ├── optional_self_hosted_nim.md
│   ├── feature_to_install_matrix.md
│   └── troubleshooting_matrix.md
│
├── modules/                 <- Core curriculum (13 modules)
│   ├── module_01_foundations.md
│   ├── module_02_architecture_patterns.md
│   ├── module_03_reasoning_planning_memory.md
│   ├── module_04_knowledge_rag.md
│   ├── module_05_tools_prompt_engineering.md
│   ├── module_06_nat_orchestration.md
│   ├── module_07_multi_agent.md
│   ├── module_08_evaluation_tuning.md
│   ├── module_09_safety_guardrails.md
│   ├── module_10_deployment_scaling.md
│   ├── module_11_observability_monitoring.md
│   ├── module_12_human_ai_oversight.md
│   └── module_13_certification_prep.md
│
├── labs/                    <- One lab per module (12 labs)
│   ├── lab_01_agent_loop.md
│   ├── ...
│   └── lab_12_human_loop.md
│
├── capstones/               <- Two substantial projects
│   ├── capstone_01.md          (Production RAG agent)
│   └── capstone_02.md          (Multi-agent system with governance)
│
├── final_review/
│   └── mock_exam_prep.md    <- 55 practice questions with answers
│
├── readings/
│   └── reading_track.md     <- Official NVIDIA readings per module
│
├── 00_overview.md           <- Full course design document
├── 01_learning_path.md      <- Study plan and environment setup
├── 02_exam_mapping.md       <- Exam domains mapped to modules
├── milestones.md            <- 8 checkpoints proving NVIDIA interaction
├── minimum_nvidia_path.md   <- 3/7/14-day accelerated paths
├── quality_gate.md          <- Completion checklist (are you ready?)
└── glossary.md              <- 55 terms defined
```

## Exam Info

| | |
|---|---|
| **Exam** | NVIDIA Certified Professional — Agentic AI |
| **Duration** | 120 minutes |
| **Questions** | 60-70 |
| **Cost** | $200 |
| **Passing** | Online, remotely proctored |

See [course/02_exam_mapping.md](course/02_exam_mapping.md) for how each exam domain maps to modules, with coverage confidence scores.

## When You're Done

Run through the [quality gate checklist](course/quality_gate.md). If everything is YES, you're ready for the exam.

## Troubleshooting

Something broke? Check the [troubleshooting matrix](course/platform_setup/troubleshooting_matrix.md) — 22 common issues with fixes.

## Source Honesty

This course is built from official NVIDIA documentation. Where official docs are thin, content is explicitly marked as inference. No exam questions are real, all practice questions are inferred from published domain descriptions. See [course/02_exam_mapping.md](course/02_exam_mapping.md) for the full source classification.
