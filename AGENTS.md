# ABC Bank: agent instructions

This repo is the bank's side of the Offers Platform project (Customer Spend
Analytics v3): the ABC Bank AWS account's infrastructure, the core-banking
Postgres database, a scale-factor synthetic data generator, change data
capture (DMS) to the Offers Platform account, and the file exchange with it.
ABC Bank is fictional; all data here is synthetic. Project-wide context
(roadmap, business problem, ADRs) is in
https://github.com/Lakshmi100/offers-platform-workspace (`docs/`).

Pipeline rules (story workspace, git conventions, base definition of done)
are in `.claude/factory/RULES.md`.

## Stories

- Source: GitHub Issues in `Lakshmi100/abc-bank`
- Key prefix: `ABC` (issue #4 is `ABC-4`)
- Every story is approved by a human (label `approved`) before `/story` runs.

## Story review

Readiness threshold: 80

Constraints every story in this repo must address, or state as not
applicable (the story reviewer scores these under C4):
- **Personal data:** which fields are personal data. Data is synthetic, but it
  is handled as if real: the Offers Platform tokenizes it downstream.
- **Card data (PCI):** full card numbers are never stored; only the last 4 digits.
- **Network and access:** no public endpoints; people get in through IAM
  Identity Center; no IAM users or long-lived access keys.
- **Encryption:** data at rest uses the ABC Bank account's customer-managed KMS keys.
- **Cost:** anything billed by the hour (RDS, DMS, EC2, interface endpoints)
  is a session resource: it has a teardown check, and the session's cost is
  recorded. Long-lived resources state their monthly cost.
- **Tagging:** every AWS resource carries the four cost tags, with the values
  under "Coding standards".
- **Downstream compatibility:** effect on CDC to the Offers Platform (DMS needs
  primary keys; schema changes reach the platform's intake).

Extra blocking rules (fail the gate whatever the score):
- **B5:** the story creates hourly-billed AWS resources and has no teardown
  acceptance criterion.
- **B6:** the story adds or changes customer, account, card or transaction
  data and does not say which fields are personal data.

## Layout

| Path | What it holds |
|------|---------------|
| `infra/` | Terraform for the ABC Bank AWS account, split into roots by lifecycle (see "Terraform roots") |
| `core-banking/` | Postgres schema: numbered SQL migrations, validation checks |
| `data-generator/` | Scale-factor synthetic data generator (Python) |
| `cdc/` | DMS endpoints and tasks |
| `exchange/` | Outbound feeds; inbound offer and settlement files |
| `docs/` | Repo docs; `docs/stories/` is the factory's story workspace |

Folders are created by the first story that needs them.

## Reference pattern

None yet. The first story in each folder sets its pattern; later stories
mirror it.

## Commands

Not set up yet. The first story that adds code must add a `Makefile` with:

    make test     # pytest for Python; terraform validate for every Terraform root
    make lint     # ruff check + ruff format --check; terraform fmt -check -recursive

Until then, a story's tests run with the commands its design names.

Test framework and locations: pytest, in `tests/` beside the code it tests
(e.g. `data-generator/tests/`). SQL and infrastructure checks live in the
same `tests/` folders as pytest tests that run SQL or read Terraform output.

**Tests that need real AWS** (applying Terraform, loading RDS) run only when
the story says so, against the ABC Bank account, and must tear down what
they create. Everything else must run offline: mock AWS calls, and use a
local Postgres (docker) for SQL tests.

## Coding standards

- Python follows `.claude/factory/standards/python-style.md`.
- Terraform: `terraform fmt`; variables and outputs documented; region
  `us-east-1`; no hard-coded account IDs; roots find each other's resources by
  name, alias or tag, not by hard-coded IDs.
- Cost tags and their values: `data-product = abc-bank`, `environment = dev`,
  `owner = Lakshmi100`, `cost-center = offers-platform-portfolio`.
- SQL: one numbered migration per change (`0001_create_customers.sql`),
  never edited once merged.

## Terraform roots

The account has **one state bucket** (created by `infra/bootstrap/`) and
**several Terraform roots**, each with its own state key in that bucket,
split by how long their resources live:

| Kind | Example | Lifecycle |
|------|---------|-----------|
| Bootstrap | `infra/bootstrap/` | Applied once; creates the state bucket |
| Long-lived | `infra/foundation/` (KMS key, network) | Stays between sessions; near-zero monthly cost; never destroyed by a session |
| Session | `infra/core-banking-db/` | Created for a working session and destroyed after it |

Anything billed by the hour belongs in a session root. Destroying a session
root never touches another root's resources.

## Definition of done (additions)

- `terraform validate` and `terraform fmt -check` pass for every changed root.
- Session roots: `terraform destroy` was run, nothing billable with the
  story's tags is left, and the session's cost is in the PR.
- Long-lived roots: the PR states the monthly cost, and `terraform plan`
  shows no changes after apply.

## Boundaries (additions)

- Never apply Terraform to any account except ABC Bank.
- Never commit Terraform state, `.tfvars` with secrets, `.env` files or credentials.
- Never use real personal or card data; synthetic only.
- Do not create NAT gateways (use S3 gateway endpoints).
