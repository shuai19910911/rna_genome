# SoyFormer-U Project

## Active Execution Plan

- Active plan: `.planning/2026-05-29-master-execution-plan/`
- Execute future work according to the active plan unless the user explicitly changes direction.
- After every task, update `.planning/2026-05-29-master-execution-plan/progress.md`.
- If a task changes metrics, conclusions, bugs, or technical decisions, also update `.planning/2026-05-29-master-execution-plan/findings.md`.
- If a task changes phase status, priorities, or Go/No-Go criteria, also update `.planning/2026-05-29-master-execution-plan/task_plan.md`.
- When providing GPU commands, always include required compute resources: GPU count and suggested GPU IDs, expected VRAM, CPU cores, RAM, expected runtime, and output files to check.


## Auto-Review Pipeline

### Plan Evaluation (3+ skills, triggered after design/planning output)
| # | Skill | Purpose |
|---|---|---|
| 1 | `brainstorming` | Evaluate design alternatives, scope, and novelty |
| 2 | `writing-plans` | Assess implementation feasibility and gating |
| 3 | `planning-with-files` | Verify plan completeness and progress tracking |

### Code Review (5+ skills, triggered after code generation)
| # | Skill | Purpose |
|---|---|---|
| 1 | `code-review` | Diff correctness and bug detection |
| 2 | `security-review` | Security vulnerability audit |
| 3 | `verification-before-completion` | Confirm changes work as intended |
| 4 | `test-driven-development` | Test coverage and edge cases |
| 5 | `systematic-debugging` | Root cause analysis if issues found |

### Gate Pipeline
Gate 0 (Data) → Gate 1 (Baselines) → Gate 2 (M1) → Gate 3 (M2) → Gate 4 (Cross-species)

### Key Constraints
- Data dirs rna_only/ and rnadata/ are READ-ONLY
- All output to rna_project/
- SLURM scripts: use full Python path, no mamba activate
- GPU scripts: add {GPU_DEVICE} placeholder for user to fill
