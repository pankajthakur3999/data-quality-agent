# AI Research File

## Problem statement
See `README.md` §3.

## Project objective
See `README.md` §1.

## Technical terms to know before discussions
- Medallion architecture (Bronze / Silver / Gold)
- ADLS Gen2 (Azure Data Lake Storage)
- Self-Hosted Integration Runtime (ADF component that connects to on-prem)
- Change Data Capture (CDC) and watermark-based incremental loads
- Schema drift / schema enforcement (Delta Lake `mergeSchema`, schema-on-read vs. schema-on-write)
- Data quality dimensions: completeness, validity, uniqueness, consistency, timeliness, accuracy
- Quarantine / dead-letter pattern for bad batches
- Bayesian belief update, posterior probability, decision threshold
- Calibration (of a probabilistic classifier/agent)
- Reconciliation (comparing row counts/sums between source and target during migration)

## Search queries used
- "on-prem to Azure data migration data quality checks"
- "medallion architecture Bronze Silver Gold data quality gate"
- "change data capture watermark lag monitoring"
- "Delta Lake schema drift detection"
- "data reconciliation source vs target migration"
- "LLM agent data pipeline decision"

## Reddit communities
- r/dataengineering
- r/databricks
- r/AZURE
- r/ETL
- r/BusinessIntelligence
- r/dataanalysis


## Discussions with people on Reddit

- For anyone who's done an on-prem-to-ADLS Gen2 migration with a Self-Hosted IR: what's the earliest signal you watch for that tells you the IR itself is the problem, versus the source data being genuinely bad?

- How do you tell a CDC watermark lag caused by a VM/network blip apart from one caused by the source database job actually failing? Is there a reliable tell, or do you just wait it out?

