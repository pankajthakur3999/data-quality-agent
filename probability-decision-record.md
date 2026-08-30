# Probability Decision Record

## Case: CustomerBookings batch, 42% row-count drop during migration

### The situation, in plain words
Every night, on-prem SQL Server data flows through Azure Data Factory
(via a Self-Hosted Integration Runtime, using CDC) into Bronze in ADLS
Gen2. Tonight, `CustomerBookings` landed with 29,000 rows instead of the
usual ~50,000 (a 42% drop), and the CDC watermark shows the last
successful pull from on-prem was 18 hours ago, not the usual 1 hour.
Nothing else looks wrong — no schema changes, no referential-integrity
failures.

| Item | Content |
|---|---|
| Evidence | Row count down 42% vs. 7-day average; watermark lag 18h vs. usual 1h; no schema diff; no RI failures |
| Hidden states | (a) real defect (on-prem CDC job or IR broke), (b) transient glitch (IR VM stuck, network blip — will likely resolve itself), (c) expected change (a planned on-prem maintenance window) |
| Beliefs (prior) | P(defect) = 0.40, P(transient) = 0.45, P(expected) = 0.15 — set from how often each of these has happened historically for this table |
| Action options | Accept / Repair / Isolate / Reject |
| Costs | Accept-when-defect = high (bad/incomplete data reaches Silver & Gold, dashboards under-report bookings). Isolate-when-expected = low (just a delay). Reject-when-transient = medium (unnecessary escalation to the on-prem team) |
| Policy | If P(defect) ≥ 0.60 → reject. If P(transient) ≥ 0.50 → isolate, re-check next run. If P(expected) ≥ 0.70 and no critical nulls → accept. Otherwise → isolate + human review |
| Decision (before checking further evidence) | No belief clears its threshold → **isolate, send to human** |

The three prior probabilities sum to 100% (0.40 + 0.45 + 0.15 = 1.00) —
this is required for a valid belief.

### Step-by-step: updating the belief with new evidence

**Step 1 — start with the prior.**
P(defect)=0.40, P(transient)=0.45, P(expected)=0.15

**Step 2 — get new evidence.**
Someone checks the on-prem change calendar: there was **no** planned
maintenance window last night. But the Azure IR VM's monitoring shows a
CPU/memory spike and an unplanned restart at the time the CDC job would
have run.

**Step 3 — estimate how likely this evidence is under each hidden state**
(this is the "likelihood" — ask: *if this hidden state were true, how
surprising would this evidence be?*):
- P(evidence | defect) = 0.20 — a real on-prem defect wouldn't usually
  show up as an IR VM restart
- P(evidence | transient) = 0.75 — an IR VM restart is exactly the
  signature of a transient infrastructure glitch
- P(evidence | expected) = 0.05 — there's no planned maintenance window,
  so "expected" is now very unlikely

**Step 4 — multiply prior × likelihood for each state:**
- defect: 0.40 × 0.20 = 0.080
- transient: 0.45 × 0.75 = 0.3375
- expected: 0.15 × 0.05 = 0.0075
- sum = 0.425

**Step 5 — normalize (divide each by the sum) so they add to 100% again:**
- P(defect) = 0.080 / 0.425 = **0.19**
- P(transient) = 0.3375 / 0.425 = **0.79**
- P(expected) = 0.0075 / 0.425 = **0.02**

**Step 6 — compare to the policy thresholds.**
P(transient) = 0.79 ≥ 0.50 → the "isolate, re-check next run" rule fires.

**Step 7 — new action: Isolate.**
Don't let the batch into Silver yet. Flag it for automatic re-check on the
next scheduled run (the IR VM restart has likely already resolved). If the
next run comes back normal, close the case as "transient, resolved" and
feed that outcome back into the prior for next time. If it's still off,
escalate as a likely defect.


