# Perturbation Log

Each experiment below changes one input or configuration value and compares the observed behavior
with the normal run. The captured command output remains in the corresponding evidence directory.

---

## System 1 — Policy extraction and routing

### Change

I created an additional offline experiment using `RecordedClient`. A required `premium_amount`
field was set to `None`, which represents information that is genuinely absent from the source.
Four identical responses were queued so that an unnecessary retry would be visible in the call count.

As a contrast, I also supplied a negative premium followed by a corrected value.

### Expected behavior

- Missing source information should cause immediate escalation.
- A correctable formatting error should be retried.

### Observed behavior

`01-policy-pipeline/perturbation/own-perturbation.txt` records:

- missing `premium_amount` → `RetryFutileEscalation`, **1 API call**
- negative premium followed by correction → successful extraction, **2 API calls**

The first case therefore avoided three unnecessary calls despite `max_retries=3`. The result supports
the rule that retrying is useful only when the failure can plausibly be corrected by another attempt.

The supporting test output is in `retry-tests.txt`.

---

## System 2 — Mortgage extraction and consistency validation

### Change

The supplied payroll document was copied into the perturbation area and its stated monthly total was
changed from `10,892.17` to `9,642.17`, matching the sum of the individual earnings.

I also evaluated the validator with a slightly different stated value and with a much larger tolerance.

### Expected behavior

- The edited document should produce a different replay key, so the old replay cache should not be
  reused.
- A corrected total should pass the consistency check.
- A small difference within tolerance should pass.
- A tolerance larger than the actual error could hide the discrepancy.

### Observed behavior

`run-A-replay.txt` confirms that replay could not find a response for the modified request and
returned a `FileNotFoundError`.

`run-B-validator.txt` shows:

- corrected total `9642.17` → `consistent=True`
- total `9642.67` with tolerance `1.00` → `consistent=True`
- original `10892.17` with tolerance `1500` → `consistent=True`

The last result is particularly important: the validator's protection is only as strong as its
configuration.

---

## System 3 — Multi-source supply-chain investigation

### Change

The `logistics` source was deliberately forced to time out using the simulated-timeout option.

### Expected behavior

The investigation should continue, explicitly mark the failed source as unavailable, and classify
metrics that depend on that source as incomplete rather than crashing.

### Observed behavior

The timeout run records:

`Sources unavailable: logistics unavailable (timeout)`

and places `late_shipment_count` under the incomplete section with a timeout-specific missing-source
reason. The command still completed successfully.

The diff in `briefing-diff.txt` exposed an additional effect: once the logistics source disappeared,
the 78% on-time-delivery value also disappeared, leaving the 95% supplier-audit value as a
single-source result. Thus, source availability can affect the apparent certainty of a metric even
when no business value has changed.

This suggests a useful future improvement: retain a visible "reduced source coverage" marker for
metrics that previously depended on a failed source, rather than relying only on the global timeout
banner.
