# Week 1 AI-Native Project — Data Quality Agent
### Scenario: On-prem to Azure migration, Medallion architecture

## 1. Project objective

A company is moving its data off an old on-prem database (say, SQL Server, Oracle)
into Azure. Every night, data flows in and lands in three layers in
Databricks — Bronze, Silver, Gold (this is called the **Medallion
architecture**). Something is going to break during a migration like this —
a table shows up short, a column changes type, numbers don't match. Someone
has to decide, fast: is this bad data, is it a temporary hiccup, or is it
actually expected?

We're building a small AI agent that makes that call, and proving — with
real numbers and real feedback from other data engineers — that it makes
good calls.

## 2. Three layers (Medallion architecture)

| Layer | What happens here | Analogy |
|---|---|---|
| **Bronze** | Raw data lands exactly as it came from the source (on-prem SQL Server), no cleanup. Just proof that "the file/table arrived." | The mailroom — nothing opened yet |
| **Silver** | Data gets cleaned: duplicates removed, types fixed, checked against reference tables (e.g. does this customer ID actually exist?). | Sorting and quality-checking the mail |
| **Gold** | Business-ready, aggregated tables that reports and dashboards actually read from. | The finished report on someone's desk |

**Where the agent sits:** at the **Bronze → Silver gate**. This is where
you first have enough information (row counts, nulls, schema, matches
against reference data) to tell whether something's wrong — and it's
before bad data can spread into Silver/Gold and reach a dashboard.

## 3. Problem statement

> During an on-prem-to-Azure migration, the agent observes a daily data
> batch landing in Bronze (in ADLS Gen2, pulled from on-prem SQL Server via
> Azure Data Factory using a Self-Hosted Integration Runtime and
> change-data-capture). It sees: row-count delta vs. the recent average,
> null-rate delta on key columns, a schema diff, referential-integrity
> failures against Silver dimension tables, and how long it's been since
> the last successful pull (watermark lag). It must select **accept,
> repair, isolate, or reject** the batch, because the true cause of the
> anomaly — (a) a real defect at the on-prem source or in the pipeline,
> (b) a temporary glitch (a VM being down, a network blip), or (c) an
> expected change (a planned maintenance window, a cutover step, a genuine
> drop in business volume) — is not known at the moment the batch lands.

### A concrete example to anchor everything
The `CustomerBookings` table normally lands ~50,000 rows/night. Tonight it
lands with 29,000 rows (a 42% drop), and the CDC watermark shows the last
successful capture from on-prem was 18 hours ago instead of the usual 1
hour. Is the on-prem CDC job broken? Was there a scheduled maintenance
window on-prem last night? Or is the self-hosted IR VM just stuck? The
agent has to decide what to do with this batch *right now*, without
knowing which of those is true.

## 4. Universal data-quality checks

These six checks are the backbone of almost any data quality system, and
map cleanly onto the medallion layers:

| Check | Question it answers | Typical layer |
|---|---|---|
| **Completeness** | Are required fields/rows missing? (null rate, row count vs. expected) | Bronze |
| **Validity** | Is the data in the right format/type/range? (schema diff, type mismatches) | Bronze → Silver |
| **Uniqueness** | Are there duplicate records or primary-key violations? | Silver |
| **Consistency / Referential integrity** | Do foreign keys match a dimension table? Do totals agree across tables? | Silver |
| **Timeliness** | Did the data arrive when expected? (watermark lag, late partitions) | Bronze |
| **Accuracy** | Do values look plausible / reconcile against a trusted source? (e.g. row count in Bronze vs. row count in the on-prem source table) | Silver → Gold |

The agent's "input" (below) is really just these six checks turned into
numbers.

## 5. Agent design

| Part | Definition |
|---|---|
| **Input** | Row-count % delta vs. 7-day average · null-rate delta on key columns · schema diff (added/removed/retyped columns) · referential-integrity failure count vs. Silver dimension tables · CDC watermark lag (hours since last successful on-prem pull) · a flag for "is a planned maintenance/cutover window active today?" |
| **Hidden state** | Which of: (a) real defect (on-prem source or pipeline bug), (b) transient glitch (network/VM issue, resolves on its own), (c) expected/planned change |
| **Belief** | P(defect), P(transient), P(expected) — starts from a prior based on past batches, updated as evidence comes in |
| **Action** | **Accept** (let it flow to Silver) · **Repair** (apply a known fix, e.g. re-run against a backup connection, backfill from the last good partition) · **Isolate** (quarantine the batch, don't let it reach Silver/Gold, flag for a human) · **Reject** (block it, notify the on-prem/migration team) |
| **Cost** | Accepting a real defect is the worst outcome — bad data reaches Gold/reporting and is hard to undo. Isolating an expected change just costs a delay. Rejecting a transient glitch causes unnecessary alarm but is cheap to recover from. |
| **Policy** | If P(defect) is high → reject. If P(transient) is high → isolate and re-check next run. If P(expected) is high and no critical nulls → accept. Otherwise → isolate and send to a human. |
| **Feedback** | Next run's results (did the row count recover? did on-prem confirm a real issue?) get used to update the prior for the next similar case. |

**Human-style reasoning added:** *compare with similar past cases* — the
agent keeps a short log of past anomaly patterns and outcomes (e.g. "watermark
lag + no maintenance flag + weekday = usually a real defect, historically
80% of the time") and uses that similarity as one more piece of evidence,
not just the raw thresholds.

**How we'll build it :** an LLM prompt that reasons over the
evidence and produces a belief + action, wrapped in simple `if/else`
guardrails for the hard limits (e.g. never auto-accept if referential
integrity fails).

