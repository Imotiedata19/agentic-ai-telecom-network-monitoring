# Technical documentation — Telecom AI: Data Quality & Outage Monitoring

## 1. Overview

The supplied n8n export contains **18 nodes**. It models an automated review pipeline driven by a **manual trigger** and a **six-hour Schedule Trigger** using the workflow timezone `Africa/Lagos`. The JSON contains `active: true`, but an exported flag alone does **not** establish present deployment or publishing status.

Operational intelligence is implemented through **deterministic JavaScript code** and upstream predictive-probability fields. Neither an LLM nor a model-inference/training integration is included.

## 2. Exact pipeline

1. `When clicking ‘Execute workflow’` OR `Schedule Trigger` → `Get Batch Progress.`
2. `Get Batch Progress.` → `Get row(s)` (up to 100 source records; `id > last_processed_id`, ascending order)
3. `Get row(s)` → `Normalize Telecom Geographic Data.` → `Code in JavaScript` (record validation)
4. `If`: on valid → `Assess Telecom Network Risk` → `Upsert row(s)1` (`telecom_network_risk_assessments`); on invalid → `Upsert row(s)` (`telecom_data_quality_issues`)
5. From the risk-assessment upsert: `Identify High-Risk Towers` → `Generate Network Risk Alert` → `If row does not exist` → `Send a message` (Gmail) → `Insert row` (`telecom_alert_delivery_log`)
6. Both persistence branches also feed `Merge` (Append) → `Calculate Batch Progress.` → `Update Batch Progress.`

### n8n tables and exact node binding

| Table | Nodes |
| --- | --- |
| `telecom_tower_risk_watchlist` | `Get row(s)` |
| `telecom_batch_progress` | `Get Batch Progress.`, `Update Batch Progress.` |
| `telecom_data_quality_issues` | `Upsert row(s)` |
| `telecom_network_risk_assessments` | `Upsert row(s)1` |
| `telecom_alert_delivery_log` | `If row does not exist`, `Insert row` |

## 3. Validation rules

The `Code in JavaScript` node checks presence of key identifiers and categories, and validates several metrics:

| Field | Validation |
| --- | --- |
| `id` | Integer ≥ 1 |
| `latest_uptime` | Number between 0 and 100 |
| `historical_total_downtime_minutes` | Number ≥ 0 |
| `historical_total_outage_count` | Integer ≥ 0 |
| `site_reliability_risk_score` | Number ≥ 0 |
| `predicted_risk_probability` | Number between 0 and 1 |
| `recommended_priority` | Integer ≥ 1 |

Required text fields include `tower_id`, `operator`, `state`, `city`, `network_type`, `data_origin`, `risk_category_final`, and `predicted_risk_category`.

- The normalization code retains `original_state` and a `geography_normalization_status`, and maps **Kano, Abuja, Abeokuta, Ibadan, Benin City, Calabar, Jos** to a limited reference state list.
- Validation rejects unknown city labels with `GEOGRAPHY_UNVERIFIED` and disallowed city/state pairs with `GEOGRAPHIC_MISMATCH`. Such rules only verify against the configured list; they are **not** comprehensive geographic truth.
- An uptime under 90 or a predicted probability ≥ 0.70 raises a validation warning but is not, by itself, a validation failure.
- Duplicated `tower_id` values inside the input batch raise warnings, not automatic rejection.
- Failed records upsert a data-quality entry; the mapping sets `is_test_data` to `true` unconditionally.

## 4. Operational-risk classification

In `Assess Telecom Network Risk`, the numeric thresholds are:

```text
criticalUptime       = 90
highUptime           = 97
watchUptime          = 99
criticalProbability  = 0.80
highProbability      = 0.60
watchProbability     = 0.40
highOutages          = 20
watchOutages         = 10
highDowntime         = 500  # minutes
watchDowntime        = 240  # minutes
```

Precedence is **CRITICAL → HIGH → MEDIUM → LOW** (and `INSUFFICIENT_DATA` if all assessed metrics are absent). `predicted_risk_probability` is an *input*, not a probability calculated inside the workflow. The assessment result sets `active_outage_status` to `NOT_VERIFIED`.

## 5. Alerting

- Alerts include towers whose computed `operational_risk_level` equals **HIGH** or **CRITICAL**.
- `Generate Network Risk Alert` sorts selected tower data and generates an `alert_signature` from tower ID, risk level, precomputed probability, data-quality status, and active-outage status.
- `If row does not exist` guards against resending the **same** `alert_key + alert_signature` combination.
- Gmail sends a **plain-text** email and the configured subject begins `[TEST]`.
- After successful Gmail-node execution, `Insert row` stores the alert key/signature, send time, Gmail message ID, and `SENT` status.

**Alert edge cases:** a failed email send does not create a success log. The concurrent checkpoint branch does not explicitly depend on Gmail success. Validate behavior and recovery before claiming guaranteed delivery. A large alert signature may also exceed practical field storage limits; test against the n8n data-table schema and consider a hash if needed.

## 6. Batch resume / checkpoint

`Get Batch Progress.` selects the `TELECOM_NETWORK` batch key. `Get row(s)` selects source IDs above `last_processed_id` (limit 100). The two downstream Data Table upsert outcomes are appended and checked against the source batch in `Calculate Batch Progress.`

Checkpoint code checks:

- Source-row count equals persisted-outcome count.
- Each source tower ID occurs the expected number of times in merged outcomes.
- Source IDs are positive safe integers.
- The next `last_processed_id` is the maximum source ID in that batch.

The `processed_count` and `checkpoint_status` values are produced by the calculation node but **not currently mapped** into `telecom_batch_progress` by `Update Batch Progress.` (only `batch_key` and `last_processed_id` are persisted).

**Testing caution:** the Merge node may behave differently on an empty branch depending on n8n execution semantics. Test batches with only valid records, only invalid records, mixed records, and no new records; verify that state advances only on successful processing.

## 7. Minimum acceptance tests before publication/activation

| Test | Expected result |
| --- | --- |
| Mixed valid / invalid data | Both persistence branches operate; checkpoint only advances after count / tower checks |
| All valid | No false failure rows; checkpoint persists correctly |
| All invalid | Failures recorded and batch progress updated if all rows accounted for |
| No new IDs | No duplicate work; workflow ends cleanly |
| Rerun unchanged risk batch | No repeated identical alert |
| Change a tower risk or probability | New signature permits a review alert after verifying recipient |
| Simulate Gmail delivery failure | Failure visible; success log not written; replay strategy defined |
| Unrecognized city | Explicit validation/review record |
| Restart next batch | Source records start after previous `last_processed_id` |
| Public JSON re-import | Rebind all tables and Gmail credentials, remain inactive until verified |

## 8. Publishing and security

- Treat the provided JSON as a sanitized portfolio **template**, not a copy of a ready-to-run private workspace.
- Reconfigure five n8n Data Tables and Gmail OAuth2 after import.
- Verify the trigger activation status in n8n itself; do not infer it solely from an exported `active` flag.
- Use sample or synthetic data in screenshots and examples. Hide email addresses, n8n workspace identifiers, bearer tokens, credentials, and private telecom network details.
- Avoid claiming live outage detection, autonomous AI-agent capabilities, or completed production deployment unless independently demonstrated.
