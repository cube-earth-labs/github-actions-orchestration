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
   - **Demo:** run `saas-onboarding.yml` with `inject_failure: splunk-integrate`. Splunk fails on attempt 1, dynatrace and cmdb stay green, smoke-tests is skipped. Click **Re-run failed jobs** — attempt 2 skips the injection guard (`github.run_attempt == 1` is now false), splunk succeeds, and downstream proceeds. Terraform, ansible, dynatrace, and cmdb are NOT re-run — their original outcome is preserved.

2. **Can you see the order in which things happened visually?**
   - **Job DAG view:** the Actions tab renders every workflow as a graph — boxes for jobs, arrows for `needs:` dependencies, color-coded by status
   - **Demo:** `saas-onboarding.yml` exercises this with steps that fan out and converge

3. **Can you chain processes serially and in parallel?**
   - **Serial:** `needs: [upstream-job]` declares a dependency
   - **Parallel:** any two jobs without a `needs:` chain between them run concurrently
   - **Fan-out:** matrix strategy (`strategy.matrix:`) spawns parallel job instances
   - **Fan-in:** a downstream job with `needs: [a, b, c]` blocks until all complete

## What Each Pipeline Demonstrates

### 1. `saas-onboarding.yml` — Flagship End-to-End Demo

- **Workflow:** validate → CRQ approval → Terraform provision → Ansible configure → Splunk + Dynatrace + CMDB (parallel) → smoke tests → notify
- **Demonstrates:** all three customer questions in one realistic run — serial chain, parallel branches, fan-in, and a CRQ-style approval gate
- **Failure injection:** the `inject_failure` workflow_dispatch input forces a chosen stage (`terraform-provision`, `ansible-configure`, or `splunk-integrate`) to exit non-zero **on the first attempt only**. The injection step is gated on `github.run_attempt == 1`, so clicking **Re-run failed jobs** skips the guard and lets the workflow complete. Demonstrates Re-run-failed-jobs *with state preservation* — failed stage re-runs in isolation, parallel siblings stay green from the original run.

### 2. `manual-approval-gates.yml` — CRQ-Style Approval

- **Pattern:** GitHub Actions environments with required reviewers pause the workflow until an approver clicks "Approve"
- **Multi-stage support:** the demo uses *two* environments (`staging-approval`, `production-approval`) so different approver groups can gate different stages — dev team approves staging, CAB approves prod
- **Use case:** change-management gates, ServiceNow CRQ-equivalent approvals

## Folder Layout

```
github-actions-orchestration/
├── README.md                     # This file
└── .github/
    └── workflows/
        ├── saas-onboarding.yml
        └── manual-approval-gates.yml
```

The `.github/workflows/` layout matches what GitHub expects, so the demo can be dropped into any repo and runs immediately.

## Run Tasks vs Actions vs AAP Kickoff — Decision Matrix

| Need | Use |
| --- | --- |
| **Inline policy / compliance check during a Terraform run** | **TFE Run Tasks** (Sentinel, OPA, Checkov, etc.) |
| **Outside-of-Terraform action triggered by a run state (e.g., on policy pass, on apply complete)** | **Terraform Actions** (when GA) or webhook → orchestrator |
| **End-to-end workflow that spans Terraform + Ansible + ITSM + observability** | **GitHub Actions** (this demo) |
| **Config management on already-provisioned hosts (drift, patching)** | **AAP** (its native job) |
| **Long-running, stateful workflow that survives restarts** | **GHA with `workflow_dispatch` + reusable workflows** |

## Sample Workflow — SaaS Onboarding

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

1. Fork or clone this repo.
2. In **Settings → Environments**, create three environments and add required reviewers to each:
   - `change-approval` — used by `saas-onboarding.yml` as the CRQ gate
   - `staging-approval` — used by `manual-approval-gates.yml`
   - `production-approval` — used by `manual-approval-gates.yml`
3. Open the **Actions** tab. Pick a workflow on the left, click **Run workflow**, and watch the graph render.
4. To demo the failed-job re-run pattern: run `SaaS Onboarding (Demo)` with `inject_failure: splunk-integrate`. Splunk turns red, dynatrace and cmdb stay green, smoke-tests is skipped. Click **Re-run failed jobs** in the top-right of the run page — only splunk re-runs; the parallel siblings are preserved from the original run.
