# FTL Implementation + E2E Testing Design

**Date:** 2026-04-14  
**Lab:** Demystifying AAP on OCP (Summit)  
**Author:** Brandon Thomas  

---

## Background

FTL (Finish The Lab) was previously implemented and then fully reverted on 2026-04-07 via four revert commits. The `runtime-automation/` directory, all solve/validation playbooks, and the `ui-config.yml` FTL configuration were removed. This spec covers re-implementing FTL with a layered e2e testing strategy as the primary focus.

---

## Goals

1. Re-implement FTL so attendees see Check (validate) and Solve buttons on each scenario page.
2. Ensure every attendee can complete all 6 scenarios from start to finish without disruption at Summit.
3. Catch playbook issues before Summit via automated testing.
4. Give facilitators visibility into stuck attendees on the day.

---

## Deployment Context

- **Platform:** Red Hat Demo Platform (RHDP)
- **Per-attendee environment:** One dedicated Single Node OpenShift (SNO) cluster per attendee
- **Isolation:** Full — attendees cannot conflict with each other at the infrastructure level
- **Namespaces:** Hardcoded per scenario (`scenario-2` through `scenario-7`) — safe because each attendee has their own cluster
- **Showroom:** Deployed per-attendee environment, serves the lab guide with FTL buttons wired to the attendee's own cluster

---

## What FTL Looks Like (User-Facing)

On each scenario page in Showroom, two buttons appear:

| Button | Behavior |
|--------|----------|
| **Check** | Runs `validation.yml` for that scenario against the attendee's live OCP cluster. Displays a checklist of pass/fail items. |
| **Solve** | Runs `solve.yml` for that scenario, auto-fixing the broken state. Attendee can then click Check to confirm. |

Example validation output displayed in the UI:
```
Module 2: Automation Hub Storage
=================================
[OK] Namespace scenario-2 exists
[X]  AutomationHub CR has file_storage_storage_class set
[X]  Hub file storage PVC is Bound
[OK] Hub pods Running (0 pods)

Complete the missing tasks and try again.
```

---

## Architecture: Three Layers

```
┌─────────────────────────────────────────────────────┐
│  Layer 1 — Syntax Gate (GitHub Actions, every push) │
│  yamllint + ansible-lint + syntax-check             │
│  No cluster needed. Blocks bad merges.              │
└────────────────────────┬────────────────────────────┘
                         │ passes
┌────────────────────────▼────────────────────────────┐
│  Layer 2 — Integration Gate (manual, real SNO)      │
│  Full scenario flow per module: deploy→break→       │
│  validate(fail)→solve→validate(pass)                │
│  Triggered before Summit. Human-approved gate.      │
└────────────────────────▼────────────────────────────┘
                         │ passes
┌────────────────────────▼────────────────────────────┐
│  Layer 3 — Runtime (live at Summit)                 │
│  FTL validate/solve buttons + structured output     │
│  Facilitators can see exactly where users are stuck │
└─────────────────────────────────────────────────────┘
```

---

## Layer 1: Syntax Gate

**Trigger:** Every push to `main`, every pull request  
**Runner:** `ubuntu-latest` (no cluster access required)  
**Duration:** Under 2 minutes  

### Checks (in order)

1. **`yamllint`** — validates all YAML files under `runtime-automation/` and `ui-config.yml`
2. **`ansible-lint`** — checks for deprecated module usage, bad task structure, missing required fields
3. **`ansible-playbook --syntax-check`** — runs on:
   - `runtime-automation/main.yml`
   - `runtime-automation/module-*/solve.yml`
   - `runtime-automation/module-*/validation.yml`

### What It Catches

- Malformed YAML (missing colons, bad indentation)
- Missing `hosts:` or `tasks:` blocks (the specific issue that caused the previous revert)
- Typos in variable names
- Deprecated Ansible module names
- Broken task includes

### What It Does Not Catch

- Whether playbook logic is correct against a real cluster (Layer 2's job)

### Implementation

New file: `.github/workflows/ftl-syntax-check.yml`

---

## Layer 2: Integration Gate

**Trigger:** Manual (`workflow_dispatch`) — run before Summit, and after any change to `runtime-automation/`  
**Runner:** `ubuntu-latest` with OCP credentials injected via GitHub secrets  
**Secrets required:** `OCP_API_URL`, `OCP_TOKEN`  

### Scope Boundary

The integration gate tests **what FTL owns** — the `runtime-automation/` validate and solve playbooks. It does not test the bastion-side provisioning playbooks (`site.yml`), which live on the bastion host and are not accessible from GitHub Actions. The deploy/break provisioning correctness is verified separately during pre-Summit dry-run sessions.

### Test Flow

Scenarios run **sequentially** (SNO cannot handle concurrent AAP deployments):

```
For each scenario (2 → 3 → 4 → 5 → 6 → 7):
  1. apply-broken-state:  oc apply -f tests/fixtures/scenario-N/broken-state.yml
  2. validate:            run validation.yml → assert FAIL
  3. solve:               run solve.yml
  4. validate:            run validation.yml → assert PASS
  5. teardown:            oc delete namespace scenario-N
```

**Broken-state fixtures** are YAML manifests stored in `tests/fixtures/scenario-N/broken-state.yml` in this repo. They apply exactly the broken resource state (wrong storage class, bad CR spec, etc.) that `site.yml -t break` would produce on the bastion — but applied directly via `oc` so no bastion access is needed.

### Key Design Decisions

- **Both directions tested:** Validation must correctly detect broken state (fail) AND solved state (pass). A validation that always passes is as broken as one that always fails.
- **Fixtures mirror real break state:** Each `broken-state.yml` is derived from inspecting what `site.yml -t break` actually does on a real cluster, ensuring the test reflects reality.
- **Clean teardown between scenarios:** `oc delete namespace scenario-N` between each scenario gives a clean slate.
- **Credentials never hardcoded:** `OCP_API_URL` and `OCP_TOKEN` stored as GitHub repository secrets only.
- **Job summary output:** GitHub Actions job summary shows a pass/fail table per scenario so the result is visible without reading raw logs.

### Output Example

```
| Scenario | Deploy | Break | Validate(fail) | Solve | Validate(pass) |
|----------|--------|-------|----------------|-------|----------------|
| 2        | ✅     | ✅    | ✅             | ✅    | ✅             |
| 3        | ✅     | ✅    | ✅             | ❌    | —              |
| 4        | ✅     | ✅    | ✅             | ✅    | ✅             |
```

### Implementation

New files:
- `.github/workflows/ftl-integration-test.yml`
- `tests/fixtures/scenario-2/broken-state.yml` through `tests/fixtures/scenario-7/broken-state.yml`

---

## Layer 3: Runtime Observability

**When:** Live at Summit, during the lab session  
**Cost:** No extra infrastructure — structured output formatting in the playbooks only  

### Standard Output Format

Every `validation.yml` and `solve.yml` must end with a human-readable summary block:

```
[SCENARIO N] <Scenario Name>
Timestamp: <ISO 8601>
Status: PASS | FAIL
------------------------------
[OK] <check description>
[X]  <check description>
------------------------------
<Next step message if FAIL>
```

### Rules

- A facilitator must be able to determine what failed within 5 seconds of reading the output.
- No raw Ansible JSON or stack traces in user-facing output.
- Failure messages tell the attendee what to fix, not just that something is wrong.
- Output is visible in the Showroom UI (attendee) and Showroom server logs (facilitator).

### Implementation

Standardize the `validation_output` fact and failure message in every existing `validation.yml`. Add equivalent summary output to every `solve.yml`.

---

## FTL Re-Implementation

### Files to Create/Restore

```
runtime-automation/
├── main.yml                          # Entry point — routes to module/stage
├── secrets.yaml                      # Vault-encrypted secrets placeholder
├── module-02/
│   ├── solve.yml
│   ├── solve-control.sh
│   ├── validation.yml
│   └── validation-control.sh
├── module-03/ ... (same structure)
├── module-04/ ... (same structure)
├── module-05/ ... (same structure)
├── module-06/ ... (same structure)
└── module-07/ ... (same structure)
```

### ui-config.yml

Wire all 6 scenarios to their runtime-automation module:

```yaml
antora:
  modules:
    - name: module-02
      label: "Scenario #2: Automation Hub Issues"
      solveButton: true
      scripts:
        - solve
        - validation
    # ... modules 03-07 same pattern
```

### Playbook Standards (applied to all modules)

- All playbooks must have a `hosts:`, `connection: local`, `gather_facts: false` play wrapper.
- All Kubernetes API calls use the in-cluster service account token at `/var/run/secrets/kubernetes.io/serviceaccount/token`.
- All playbooks end with the standard Layer 3 output format.
- `main.yml` routes via `module_directory` and `module_stage` environment variables injected by Showroom.

---

## Scenarios Reference

| Module | Scenario | Component | Namespace |
|--------|----------|-----------|-----------|
| module-02 | Scenario #2 | Automation Hub Storage | scenario-2 |
| module-03 | Scenario #3 | Gateway Issues | scenario-3 |
| module-04 | Scenario #4 | Gateway Memory Issues | scenario-4 |
| module-05 | Scenario #5 | Automation Hub Sync Issues | scenario-5 |
| module-06 | Scenario #6 | Automation Controller Job Issues | scenario-6 |
| module-07 | Scenario #7 | Ansible Lightspeed (ALIA) Issues | scenario-7 |

---

## Success Criteria

- [ ] Layer 1 CI runs on every push and blocks merges with syntax errors
- [ ] Layer 2 integration test runs successfully against a real SNO cluster for all 6 scenarios
- [ ] FTL Check and Solve buttons are visible and functional on all 6 scenario pages in Showroom
- [ ] Validation correctly detects broken state (FAIL) and solved state (PASS) for every scenario
- [ ] Solve correctly fixes every scenario such that a subsequent validation PASS
- [ ] Every validation/solve output is human-readable within 5 seconds
- [ ] No attendee is blocked from completing any scenario due to a playbook bug

---

## Out of Scope

- Multi-user shared cluster support (each attendee has their own SNO — not needed)
- A live dashboard or database for tracking attendee progress
- Automated provisioning of the test SNO cluster (manual RHDP provision for Layer 2)
