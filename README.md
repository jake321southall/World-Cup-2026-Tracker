# World-Cup-2026-Tracker
Repo to host a public README of my private World Cup 2026 Tracker that's hosted on AWS.

# 2026 World Cup — Favourite Tracker

A self-updating data pipeline that tracks which team is most likely to win the 2026 FIFA World Cup, based on bookmaker odds. The same data is served two ways — a low-latency operational store and an analytical data lake — and the whole stack (infrastructure, code, tests, deployment) is defined as code and shipped through CI/CD.

🔗 **Live site:** https://d1xkbbnwr650k2.cloudfront.net

---

## What it does

On a schedule, a Lambda function fetches World Cup outright-winner odds, converts them into normalised win probabilities, and writes them to three places:

- **`data.json`** in S3 — the latest snapshot the website reads.
- **`history/` in S3** — an immutable, date-partitioned record of every run, catalogued in Glue and queried with Athena.
- **DynamoDB** — the same snapshots modelled for fast key lookups.

A second scheduled Lambda runs an Athena query over the history to compute the biggest 24-hour mover, which the page displays. The site is served privately through CloudFront over HTTPS.

## Architecture

```
                          +------------------+         +------------------+
                          |  EventBridge 6h  |         |  EventBridge 6h  |
                          +--------+---------+         +--------+---------+
                                   |                            |
                                   v                            v
              Odds API <---- Updater Lambda              Stats Lambda
                                   |                       |      ^
       transform: odds ->          |                  StartQuery  | results
       normalised probabilities    |                       v      |
                                   |                    Athena ---+
                                   |                       |
                                   |                  Glue Data Catalog
                                   |                  (schema for history/)
                                   |                            |
            writes:                |                  writes:   |
              data.json            |                    stats.json
              history/             |                            |
              (+ DynamoDB)         |                            |
                                   v                            v
        +--------------------------------------------------------------+
        |                 S3  -  single private bucket                 |
        |   data.json    stats.json    history/    athena-results/     |
        +--------------------------------------------------------------+
                                   ^   ^                    |
                  reads history/ --+   | writes results ----+
                  (Athena)             | (Athena)
                                       |
                          +------------+-------------+
                          |  CloudFront (OAC, HTTPS) |
                          +------------+-------------+
                                       v
                                    visitor


  DynamoDB  -  NoSQL operational store
    Written by the Updater Lambda; queried by key via CLI / console.
    Not read by the website (there is no live backend API).
```


Two stores, on purpose:

- **DynamoDB (OLTP)** — single-digit-millisecond lookups by key. Modelled around two access patterns.
- **S3 + Glue + Athena (OLAP)** — cheap, serverless analytical scans over the full history.

The same data lives in both because each is good at a different shape of question — keyed operational reads vs. scan-and-aggregate analytics — and forcing either to do the other's job is the classic mistake.

## Tech stack

| Area | Tools |
|---|---|
| Cloud | AWS — Lambda, S3, CloudFront, EventBridge, IAM, SSM Parameter Store, Glue, Athena, DynamoDB |
| Infrastructure as Code | Terraform, remote state in S3 + DynamoDB locking |
| Application | Python 3.12, `boto3` |
| Analytics | Athena (SQL, window functions, partition projection) over a Glue-catalogued data lake |
| NoSQL | DynamoDB — primary key schema + global secondary index, on-demand capacity, TTL |
| Testing | `pytest` (TDD on the core transform logic) |
| CI/CD | GitHub Actions — tests on every push, automated deploy on merge to `main` via OIDC |
| Security | Keyless CI/CD (GitHub OIDC), least-privilege IAM on Lambda roles, secrets in SSM SecureString, private S3 behind CloudFront |

## Project structure

```
.
├── lambda_src/
│   └── handler.py        # updater: fetch -> transform -> S3 + history + DynamoDB
├── stats_src/
│   └── handler.py        # scheduled Athena query -> stats.json
├── tests/
│   └── test_transform.py # pytest unit tests on the pure functions
├── site/
│   └── index.html        # static front-end (favourite, board, biggest mover)
├── terraform/
│   ├── providers.tf
│   ├── variables.tf
│   ├── backend.tf        # remote state config
│   ├── s3.tf
│   ├── cloudfront.tf     # OAC + distribution (private bucket, HTTPS)
│   ├── lambda.tf         # updater Lambda + role
│   ├── schedule.tf       # updater EventBridge schedule
│   ├── glue.tf           # Glue database + table (partition projection)
│   ├── athena.tf         # Athena workgroup
│   ├── stats.tf          # stats Lambda + role + schedule
│   ├── dynamodb.tf       # NoSQL table + GSI
│   ├── oidc.tf           # GitHub OIDC provider + CI deploy role
│   └── outputs.tf
└── .github/workflows/
    ├── test.yml          # pytest on every push/PR
    └── deploy.yml        # terraform apply on push to main
```

## Design notes

**Pure vs. impure separation.** `extract_outcomes()` and `compute_probabilities()` are pure functions — same input, same output, no AWS or environment dependencies — which makes them trivially unit-testable. All AWS/environment code is kept separate and lazily initialised, so the module imports cleanly even in a CI runner with no AWS configuration.

**Two data structures, chosen by access pattern.** DynamoDB serves keyed operational reads; S3 + Athena serves analytical scans. This is a deliberate OLTP/OLAP split — the project's answer to "different data structures and their benefits and limitations under particular use cases."

**DynamoDB modelling (logical -> physical).** Design started from the access patterns:
- *AP1 — a team's history over time:* partition key `team`, sort key `run_ts`. A single-partition `Query` with a sort-key range — never a `Scan`.
- *AP2 — the ranked board for one run:* a global secondary index `by-run` (partition key `run_ts`, sort key `probability`), so the leaderboard falls out of the sort order.
- *Tuning:* on-demand capacity (no standing charge for a spiky, low-volume workload), TTL to bound storage, batch writes for throughput, and an `INCLUDE` projection so the GSI carries only what that pattern needs.

**Athena over a Glue catalogue, with partition projection.** History lands as newline-delimited JSON, date-partitioned. Glue holds the schema; partition projection lets Athena infer partition paths from the query (no crawler to schedule), so new data is queryable the moment it lands and queries prune to the relevant days.

**Secrets never touch the repo or state.** The odds API key is an encrypted SSM SecureString, created out-of-band via the CLI and fetched at runtime — never in code, Git, or Terraform state.

**Keyless CI/CD.** GitHub Actions authenticates to AWS via OIDC: a short-lived, GitHub-signed token is exchanged for temporary credentials, and the deploy role is assumable only from this repository's `main` branch. No long-lived AWS keys are stored anywhere. (Scoping the CI deploy role from `AdministratorAccess` down to exactly the resources this stack manages is a documented next hardening step; the OIDC trust conditions are the primary access boundary today.)

**Private origin.** The S3 bucket is private; CloudFront reaches it via Origin Access Control, and HTTPS is terminated at the edge. The bucket isn't directly reachable.

## Running the tests

```bash
python -m pytest -q
```

The tests cover the odds-to-probability conversion: ranking, the normalisation invariant (probabilities sum to 1), the empty-input edge case, and averaging odds across bookmakers.

## Querying the data

**Athena (analytical) — a team's probability over time, in the AWS console:**

```sql
SELECT dt, updated_at, probability
FROM "wc2026"."history"
WHERE team = 'France'
ORDER BY updated_at;
```

`EXPLAIN` on a `WHERE dt = ...` query shows partition pruning — the query-planning evidence.

**DynamoDB (operational) — the same question as a single-partition `Query`, not a `Scan`:**

```bash
aws dynamodb query \
  --table-name wc2026-favourite-probabilities \
  --key-condition-expression "team = :t" \
  --expression-attribute-values '{":t":{"S":"France"}}' \
  --region eu-west-2
```

## Deploying

Infrastructure is managed through Terraform with remote state in S3:

```bash
cd terraform
terraform init -backend-config="bucket=<your-state-bucket>"
terraform plan
terraform apply
```

In normal operation, deployment is automatic: every push to `main` that passes the tests triggers `terraform apply` via GitHub Actions.

## Known limitations

- Implied probabilities come from bookmaker odds, which reflect a market consensus shaped by betting volume and public sentiment — not a calibrated statistical model. A natural extension is an Elo or simulation-based model alongside the market odds.
- The `by-run` GSI places all of one run's writes in a single partition. Fine at ~40 items every 6 hours; at high write volume the GSI partition key would be redesigned to avoid a hot partition.
- The CI deploy role currently uses broad permissions gated by 
