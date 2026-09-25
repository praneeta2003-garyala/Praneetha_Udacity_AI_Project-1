# Agentic AI Evaluation & Observability — Submission Guide

This folder contains the evidence produced while evaluating three agentic AI workflows.
The goal of the package is to make the implementation behavior, validation checks, routing
decisions, and failure-handling experiments easy to review.

## Project layout

| Folder | Focus | Main evidence |
|---|---|---|
| `01-policy-pipeline/` | Validated extraction, retry behavior, routing, and calibration | tests, routing JSON, calibration report, run output |
| `02-mortgage-extraction/` | Structured extraction plus deterministic mathematical validation | tests, extraction output, discrepancy output |
| `03-supply-chain/` | Multi-source investigation and graceful degradation | briefing, timeout run, comparison output |

Additional project-level files:

- `environment.txt` — execution environment and setup details.
- `perturbation-log.md` — controlled experiments and observed behavior.
- `reflection-brief.md` — observations, evidence, and design lessons.
- `calibration_report.py` — small reproducible calibration-report example.

## How the evidence is organized

The numbered directories contain the captured results from the three workflows. In particular:

- `tests.txt` contains the relevant test-suite output.
- `static-checks.txt` contains the static-analysis results.
- `*-run.txt` files preserve representative command output.
- `routing_decisions.json` records routing decisions produced by the policy workflow.
- `calibration-report.txt` shows calibration broken down by policy type and field.
- `perturbation/` contains controlled-change experiments.
- `screenshots/` contains visual evidence from the runs.

The evidence files are intentionally retained as captured. They should be read together with the
reflection and perturbation notes rather than treated as independent claims.

## Main engineering observations

### 1. Validation should determine whether a retry is useful

A missing source value is different from a malformed value. In the policy workflow, a genuinely
absent required field causes an immediate escalation instead of repeated model calls. A correctable
format problem can still be retried. This keeps retries purposeful and avoids encouraging the model
to invent a value simply because a field is expected.

### 2. Confidence is only one routing signal

The policy workflow combines extraction confidence with reviewer and integration signals. The
calibration evidence also shows why a single confidence threshold is unsafe: one policy/field slice
has high confidence but zero observed accuracy.

### 3. Structured output and semantic validation solve different problems

The mortgage workflow uses structured tool output to constrain the shape of the response, while a
separate validator checks relationships between values. A response can satisfy the schema and still
contain an internally inconsistent total.

### 4. Source disagreement should remain visible

The supply-chain workflow preserves conflicting source values instead of silently averaging them.
When a source becomes unavailable, the failure is recorded and the investigation continues with the
remaining sources.

## Reproducing the evidence

The exact commands and environment details used for the captured runs are recorded in the evidence
files. Reproduction should use the same project environment and the same input data when exact
replay behavior is required.
