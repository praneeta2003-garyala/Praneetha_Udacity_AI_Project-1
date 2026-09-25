# Reflection Brief — Agentic AI Evaluation & Observability

**Name:** Mallu Sumasahithi  
**Date:** 2026-09-24

This document summarizes observations from the captured runs. Numerical results and quoted outputs
below are taken from the corresponding evidence files in this submission.

---

## 0. Execution environment

| Item | Recorded value |
|---|---|
| Operating system | Debian GNU/Linux 12 (bookworm), x86_64 |
| Python | 3.13.0 |
| Workspace | Udacity/Vocareum environment |
| Run date | 2026-09-23 |
| Live API execution | No. System 1 used its offline test path, System 2 used replay mode, and System 3 used offline mode. |

The environment details are also preserved in `environment.txt`.

---

## 1. Validated policy extraction and routing

### Test and routing evidence

`01-policy-pipeline/tests.txt` records **45 passed and 3 skipped** tests. The routing evidence contains
two representative decisions: `POL-A` was auto-approved and `POL-B` was sent to human review.

### 1a. What happened when a required value was absent?

In my additional perturbation, `premium_amount` was changed to `None`. The recorded result in
`01-policy-pipeline/perturbation/own-perturbation.txt` was:

`RetryFutileEscalation api_calls=1`

The system stopped after the first call even though four responses had been queued. This is the
appropriate behavior for an information-absence case: another model call cannot discover information
that the source document does not contain.

The contrast experiment used `premium_amount=-1500.0`, which is a repairable formatting problem.
That run used two calls and reached a successful extraction. The important distinction is therefore
not "retry versus never retry"; it is whether another attempt has a reasonable chance of fixing the
failure.

### 1b. Why did `POL-B` go to human review?

`01-policy-pipeline/routing_decisions.json` shows:

- policy: `POL-B`
- type: `home`
- decision: `human_review`
- field below threshold: `premium_amount`
- premium confidence: `0.5`
- reviewer disagreements: none
- integration failures: none

For this particular record, the low-confidence field was sufficient to trigger review. The broader
test suite also checks that reviewer disagreement and integration failures can independently force
human review.

### 1c. What did the calibration slice reveal?

`01-policy-pipeline/calibration-report.txt` contains this notable cell:

`umbrella exclusions n=2 conf=0.93 acc=0.00 brier=0.865`

The overall Brier score is `0.291`.

The sliced result is more informative than the aggregate because it identifies a specific policy
type and field where confidence is high but the observed answers are consistently wrong. An overall
metric can hide that kind of localized failure.

---

## 2. Mortgage document extraction and validation

### Test and representative runs

`02-mortgage-extraction/tests.txt` records **25 passed** tests. The run artifacts include appraisal
and income-verification examples, as well as a document deliberately containing an inconsistent
monthly-income total.

### 2a. Why both schema validation and a separate validator are needed

The discrepancy evidence reports:

- calculated monthly income: `9642.17`
- stated monthly income: `10892.17`
- difference: `-1250.0`
- `consistent`: `false`

The structured tool response can guarantee that the output has the expected fields and data types.
It cannot prove that related numeric values agree. The deterministic validator supplies that second
guarantee.

Conversely, the validator cannot detect every semantic error. It only checks the relationships it
was designed to check. A value can be structurally valid and arithmetically consistent while still
being wrong in another respect.

The perturbation also demonstrated that validation strength depends on configuration: increasing
the tolerance to `1500` caused the original $1,250 discrepancy to be accepted.

### 2b. Why missing information remains `null`

The `income_missing_bonus.txt` replay contains:

`"bonus_monthly": null`

This is preferable to fabricating a value. A missing financial field should remain explicitly
unknown rather than being converted into a plausible-looking number.

### 2c. Why normalize during extraction?

The appraisal example contains text such as approximately `2,400 sq ft`, while the extracted result
stores `gross_living_area_sqft` as `2400`.

Normalizing once during extraction gives downstream consumers a consistent typed value. Otherwise,
each later consumer would need to interpret commas, approximation markers, units, and surrounding
labels independently.

---

## 3. Multi-source supply-chain investigation

### Test and briefing evidence

`03-supply-chain/tests.txt` records **34 passed** tests. The normal briefing keeps a disagreement in
the `Contested` section rather than replacing it with a single derived value.

### 3a. Preserving disagreement

For `on_time_delivery_rate`, the normal briefing records:

- `95.0 percent` — `supplier_audit`, dated 2026-04-10
- `78.0 percent` — `logistics`, dated 2026-04-05

Keeping both values is more useful than inventing an average. The difference itself is a signal that
needs investigation, and the dates and source names give the reviewer something concrete to follow up.

### 3b. What happens when a source times out?

The timeout run records:

`Sources unavailable: logistics unavailable (timeout)`

and moves `late_shipment_count` into the incomplete area with the reason that the logistics source
could not be read.

This is different from a metric for which no source reports a value. In the latter case the system
has evidence of a genuine information gap; in the former, the information may still exist but was
temporarily unreachable.

The run still completes because a failure from one source is represented as information in the
briefing instead of aborting the entire investigation.

One important observation from `perturbation/briefing-diff.txt` is that removing the logistics
source also removes its 78% contribution to the on-time-delivery conflict. The remaining 95% value
then appears as a single-source result. That means source outages themselves can change how a reader
interprets a metric, even when the underlying business situation has not changed.

### 3c. Why dates matter

The defect-rate evidence contains:

- `180.0 ppm` from `supplier_audit` on 2026-04-10
- `190.0 ppm` from `internal_quality` on 2026-04-08

A small difference across nearby dates does not necessarily represent a source contradiction. Keeping
the observation date alongside the value gives the reviewer the temporal context needed to distinguish
a trend from a disagreement.

---

## 4. Overall lessons

### 4a. Evaluate the result instead of trusting the model

The clearest example is the mortgage workflow. The extraction had valid structure, but its monthly
income total was inconsistent with the component values. The deterministic check caught the problem
before the result could be treated as internally reliable.

### 4b. Confidence is not the same as correctness

The policy calibration slice with `conf=0.93` and `acc=0.00` is the strongest example. A high model
confidence score by itself is not enough evidence for automatic approval.

### 4c. Applying the patterns elsewhere

For a document-to-accounting workflow, I would combine structured extraction with deterministic
checks. For example, an invoice pipeline could extract vendor, invoice number, dates, line items,
tax, and total; validate that the arithmetic is consistent; retry only fixable failures; and route
missing or unresolved information to a person.

Useful operational metrics would include retry frequency, escalation rate, validation discrepancy
rate, and calibration sliced by vendor and field. Configuration changes to numerical tolerances should
also be monitored because the mortgage perturbation showed that a loose tolerance can conceal a real
error.
