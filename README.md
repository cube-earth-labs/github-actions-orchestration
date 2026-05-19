# GitHub Actions — Orchestration Demo

Scaffolded GitHub Actions pipelines built to answer Acme Corporation's orchestration-tool evaluation. The steps in every workflow are simple `echo` scripts — the objective is to **show the orchestration model, not run real work**.

## Customer Driver

Acme Corporation's automation team does not have an orchestration tool today. Current orchestration is manual. They are evaluating:

| Option | Status |
| --- | --- |
| **Terraform Actions** | Under evaluation — wants demo of TF Actions vs Ansible provider |
| **Terraform Stacks** | Customer hinted they want a demo (not built yet) |
| **Ansible Automation Platform (AAP)** | Existing platform; native workflow templates strong for Ansible-heavy orchestration |
| **Jenkins** | **Dismissed** — customer-specific reliability issues (down weekly, ops burden, not team-owned) |
| **GitHub Actions** | Future possibility once GitHub Enterprise Cloud is connected |

> [!NOTE]
> **Recommendation:** GitHub Actions is likely the best fit. TFE alone won't orchestrate the entire workflow (CRQ → provision → config → observability → notify). This demo exists to show the customer what that end-to-end orchestration looks like in GHA.

## The Customer's Three Requirements

The README and the pipelines below are organized around these three questions:

1. **Can parts be re-triggered if they fail?**
   - **Native UI:** "Re-run failed jobs" and "Re-run all jobs" buttons on every run page
   - **State preservation:** outputs and artifacts from successful upstream jobs are kept, so re-running starts from the failure point — not from scratch
   - **Demo:** run any of the three deploy workflows with `inject_failure: splunk-integrate`. Splunk fails on attempt 1, dynatrace and cmdb stay green, smoke-tests is skipped. Click **Re-run failed jobs** — attempt 2 skips the injection guard (`github.run_attempt == 1` is now false), splunk succeeds, and downstream proceeds. Terraform, ansible, dynatrace, and cmdb are NOT re-run — their original outcome is preserved.

2. **Can you see the order in which things happened visually?**
   - **Job DAG view:** the Actions tab renders every workflow as a graph — boxes for jobs, arrows for `needs:` dependencies, color-coded by status
   - **Demo:** all three deploy workflows exercise this with steps that fan out and converge

3. **Can you chain processes serially and in parallel?**
   - **Serial:** `needs: [upstream-job]` declares a dependency
   - **Parallel:** any two jobs without a `needs:` chain between them run concurrently
   - **Fan-out:** matrix strategy (`strategy.matrix:`) spawns parallel job instances
   - **Fan-in:** a downstream job with `needs: [a, b, c]` blocks until all complete

## What the Pipelines Demonstrate

Each environment has its own workflow file with environment-specific gate logic. The downstream pipeline (terraform → ansible → integrations → smoke → notify) is the same shape in all three.

### `deploy-sandbox.yml` — Deploy to Sandbox

- **Gate:** GitHub Actions environment (`change-approval`) with one required reviewer. No change request is created.
- **Flow:** validate → approval → terraform → ansible → splunk + dynatrace + cmdb (parallel) → smoke tests → notify
- **Inputs:** `app_name`, `change_title`, `change_description`, `inject_failure`

### `deploy-staging.yml` — Deploy to Staging

- **Gate:** none. Allowed any time. The workflow **auto-creates a change request issue** (labels: `change-request` + `status:approved` + `env:staging`) for audit/record.
- **Flow:** validate → create-cr → terraform → ansible → integrations → smoke → mark-deployed → notify
- **Inputs:** `app_name`, `change_title` (becomes CR title), `change_description` (becomes CR body), `inject_failure`

### `deploy-production.yml` — Deploy to Production

- **Gate:** **requires an existing approved CR** (provided via `change_request_id`). The workflow fetches the issue and fails fast unless labels include `change-request` + `status:approved` + `env:production`. Approval is performed in the webapp.
- **Flow:** validate → verify-cr → terraform → ansible → integrations → smoke → mark-deployed → notify
- **Inputs:** `app_name`, `change_request_id` (REQUIRED), `change_title`, `change_description`, `inject_failure`

### Common: failure injection

The `inject_failure` input on any of the three workflows forces a chosen stage (`terraform-provision`, `ansible-configure`, or `splunk-integrate`) to exit non-zero **on attempt 1 only** (guarded by `github.run_attempt == 1`). Click **Re-run failed jobs** → attempt 2 skips the injection guard → workflow succeeds. Demonstrates state preservation across retries.

## Change Management Webapp

Live at: **https://cube-earth-labs.github.io/github-actions-orchestration/**

A mock change-management system, served from `docs/index.html` via GitHub Pages. Backed by GitHub Issues (each CR is an issue labeled `change-request` + `status:*` + `env:*`). Lets you:

- Create a change request (title, description, environment)
- Approve / reject pending CRs
- See history of deployed CRs

**Auth:** the webapp uses a GitHub Personal Access Token (PAT) with `repo` scope, stored in `localStorage`. Reads work anonymously; create/approve actions need a PAT. [Create one →](https://github.com/settings/tokens/new?scopes=repo&description=cube-earth-labs%20CMS%20demo)

## Folder Layout

```
github-actions-orchestration/
├── README.md                     # This file
├── docs/
│   └── index.html                # Change-management webapp (served by GitHub Pages from /docs)
└── .github/
    └── workflows/
        ├── deploy-sandbox.yml        # Deploy to Sandbox    (human-gate, no CR)
        ├── deploy-staging.yml        # Deploy to Staging    (auto-creates a CR, deploys, marks deployed)
        └── deploy-production.yml     # Deploy to Production (verifies an approved CR, deploys, marks deployed)
```

## Run Tasks vs Actions vs AAP Kickoff — Decision Matrix

| Need | Use |
| --- | --- |
| **Inline policy / compliance check during a Terraform run** | **TFE Run Tasks** (Sentinel, OPA, Checkov, etc.) |
| **Outside-of-Terraform action triggered by a run state (e.g., on policy pass, on apply complete)** | **Terraform Actions** (when GA) or webhook → orchestrator |
| **End-to-end workflow that spans Terraform + Ansible + ITSM + observability** | **GitHub Actions** (this demo) |
| **Config management on already-provisioned hosts (drift, patching)** | **AAP** (its native job) |
| **Long-running, stateful workflow that survives restarts** | **GHA with `workflow_dispatch` + reusable workflows** |

## Sample Workflow — Production Deploy

```mermaid
graph TD
    A[Trigger: PR or workflow_dispatch] --> B[Validate request payload]
    B --> C{CRQ approval gate}
    C -->|Approved| D[Terraform: provision infra]
    C -->|Rejected| Z[Notify requester, exit]
    D --> E[Ansible: configure hosts]
    E --> F[Integrate Splunk forwarder]
    E --> G[Register Dynatrace OneAgent]
    E --> H[Update CMDB]
    F --> I[Smoke tests]
    G --> I
    H --> I
    I --> J[Notify success / close CRQ]
```

The serial path is `validate → CRQ → provision → configure`. The integration steps (Splunk, Dynatrace, CMDB) run in **parallel** after configuration. The final notification step fans in once all three integrations are healthy.

## How to Run This Demo

### One-time setup

1. In **Settings → Environments**, create the `change-approval` environment with a required reviewer — used as the gate in `deploy-sandbox.yml`.
2. Open the webapp at https://cube-earth-labs.github.io/github-actions-orchestration/ and paste your GitHub PAT in **Settings**. Labels (`change-request`, `status:*`, `env:*`) are already created in the repo.

### Sandbox deploy (no change request)

1. Actions → **Deploy to Sandbox** → **Run workflow**
2. Fill in: `change_title` and `change_description`.
3. The `approval` job pauses with a "Waiting" status. Click **Review deployments** → Approve.
4. Pipeline proceeds to terraform → ansible → integrations → smoke → notify.

### Staging deploy (auto-creates a change request)

1. Actions → **Deploy to Staging** → **Run workflow**
2. Fill in: `change_title` and `change_description` (these populate the new CR).
3. Pipeline immediately creates a CR (status:approved), then deploys. After smoke-tests, the CR is marked `status:deployed`.
4. Open the webapp to see the CR appear in **Approved** → then move to **Deployed** after the run completes.

### Production deploy (requires an approved CR from the webapp)

1. **In the webapp:** create a CR with environment = `production`. It enters **Pending Approval**.
2. **In the webapp:** click **Approve** on that CR. It moves to **Approved**.
3. **Note the issue number** (e.g., `#42`).
4. Actions → **Deploy to Production** → **Run workflow** → `change_request_id: 42`, plus `change_title` and `change_description`.
5. The `verify-cr` job fetches the issue and validates its labels. If approved + production, the pipeline proceeds. If not, it fails fast with a clear message.
6. After smoke-tests, the CR is marked `status:deployed`.

### Demo the Re-run failed jobs pattern

Run any of the three workflows with `inject_failure: splunk-integrate`. Splunk turns red on attempt 1, dynatrace and cmdb stay green, smoke-tests is skipped. Click **Re-run failed jobs** in the top-right of the run page → attempt 2 skips the injection guard → splunk succeeds → downstream proceeds. Only the failed job and its skipped downstream re-execute; the parallel siblings (dynatrace, cmdb) keep their original-run outcome.
