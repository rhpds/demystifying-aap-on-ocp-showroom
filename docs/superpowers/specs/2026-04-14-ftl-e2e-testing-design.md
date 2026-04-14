# FTL Implementation + E2E Testing Design

**Date:** 2026-04-14  
**Lab:** Demystifying AAP on OCP (Summit)  
**Author:** Brandon Thomas  
**Reference implementation:** LB1390 (AAP + HashiCorp) — `rhpds/zt-ans-bu-hashi-aap` branch `lb1390-validation`

---

## Background

FTL (Finish The Lab) was previously implemented and then fully reverted on 2026-04-07 via four revert commits. The `runtime-automation/` directory, all solve/validation playbooks, and the `ui-config.yml` FTL configuration were removed. This spec covers re-implementing FTL with a layered e2e testing strategy as the primary focus.

The reference guide from another lab developer (LB1390) provides proven patterns for solve/validate. Key findings from that guide are incorporated throughout this spec.

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
- **Showroom:** Deployed per-attendee environment; the FTL runner executes **inside the Showroom pod** via `ansible-runner` — not on the bastion host and not via SSH

---

## How the Runner Works (Critical Architecture)

The solve/validate buttons do NOT run on the bastion host. They execute inside the **Showroom pod** via `ansible-runner`. The runner:

- Has no Ansible collections installed — all API calls must use `ansible.builtin.uri`
- Calls the Kubernetes API directly at `https://kubernetes.default.svc` using the pod's service account token at `/var/run/secrets/kubernetes.io/serviceaccount/token`
- Receives these environment variables automatically:

```bash
GUID=<lab-guid>
DOMAIN=apps.ocpvdev01.rhdp.net
VAULT_PASSWORD=<vault-password>         # auto-decrypts secrets.yaml
ANSIBLE_USER=rhel
ANSIBLE_PASSWORD=<lab-ssh-password>
ANSIBLE_PORT=22
```

- Detects available stages by scanning for `{stage}-control.sh` stub files on startup — **the `.sh` stubs must exist or the buttons will not appear**
- Exposes a REST API used by the Showroom UI buttons and available for manual testing:

```bash
# Check what stages are detected
curl -sk $SHOWROOM/runner/api/config | python3 -m json.tool

# Trigger a stage manually
curl -sk -X POST $SHOWROOM/runner/api/module-02/validation
curl -sk -X POST $SHOWROOM/runner/api/module-02/solve

# Poll job status (use Job_id from POST response)
curl -sk $SHOWROOM/runner/api/job/{Job_id} | python3 -m json.tool
```

Where `SHOWROOM=https://showroom-{GUID}-1.apps.ocpvdev01.rhdp.net`

---

## What FTL Looks Like (User-Facing)

On each scenario page in Showroom, two buttons appear:

| Button | Behavior |
|--------|----------|
| **Check** (Next) | Runs `validation.yml` for that scenario against the attendee's live OCP cluster. Displays a checklist of pass/fail items. |
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
│  Full scenario flow per module:                     │
│  apply-broken-state → validate(fail) →              │
│  solve → validate(pass)                             │
│  Triggered before Summit. Human-approved gate.      │
└────────────────────────▼────────────────────────────┘
                         │ passes
┌────────────────────────▼────────────────────────────┐
│  Layer 3 — Runtime (live at Summit)                 │
│  FTL validate/solve buttons + structured output     │
│  Facilitators debug via Showroom runner API         │
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
- Missing `hosts:`, `connection: local`, or `gather_facts: false` on play wrapper (the specific issue that caused the previous revert)
- Typos in variable names
- Use of collection-based modules (must catch `ansible.builtin.uri` being replaced with collection modules — the runner has no collections installed)
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

The integration gate tests **what FTL owns** — the `runtime-automation/` validate and solve playbooks. It does not test the bastion-side provisioning playbooks (`site.yml`), which live on the bastion host. The deploy/break provisioning correctness is verified separately during pre-Summit dry-run sessions with a live RHDP lab.

### Test Flow

Scenarios run **sequentially** (SNO cannot handle concurrent AAP deployments):

```
For each scenario (2 → 3 → 4 → 5 → 6 → 7):
  1. apply-broken-state:  oc apply -f tests/fixtures/scenario-N/broken-state.yml
  2. validate:            ansible-playbook runtime-automation/main.yml
                          (module_directory=module-0N, module_stage=validation)
                          → assert FAIL
  3. solve:               ansible-playbook runtime-automation/main.yml
                          (module_directory=module-0N, module_stage=solve)
  4. validate:            ansible-playbook runtime-automation/main.yml
                          (module_directory=module-0N, module_stage=validation)
                          → assert PASS
  5. teardown:            oc delete namespace scenario-N
```

**Broken-state fixtures** are YAML manifests stored in `tests/fixtures/scenario-N/broken-state.yml`. They apply exactly the broken resource state (wrong storage class, bad CR spec, etc.) that `site.yml -t break` would produce on the bastion — applied directly via `oc` so no bastion access is needed. Each fixture is derived by inspecting what the bastion break role actually does on a real cluster.

### Key Design Decisions

- **Both directions tested:** Validation must correctly detect broken state (fail) AND solved state (pass). A validation that always passes is as broken as one that always fails.
- **Fixtures mirror real break state:** Each `broken-state.yml` matches exactly what `site.yml -t break` produces, so the test reflects reality.
- **Clean teardown between scenarios:** `oc delete namespace scenario-N` between each scenario gives a clean slate.
- **Credentials never hardcoded:** `OCP_API_URL` and `OCP_TOKEN` stored as GitHub repository secrets only.
- **Job summary output:** GitHub Actions job summary shows a pass/fail table per scenario so the result is visible without reading raw logs.

### Output Example

```
| Scenario | Broken State | Validate(fail) | Solve | Validate(pass) |
|----------|--------------|----------------|-------|----------------|
| 2        | ✅           | ✅             | ✅    | ✅             |
| 3        | ✅           | ✅             | ❌    | —              |
| 4        | ✅           | ✅             | ✅    | ✅             |
```

### Implementation

New files:
- `.github/workflows/ftl-integration-test.yml`
- `tests/fixtures/scenario-2/broken-state.yml` through `tests/fixtures/scenario-7/broken-state.yml`

---

## Layer 3: Runtime Observability

**When:** Live at Summit, during the lab session  
**Cost:** No extra infrastructure — structured output formatting in playbooks + the Showroom runner API that already exists

### Primary Debugging Tool — Showroom Runner API

When an attendee reports an issue, the facilitator uses these curl commands against that attendee's Showroom URL:

```bash
# Set the attendee's Showroom URL
SHOWROOM=https://showroom-{GUID}-1.apps.ocpvdev01.rhdp.net

# Check which stages the runner has detected (confirms .sh stubs are loaded)
curl -sk $SHOWROOM/runner/api/config | python3 -m json.tool

# Manually trigger validation to see the exact output
curl -sk -X POST $SHOWROOM/runner/api/module-02/validation
# Returns: {"job_id": "abc123", ...}

# Poll for the full result
curl -sk $SHOWROOM/runner/api/job/abc123 | python3 -m json.tool
```

This gives the facilitator the exact same output the attendee sees, plus the full Ansible stdout — far more useful than reading pod logs.

### Standard Output Format

Every `validation.yml` and `solve.yml` must end with a human-readable summary written to `{{ job_info_dir }}/validation_failure.out`:

```
[SCENARIO N] <Scenario Name>
Status: PASS | FAIL
------------------------------
[OK] <check description>
[X]  <check description>
------------------------------
<Actionable next step if FAIL>
```

### Rules

- A facilitator must be able to determine what failed within 5 seconds of reading the output.
- No raw Ansible JSON or stack traces in user-facing output.
- `[X]` items must tell the attendee **what to fix**, not just that something is wrong.
- Output is written to `{{ job_info_dir }}/validation_failure.out` and displayed in the Showroom UI.

### Runner Pod Logs (fallback)

If the runner API is insufficient, full ansible-runner stdout is available:

```bash
oc get pods -n sandbox-{guid}-1-zt-ansiblebu | grep showroom

oc exec -n sandbox-{guid}-1-zt-ansiblebu {pod} -c runner -- bash -c "
  find /app/logs/runtime-automation/module-02/validation \
    -name 'ansible_runner.stdout' | sort | tail -1 | xargs tail -30
"
```

---

## FTL Re-Implementation

### Files to Create

```
runtime-automation/
├── main.yml                          # Entry point — routes to module/stage
├── secrets.yaml                      # Vault-encrypted secrets placeholder
├── module-02/
│   ├── solve-control.sh              # Stub — required for runner API detection
│   ├── validation-control.sh         # Stub — required for runner API detection
│   ├── solve.yml                     # Solve logic
│   └── validation.yml                # Validation logic
├── module-03/ ... (same structure)
├── module-04/ ... (same structure)
├── module-05/ ... (same structure)
├── module-06/ ... (same structure)
└── module-07/ ... (same structure)
```

### main.yml (exact pattern — copy from LB1390)

```yaml
---
- name: Run module stage via direct API calls
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    _module_dir: "{{ module_dir | default(module_directory | default('')) }}"

  tasks:
    - name: Load secrets (vault-decrypted via VAULT_PASSWORD env var)
      ansible.builtin.include_vars: secrets.yaml
      ignore_errors: true

    - name: Include module tasks
      ansible.builtin.include_tasks: "{{ _module_dir }}/{{ module_stage }}.yml"
```

### Shell Script Stubs (required for button detection)

The runner scans for `{stage}-control.sh` files on startup. If they don't exist, the buttons will not appear in the UI. Each stub is identical:

```bash
#!/bin/bash
# Stub only — actual logic is in solve.yml / validation.yml (Ansible)
exit 0
```

**Important:** If `.sh` stubs are added to a **running** pod, the runner must be restarted before it detects them:
```bash
oc exec -n sandbox-{guid}-1-zt-ansiblebu {pod} -c runner -- kill 1
```
For fresh lab orders, this is not needed.

### ui-config.yml

```yaml
---
type: showroom
default_width: 30
persist_url_state: true

antora:
  name: modules
  dir: www
  modules:
    - name: module-02
      label: "Scenario #2: Automation Hub Issues"
      solveButton: true
      scripts:
        - solve
        - validation

    - name: module-03
      label: "Scenario #3: Gateway Issues"
      solveButton: true
      scripts:
        - solve
        - validation

    - name: module-04
      label: "Scenario #4: Gateway Memory Issues"
      solveButton: true
      scripts:
        - solve
        - validation

    - name: module-05
      label: "Scenario #5: Automation Hub Sync Issues"
      solveButton: true
      scripts:
        - solve
        - validation

    - name: module-06
      label: "Scenario #6: Automation Controller Job Issues"
      solveButton: true
      scripts:
        - solve
        - validation

    - name: module-07
      label: "Scenario #7: Ansible Lightspeed (ALIA) Issues"
      solveButton: true
      scripts:
        - solve
        - validation
```

### Playbook Standards (all modules)

- **Play wrapper required on every playbook:** `hosts: localhost`, `connection: local`, `gather_facts: false`
- **`uri` module only** — the runner has no Ansible collections installed; never use collection-based modules
- **Kubernetes API calls** use the in-cluster service account token:
  ```yaml
  headers:
    Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
  validate_certs: false
  ```
- **Every validation.yml** must write to `{{ job_info_dir }}/validation_failure.out` and call `ansible.builtin.fail` when not passed
- **Every solve.yml** must be idempotent — check before creating/patching
- All playbooks end with the standard Layer 3 output format

### Integer ID Gotcha

When posting body fields that require integer IDs to any API, force the type:
```yaml
body: "{{ {'id': my_id | int} }}"
```
Passing as a string causes the API to return `"id" field must be an integer`.

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

## Common Issues Reference

| Issue | Cause | Fix |
|-------|-------|-----|
| Solve/Check button not appearing | Missing `.sh` stub file, or runner not restarted after adding it | Create stub; restart runner if already running |
| "stage not found" error | `.sh` stub not detected | Check `curl -sk $SHOWROOM/runner/api/config` |
| Empty error dialog in UI | `validation_failure.out` not written | Verify `job_info_dir` is defined and task runs |
| Missing `hosts:` block | Play wrapper missing | All playbooks require `hosts: localhost` wrapper |
| Collection module fails | Runner has no collections | Replace with `ansible.builtin.uri` |

---

## Success Criteria

- [ ] Layer 1 CI runs on every push and blocks merges with syntax errors
- [ ] Layer 2 integration test passes all 6 scenarios against a real SNO cluster
- [ ] FTL Check and Solve buttons are visible on all 6 scenario pages in Showroom
- [ ] Validation correctly detects broken state (FAIL) and solved state (PASS) for every scenario
- [ ] Solve correctly fixes every scenario such that a subsequent validation PASSes
- [ ] Every validation/solve output is human-readable within 5 seconds
- [ ] Facilitators can debug any attendee issue using the Showroom runner API curl commands
- [ ] No attendee is blocked from completing any scenario due to a playbook bug

---

## Out of Scope

- Multi-user shared cluster support (each attendee has their own SNO — not needed)
- A live dashboard or database for tracking attendee progress
- Automated provisioning of the test SNO cluster (manual RHDP provision for Layer 2)
- SSH-based or bastion-based execution (runner is inside the Showroom pod)
