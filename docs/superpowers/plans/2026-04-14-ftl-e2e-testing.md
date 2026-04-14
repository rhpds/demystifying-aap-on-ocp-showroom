# FTL Implementation + E2E Testing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Re-implement FTL (solve/validate buttons) for all 6 lab scenarios with a layered e2e testing strategy ensuring Summit attendees can complete the lab without disruption.

**Architecture:** FTL runs inside the Showroom pod via `ansible-runner`. Validation and solve playbooks use `ansible.builtin.uri` (no collections installed in runner) to call the Kubernetes API directly using the pod's in-cluster service account token. Three testing layers: syntax gate CI (every push), integration gate (manual pre-Summit), runtime observability (facilitator curl commands on the day).

**Tech Stack:** Ansible (uri module only), Kubernetes REST API, GitHub Actions, yamllint, ansible-lint, bash stubs

---

## File Map

### Create (new files)

```
runtime-automation/
├── main.yml
├── secrets.yaml
├── module-02/
│   ├── solve-control.sh
│   ├── validation-control.sh
│   ├── solve.yml
│   └── validation.yml
├── module-03/
│   ├── solve-control.sh
│   ├── validation-control.sh
│   ├── solve.yml
│   └── validation.yml
├── module-04/
│   ├── solve-control.sh
│   ├── validation-control.sh
│   ├── solve.yml
│   └── validation.yml
├── module-05/
│   ├── solve-control.sh
│   ├── validation-control.sh
│   ├── solve.yml
│   └── validation.yml
├── module-06/
│   ├── solve-control.sh
│   ├── validation-control.sh
│   ├── solve.yml
│   └── validation.yml
└── module-07/
    ├── solve-control.sh
    ├── validation-control.sh
    ├── solve.yml
    └── validation.yml

.github/workflows/
├── ftl-syntax-check.yml
└── ftl-integration-test.yml
```

### Modify (existing files)

```
ui-config.yml    ← add antora.modules block for all 6 scenarios
```

---

## Important Context Before You Start

- **No Ansible collections in runner.** Every task must use `ansible.builtin.uri`. Never use collection-based modules (e.g. `kubernetes.core.k8s`). The syntax-check CI will catch this.
- **Play wrapper required on every `.yml` playbook.** Each file must start with a `- name:` / `hosts: localhost` / `connection: local` / `gather_facts: false` block. Missing this was the exact cause of the previous revert.
- **`.sh` stubs are mandatory.** The Showroom runner scans for `{stage}-control.sh` files on startup to detect which buttons to show. If the stub doesn't exist, the button doesn't appear — no error, just silence.
- **in-cluster token path:** `lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token')`
- **`job_info_dir` may be undefined** in test runs — always guard writes with `when: job_info_dir is defined`.
- **Scenarios 2 uses `scenario-2` namespace. Scenarios 3–7 use `aap-rh1` namespace.**

---

## Task 1: Foundation — main.yml, secrets.yaml, all stubs

**Files:**
- Create: `runtime-automation/main.yml`
- Create: `runtime-automation/secrets.yaml`
- Create: `runtime-automation/module-02/solve-control.sh`
- Create: `runtime-automation/module-02/validation-control.sh`
- Create: `runtime-automation/module-03/solve-control.sh`
- Create: `runtime-automation/module-03/validation-control.sh`
- Create: `runtime-automation/module-04/solve-control.sh`
- Create: `runtime-automation/module-04/validation-control.sh`
- Create: `runtime-automation/module-05/solve-control.sh`
- Create: `runtime-automation/module-05/validation-control.sh`
- Create: `runtime-automation/module-06/solve-control.sh`
- Create: `runtime-automation/module-06/validation-control.sh`
- Create: `runtime-automation/module-07/solve-control.sh`
- Create: `runtime-automation/module-07/validation-control.sh`

- [ ] **Step 1: Create main.yml**

```yaml
# runtime-automation/main.yml
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

- [ ] **Step 2: Create secrets.yaml placeholder**

```yaml
# runtime-automation/secrets.yaml
# =============================================================================
# Runtime Automation Secrets — Ansible Vault encrypted
# =============================================================================
# Encrypt with: ansible-vault encrypt secrets.yaml
# The VAULT_PASSWORD env var is injected by Showroom and auto-decrypts this file.
# =============================================================================

# No lab-specific secrets required at this time.
# Add vault-encrypted credentials here if needed in future.
```

- [ ] **Step 3: Create all 12 stub shell scripts**

Each file is identical. Create all 12:
- `runtime-automation/module-02/solve-control.sh`
- `runtime-automation/module-02/validation-control.sh`
- `runtime-automation/module-03/solve-control.sh`
- `runtime-automation/module-03/validation-control.sh`
- `runtime-automation/module-04/solve-control.sh`
- `runtime-automation/module-04/validation-control.sh`
- `runtime-automation/module-05/solve-control.sh`
- `runtime-automation/module-05/validation-control.sh`
- `runtime-automation/module-06/solve-control.sh`
- `runtime-automation/module-06/validation-control.sh`
- `runtime-automation/module-07/solve-control.sh`
- `runtime-automation/module-07/validation-control.sh`

Content for each (swap the comment for the actual stage/module):

```bash
#!/bin/bash
# Stub only — actual solve logic is in solve.yml (Ansible)
# Required for Showroom runner API stage detection.
exit 0
```

Make all stubs executable:
```bash
chmod +x runtime-automation/module-*/solve-control.sh runtime-automation/module-*/validation-control.sh
```

- [ ] **Step 4: Commit**

```bash
git add runtime-automation/
git commit -m "feat: add FTL foundation — main.yml, secrets placeholder, all stubs"
```

---

## Task 2: ui-config.yml

**Files:**
- Modify: `ui-config.yml`

- [ ] **Step 1: Replace ui-config.yml content**

```yaml
# ui-config.yml
---
type: showroom

# Set the left column width to 30%
default_width: 30
# Persist the URL state so browser refresh doesn't reset the UI
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

- [ ] **Step 2: Verify the YAML parses**

```bash
python3 -c "import yaml; yaml.safe_load(open('ui-config.yml'))" && echo "OK"
```

Expected output: `OK`

- [ ] **Step 3: Commit**

```bash
git add ui-config.yml
git commit -m "feat: wire FTL modules in ui-config.yml for all 6 scenarios"
```

---

## Task 3: Module 02 — Automation Hub Storage

**Scenario:** Automation Hub CR missing `file_storage_storage_class`, causing PVC to use wrong access mode and hub pods to fail.  
**Namespace:** `scenario-2`

**Files:**
- Create: `runtime-automation/module-02/validation.yml`
- Create: `runtime-automation/module-02/solve.yml`

- [ ] **Step 1: Create validation.yml**

```yaml
# runtime-automation/module-02/validation.yml
---
- name: Validate Module 2 — Automation Hub Storage
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Get namespace scenario-2
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/scenario-2"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_namespace

    - name: Get AutomationHub CR
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/automationhub.ansible.com/v1beta1/namespaces/scenario-2/automationhubs"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_ah_cr

    - name: Get PVCs in scenario-2
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/scenario-2/persistentvolumeclaims?labelSelector=app.kubernetes.io%2Fcomponent%3Dstorage"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_pvcs

    - name: Get pods in scenario-2
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/scenario-2/pods"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_pods

    - name: Evaluate checks
      ansible.builtin.set_fact:
        namespace_exists: "{{ r_namespace.status == 200 }}"
        storage_class_set: >-
          {{
            r_ah_cr.status == 200 and
            r_ah_cr.json.items | length > 0 and
            r_ah_cr.json.items[0].spec.file_storage_storage_class is defined and
            r_ah_cr.json.items[0].spec.file_storage_storage_class | length > 0
          }}
        pvc_bound: >-
          {{
            r_pvcs.status == 200 and
            r_pvcs.json.items | selectattr('status.phase', 'equalto', 'Bound') | list | length > 0
          }}
        hub_pods_running: >-
          {{
            r_pods.status == 200 and
            r_pods.json.items | selectattr('status.phase', 'equalto', 'Running') | list | length
          }}

    - name: Set validation output
      ansible.builtin.set_fact:
        validation_output: |
          Module 2: Automation Hub Storage
          =================================
          {{ '[OK]' if namespace_exists else '[X] ' }} Namespace scenario-2 exists
          {{ '[OK]' if storage_class_set else '[X] ' }} AutomationHub CR has file_storage_storage_class set
          {{ '[OK]' if pvc_bound else '[X] ' }} Hub file storage PVC is Bound
          {{ '[OK]' if (hub_pods_running | int) > 0 else '[X] ' }} Hub pods Running ({{ hub_pods_running | default(0) }} pods)
        validation_passed: >-
          {{
            namespace_exists and
            storage_class_set and
            pvc_bound and
            (hub_pods_running | int) > 0
          }}

    - name: Write failure details
      ansible.builtin.copy:
        content: "{{ validation_output }}"
        dest: "{{ job_info_dir }}/validation_failure.out"
      delegate_to: localhost
      when:
        - job_info_dir is defined
        - not validation_passed | bool

    - name: Fail if not passed
      ansible.builtin.fail:
        msg: "Complete the missing tasks and try again."
      when: not validation_passed | bool
```

- [ ] **Step 2: Create solve.yml**

```yaml
# runtime-automation/module-02/solve.yml
---
- name: Solve Module 2 — Automation Hub Storage
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Get AutomationHub CR
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/automationhub.ansible.com/v1beta1/namespaces/scenario-2/automationhubs"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
      register: r_ah_cr

    - name: Check if storage class already set
      ansible.builtin.set_fact:
        storage_class_set: >-
          {{
            r_ah_cr.json.items | length > 0 and
            r_ah_cr.json.items[0].spec.file_storage_storage_class is defined and
            r_ah_cr.json.items[0].spec.file_storage_storage_class | length > 0
          }}
        ah_name: "{{ r_ah_cr.json.items[0].metadata.name }}"
      when: r_ah_cr.json.items | length > 0

    - name: Patch AutomationHub CR to set storage class
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/automationhub.ansible.com/v1beta1/namespaces/scenario-2/automationhubs/{{ ah_name }}"
        method: PATCH
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
          Content-Type: "application/merge-patch+json"
        body_format: json
        body:
          spec:
            file_storage_storage_class: "ocs-external-storagecluster-cephfs"
        validate_certs: false
        status_code: [200, 201]
      when:
        - not storage_class_set | bool
        - ah_name is defined

    - name: Get old RWO PVC
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/scenario-2/persistentvolumeclaims?labelSelector=app.kubernetes.io%2Fcomponent%3Dstorage"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
      register: r_pvcs

    - name: Delete old RWO PVC
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/scenario-2/persistentvolumeclaims/{{ item.metadata.name }}"
        method: DELETE
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 202, 404]
      loop: "{{ r_pvcs.json.items | selectattr('spec.accessModes', 'contains', 'ReadWriteOnce') | list }}"
      when:
        - r_pvcs.json.items | length > 0
        - not storage_class_set | bool

    - name: Get operator manager pod
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/scenario-2/pods?labelSelector=control-plane%3Dcontroller-manager"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
      register: r_operator_pods

    - name: Delete operator manager pod to force reconciliation
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/scenario-2/pods/{{ item.metadata.name }}"
        method: DELETE
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 202, 404]
      loop: "{{ r_operator_pods.json.items }}"
      when:
        - r_operator_pods.json.items | length > 0
        - not storage_class_set | bool

    - name: Show completion message
      ansible.builtin.copy:
        content: |
          [INFO]
          [OK] Module 2 solved

          Fixed:
          - Set file_storage_storage_class to ocs-external-storagecluster-cephfs
          - Deleted old RWO PVC
          - Restarted operator manager to reconcile

          Hub pods should start automatically. Click Next to validate.
        dest: "{{ job_info_dir }}/validation_failure.out"
      delegate_to: localhost
      when: job_info_dir is defined
```

- [ ] **Step 3: Verify syntax**

```bash
ansible-playbook --syntax-check runtime-automation/main.yml \
  -e module_directory=module-02 -e module_stage=validation
ansible-playbook --syntax-check runtime-automation/main.yml \
  -e module_directory=module-02 -e module_stage=solve
```

Expected: no errors, playbook parses cleanly.

- [ ] **Step 4: Commit**

```bash
git add runtime-automation/module-02/
git commit -m "feat: add FTL validation + solve for module-02 (Automation Hub Storage)"
```

---

## Task 4: Module 03 — Gateway Issues

**Scenario:** AAP PostgreSQL Service was deleted from namespace `aap-rh1`, breaking the Gateway's database connection.  
**Namespace:** `aap-rh1`

**Files:**
- Create: `runtime-automation/module-03/validation.yml`
- Create: `runtime-automation/module-03/solve.yml`

- [ ] **Step 1: Create validation.yml**

```yaml
# runtime-automation/module-03/validation.yml
---
- name: Validate Module 3 — Gateway Issues
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Get namespace aap-rh1
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_namespace

    - name: Get AnsibleAutomationPlatform CR
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/aap.ansible.com/v1alpha1/namespaces/aap-rh1/ansibleautomationplatforms"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_aap_cr

    - name: Set postgres service name
      ansible.builtin.set_fact:
        postgres_service_name: "{{ r_aap_cr.json.items[0].metadata.name }}-aap-postgres-15"
      when:
        - r_aap_cr.status == 200
        - r_aap_cr.json.items | length > 0

    - name: Get PostgreSQL Service
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/services/{{ postgres_service_name | default('') }}"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_postgres_svc
      when: postgres_service_name is defined

    - name: Get Gateway pods
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/pods?labelSelector=app.kubernetes.io%2Fname%3Daap-gateway"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_gateway_pods

    - name: Evaluate checks
      ansible.builtin.set_fact:
        namespace_exists: "{{ r_namespace.status == 200 }}"
        postgres_svc_exists: "{{ r_postgres_svc.status | default(404) == 200 }}"
        gateway_pods_running: >-
          {{
            r_gateway_pods.status == 200 and
            r_gateway_pods.json.items | selectattr('status.phase', 'equalto', 'Running') | list | length > 0
          }}

    - name: Set validation output
      ansible.builtin.set_fact:
        validation_output: |
          Module 3: Gateway Issues
          ========================
          {{ '[OK]' if namespace_exists else '[X] ' }} Namespace aap-rh1 exists
          {{ '[OK]' if postgres_svc_exists else '[X] ' }} PostgreSQL Service {{ postgres_service_name | default('') }} exists
          {{ '[OK]' if gateway_pods_running else '[X] ' }} Gateway pod Running
        validation_passed: >-
          {{
            namespace_exists and
            postgres_svc_exists and
            gateway_pods_running
          }}

    - name: Write failure details
      ansible.builtin.copy:
        content: "{{ validation_output }}"
        dest: "{{ job_info_dir }}/validation_failure.out"
      delegate_to: localhost
      when:
        - job_info_dir is defined
        - not validation_passed | bool

    - name: Fail if not passed
      ansible.builtin.fail:
        msg: "Complete the missing tasks and try again."
      when: not validation_passed | bool
```

- [ ] **Step 2: Create solve.yml**

```yaml
# runtime-automation/module-03/solve.yml
---
- name: Solve Module 3 — Gateway Issues
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Get AnsibleAutomationPlatform CR
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/aap.ansible.com/v1alpha1/namespaces/aap-rh1/ansibleautomationplatforms"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
      register: r_aap_cr

    - name: Set resource names
      ansible.builtin.set_fact:
        aap_cr_name: "{{ r_aap_cr.json.items[0].metadata.name }}"
        postgres_service_name: "{{ r_aap_cr.json.items[0].metadata.name }}-aap-postgres-15"
      when: r_aap_cr.json.items | length > 0

    - name: Check if PostgreSQL Service exists
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/services/{{ postgres_service_name }}"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_postgres_check
      when: postgres_service_name is defined

    - name: Create PostgreSQL Service if missing
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/services"
        method: POST
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
          Content-Type: "application/json"
        body_format: json
        body:
          apiVersion: v1
          kind: Service
          metadata:
            name: "{{ postgres_service_name }}"
            namespace: aap-rh1
            labels:
              app.kubernetes.io/name: postgres
          spec:
            selector:
              app.kubernetes.io/name: "{{ aap_cr_name }}-postgres-15"
            ports:
              - port: 5432
                targetPort: 5432
                protocol: TCP
        validate_certs: false
        status_code: [200, 201]
      when:
        - postgres_service_name is defined
        - r_postgres_check.status | default(404) == 404

    - name: Show completion message
      ansible.builtin.copy:
        content: |
          [INFO]
          [OK] Module 3 solved

          Fixed:
          - Created PostgreSQL Service: {{ postgres_service_name | default('') }}
          - Gateway pods should now connect to database

          Click Next to validate.
        dest: "{{ job_info_dir }}/validation_failure.out"
      delegate_to: localhost
      when: job_info_dir is defined
```

- [ ] **Step 3: Verify syntax**

```bash
ansible-playbook --syntax-check runtime-automation/main.yml \
  -e module_directory=module-03 -e module_stage=validation
ansible-playbook --syntax-check runtime-automation/main.yml \
  -e module_directory=module-03 -e module_stage=solve
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add runtime-automation/module-03/
git commit -m "feat: add FTL validation + solve for module-03 (Gateway Issues)"
```

---

## Task 5: Module 04 — Gateway Memory Issues

**Scenario:** AAP CR `api.resource_requirements.limits.memory` set too low (128Mi), causing Gateway API pod OOMKills and preventing pods from running.  
**Namespace:** `aap-rh1`

**Files:**
- Create: `runtime-automation/module-04/validation.yml`
- Create: `runtime-automation/module-04/solve.yml`

- [ ] **Step 1: Create validation.yml**

```yaml
# runtime-automation/module-04/validation.yml
---
- name: Validate Module 4 — Gateway Memory Issues
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Get namespace aap-rh1
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_namespace

    - name: Get AnsibleAutomationPlatform CR
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/aap.ansible.com/v1alpha1/namespaces/aap-rh1/ansibleautomationplatforms"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_aap_cr

    - name: Check API memory limits
      ansible.builtin.set_fact:
        api_memory_limit: >-
          {{
            r_aap_cr.json.items[0].spec.api.resource_requirements.limits.memory | default('0Gi')
            if r_aap_cr.json.items | length > 0
            else '0Gi'
          }}

    - name: Get PostgreSQL pods
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/pods?labelSelector=app.kubernetes.io%2Fname%3Dpostgres"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_postgres_pods

    - name: Get Gateway pods
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/pods?labelSelector=app.kubernetes.io%2Fname%3Daap-gateway"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_gateway_pods

    - name: Evaluate checks
      ansible.builtin.set_fact:
        namespace_exists: "{{ r_namespace.status == 200 }}"
        memory_increased: "{{ api_memory_limit is version('1Gi', '>=') }}"
        postgres_running: >-
          {{
            r_postgres_pods.status == 200 and
            r_postgres_pods.json.items | selectattr('status.phase', 'equalto', 'Running') | list | length > 0
          }}
        gateway_running: >-
          {{
            r_gateway_pods.status == 200 and
            r_gateway_pods.json.items | selectattr('status.phase', 'equalto', 'Running') | list | length > 0
          }}

    - name: Set validation output
      ansible.builtin.set_fact:
        validation_output: |
          Module 4: Gateway Memory Issues
          ================================
          {{ '[OK]' if namespace_exists else '[X] ' }} Namespace aap-rh1 exists
          {{ '[OK]' if memory_increased else '[X] ' }} API memory limit ≥ 1Gi (current: {{ api_memory_limit }})
          {{ '[OK]' if postgres_running else '[X] ' }} PostgreSQL pod Running
          {{ '[OK]' if gateway_running else '[X] ' }} Gateway pod Running
        validation_passed: >-
          {{
            namespace_exists and
            memory_increased and
            postgres_running and
            gateway_running
          }}

    - name: Write failure details
      ansible.builtin.copy:
        content: "{{ validation_output }}"
        dest: "{{ job_info_dir }}/validation_failure.out"
      delegate_to: localhost
      when:
        - job_info_dir is defined
        - not validation_passed | bool

    - name: Fail if not passed
      ansible.builtin.fail:
        msg: "Complete the missing tasks and try again."
      when: not validation_passed | bool
```

- [ ] **Step 2: Create solve.yml**

```yaml
# runtime-automation/module-04/solve.yml
---
- name: Solve Module 4 — Gateway Memory Issues
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Get AnsibleAutomationPlatform CR
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/aap.ansible.com/v1alpha1/namespaces/aap-rh1/ansibleautomationplatforms"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
      register: r_aap_cr

    - name: Set CR name
      ansible.builtin.set_fact:
        aap_cr_name: "{{ r_aap_cr.json.items[0].metadata.name }}"
      when: r_aap_cr.json.items | length > 0

    - name: Patch AAP CR to increase API memory
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/aap.ansible.com/v1alpha1/namespaces/aap-rh1/ansibleautomationplatforms/{{ aap_cr_name }}"
        method: PATCH
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
          Content-Type: "application/merge-patch+json"
        body_format: json
        body:
          spec:
            api:
              resource_requirements:
                limits:
                  memory: "1Gi"
                requests:
                  memory: "750Mi"
        validate_certs: false
        status_code: [200, 201]
      when: aap_cr_name is defined

    - name: Get PostgreSQL StatefulSet
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/apps/v1/namespaces/aap-rh1/statefulsets?labelSelector=app.kubernetes.io%2Fname%3Dpostgres"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
      register: r_postgres_sts

    - name: Delete PostgreSQL StatefulSet to force recreation
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/apps/v1/namespaces/aap-rh1/statefulsets/{{ item.metadata.name }}"
        method: DELETE
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 202, 404]
      loop: "{{ r_postgres_sts.json.items }}"
      when: r_postgres_sts.json.items | length > 0

    - name: Show completion message
      ansible.builtin.copy:
        content: |
          [INFO]
          [OK] Module 4 solved

          Fixed:
          - Increased API memory limit to 1Gi / requests 750Mi
          - Deleted PostgreSQL StatefulSet (operator will recreate it)

          Pods should restart automatically. Click Next to validate.
        dest: "{{ job_info_dir }}/validation_failure.out"
      delegate_to: localhost
      when: job_info_dir is defined
```

- [ ] **Step 3: Verify syntax**

```bash
ansible-playbook --syntax-check runtime-automation/main.yml \
  -e module_directory=module-04 -e module_stage=validation
ansible-playbook --syntax-check runtime-automation/main.yml \
  -e module_directory=module-04 -e module_stage=solve
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add runtime-automation/module-04/
git commit -m "feat: add FTL validation + solve for module-04 (Gateway Memory Issues)"
```

---

## Task 6: Module 05 — Automation Hub Sync Issues

**Scenario:** Hub memory limit too low (<8Gi) causing OOMKills, and Hub content PVC too small (<10Gi) causing sync failures.  
**Namespace:** `aap-rh1`

**Files:**
- Create: `runtime-automation/module-05/validation.yml`
- Create: `runtime-automation/module-05/solve.yml`

- [ ] **Step 1: Create validation.yml**

```yaml
# runtime-automation/module-05/validation.yml
---
- name: Validate Module 5 — Automation Hub Sync Issues
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Get namespace aap-rh1
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_namespace

    - name: Get AnsibleAutomationPlatform CR
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/aap.ansible.com/v1alpha1/namespaces/aap-rh1/ansibleautomationplatforms"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_aap_cr

    - name: Check Hub memory limits
      ansible.builtin.set_fact:
        hub_memory_limit: >-
          {{
            r_aap_cr.json.items[0].spec.hub.resource_requirements.limits.memory | default('0Gi')
            if r_aap_cr.json.items | length > 0
            else '0Gi'
          }}

    - name: Get Hub content PVC
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/persistentvolumeclaims?labelSelector=app.kubernetes.io%2Fcomponent%3Dcontent"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_hub_pvc

    - name: Check PVC size
      ansible.builtin.set_fact:
        pvc_size: >-
          {{
            r_hub_pvc.json.items[0].spec.resources.requests.storage | default('0Gi')
            if r_hub_pvc.json.items | length > 0
            else '0Gi'
          }}

    - name: Get Hub pods
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/pods?labelSelector=app.kubernetes.io%2Fname%3Daap-hub"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_hub_pods

    - name: Evaluate checks
      ansible.builtin.set_fact:
        namespace_exists: "{{ r_namespace.status == 200 }}"
        memory_increased: "{{ hub_memory_limit is version('8Gi', '>=') }}"
        pvc_expanded: "{{ pvc_size is version('10Gi', '>=') }}"
        no_oom_pods: >-
          {{
            r_hub_pods.status == 200 and
            (r_hub_pods.json.items | selectattr('status.containerStatuses', 'defined') |
            map(attribute='status.containerStatuses') | flatten |
            selectattr('lastState.terminated.reason', 'defined') |
            selectattr('lastState.terminated.reason', 'equalto', 'OOMKilled') | list | length) == 0
          }}

    - name: Set validation output
      ansible.builtin.set_fact:
        validation_output: |
          Module 5: Automation Hub Sync Issues
          ======================================
          {{ '[OK]' if namespace_exists else '[X] ' }} Namespace aap-rh1 exists
          {{ '[OK]' if memory_increased else '[X] ' }} Hub memory limit ≥ 8Gi (current: {{ hub_memory_limit }})
          {{ '[OK]' if pvc_expanded else '[X] ' }} Hub content PVC ≥ 10Gi (current: {{ pvc_size }})
          {{ '[OK]' if no_oom_pods else '[X] ' }} No OOMKilled Hub pods
        validation_passed: >-
          {{
            namespace_exists and
            memory_increased and
            pvc_expanded and
            no_oom_pods
          }}

    - name: Write failure details
      ansible.builtin.copy:
        content: "{{ validation_output }}"
        dest: "{{ job_info_dir }}/validation_failure.out"
      delegate_to: localhost
      when:
        - job_info_dir is defined
        - not validation_passed | bool

    - name: Fail if not passed
      ansible.builtin.fail:
        msg: "Complete the missing tasks and try again."
      when: not validation_passed | bool
```

- [ ] **Step 2: Create solve.yml**

```yaml
# runtime-automation/module-05/solve.yml
---
- name: Solve Module 5 — Automation Hub Sync Issues
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Get AnsibleAutomationPlatform CR
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/aap.ansible.com/v1alpha1/namespaces/aap-rh1/ansibleautomationplatforms"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
      register: r_aap_cr

    - name: Set CR name
      ansible.builtin.set_fact:
        aap_cr_name: "{{ r_aap_cr.json.items[0].metadata.name }}"
      when: r_aap_cr.json.items | length > 0

    - name: Patch AAP CR to increase Hub memory to 8Gi
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/aap.ansible.com/v1alpha1/namespaces/aap-rh1/ansibleautomationplatforms/{{ aap_cr_name }}"
        method: PATCH
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
          Content-Type: "application/merge-patch+json"
        body_format: json
        body:
          spec:
            hub:
              resource_requirements:
                limits:
                  memory: "8Gi"
                requests:
                  memory: "4Gi"
        validate_certs: false
        status_code: [200, 201]
      when: aap_cr_name is defined

    - name: Get Hub content PVC
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/persistentvolumeclaims?labelSelector=app.kubernetes.io%2Fcomponent%3Dcontent"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
      register: r_hub_pvc

    - name: Expand Hub content PVC to 10Gi
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/persistentvolumeclaims/{{ item.metadata.name }}"
        method: PATCH
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
          Content-Type: "application/merge-patch+json"
        body_format: json
        body:
          spec:
            resources:
              requests:
                storage: "10Gi"
        validate_certs: false
        status_code: [200, 201]
      loop: "{{ r_hub_pvc.json.items }}"
      when: r_hub_pvc.json.items | length > 0

    - name: Show completion message
      ansible.builtin.copy:
        content: |
          [INFO]
          [OK] Module 5 solved

          Fixed:
          - Increased Hub memory to 8Gi limit / 4Gi request
          - Expanded Hub content PVC to 10Gi

          Pods will restart with new limits. Click Next to validate.
        dest: "{{ job_info_dir }}/validation_failure.out"
      delegate_to: localhost
      when: job_info_dir is defined
```

- [ ] **Step 3: Verify syntax**

```bash
ansible-playbook --syntax-check runtime-automation/main.yml \
  -e module_directory=module-05 -e module_stage=validation
ansible-playbook --syntax-check runtime-automation/main.yml \
  -e module_directory=module-05 -e module_stage=solve
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add runtime-automation/module-05/
git commit -m "feat: add FTL validation + solve for module-05 (Hub Sync Issues)"
```

---

## Task 7: Module 06 — Controller Job Issues

**Scenario:** A job template in AAP Controller is configured with a non-existent Execution Environment, causing `automation-job-*` pods to ImagePullBackOff.  
**Namespace:** `aap-rh1`  
**Note:** This module calls the AAP Controller API (`https://control-https/api/controller/v2/`) in addition to the Kubernetes API. The admin password is read from the `{cr-name}-admin-password` secret.

**Files:**
- Create: `runtime-automation/module-06/validation.yml`
- Create: `runtime-automation/module-06/solve.yml`

- [ ] **Step 1: Create validation.yml**

```yaml
# runtime-automation/module-06/validation.yml
---
- name: Validate Module 6 — Controller Job Issues
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Get namespace aap-rh1
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_namespace

    - name: Get Controller task pods
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/pods?labelSelector=app.kubernetes.io%2Fcomponent%3Dtask"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_controller_pods

    - name: Get all pods in aap-rh1
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/pods"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_all_pods

    - name: Filter for automation-job pods with ImagePullBackOff
      ansible.builtin.set_fact:
        failed_job_pods: >-
          {{
            r_all_pods.json.items |
            selectattr('metadata.name', 'match', '^automation-job-.*') |
            selectattr('status.containerStatuses', 'defined') |
            map(attribute='status.containerStatuses') | flatten |
            selectattr('state.waiting.reason', 'defined') |
            selectattr('state.waiting.reason', 'match', '.*ImagePull.*') | list
          }}

    - name: Evaluate checks
      ansible.builtin.set_fact:
        namespace_exists: "{{ r_namespace.status == 200 }}"
        controller_running: >-
          {{
            r_controller_pods.status == 200 and
            r_controller_pods.json.items | selectattr('status.phase', 'equalto', 'Running') | list | length > 0
          }}
        no_image_pull_errors: "{{ (failed_job_pods | default([])) | length == 0 }}"

    - name: Set validation output
      ansible.builtin.set_fact:
        validation_output: |
          Module 6: Controller Job Issues
          ================================
          {{ '[OK]' if namespace_exists else '[X] ' }} Namespace aap-rh1 exists
          {{ '[OK]' if controller_running else '[X] ' }} Controller pods Running
          {{ '[OK]' if no_image_pull_errors else '[X] ' }} No ImagePullBackOff errors on job pods ({{ failed_job_pods | default([]) | length }} found)
        validation_passed: >-
          {{
            namespace_exists and
            controller_running and
            no_image_pull_errors
          }}

    - name: Write failure details
      ansible.builtin.copy:
        content: "{{ validation_output }}"
        dest: "{{ job_info_dir }}/validation_failure.out"
      delegate_to: localhost
      when:
        - job_info_dir is defined
        - not validation_passed | bool

    - name: Fail if not passed
      ansible.builtin.fail:
        msg: "Complete the missing tasks and try again."
      when: not validation_passed | bool
```

- [ ] **Step 2: Create solve.yml**

```yaml
# runtime-automation/module-06/solve.yml
---
- name: Solve Module 6 — Controller Job Issues
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Get AnsibleAutomationPlatform CR
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/apis/aap.ansible.com/v1alpha1/namespaces/aap-rh1/ansibleautomationplatforms"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
      register: r_aap_cr

    - name: Set CR name
      ansible.builtin.set_fact:
        aap_cr_name: "{{ r_aap_cr.json.items[0].metadata.name }}"
      when: r_aap_cr.json.items | length > 0

    - name: Get AAP admin password from secret
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/secrets/{{ aap_cr_name }}-admin-password"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
      register: r_admin_secret
      when: aap_cr_name is defined

    - name: Decode admin password
      ansible.builtin.set_fact:
        aap_admin_password: "{{ r_admin_secret.json.data.password | b64decode }}"
      when: r_admin_secret.json.data.password is defined

    - name: Get all execution environments from Controller
      ansible.builtin.uri:
        url: "https://control-https/api/controller/v2/execution_environments/"
        method: GET
        user: admin
        password: "{{ aap_admin_password }}"
        force_basic_auth: true
        validate_certs: false
      register: r_ees
      when: aap_admin_password is defined

    - name: Build list of valid EE IDs
      ansible.builtin.set_fact:
        valid_ee_ids: "{{ r_ees.json.results | map(attribute='id') | list }}"
      when:
        - r_ees.json.results is defined
        - r_ees.json.results | length > 0

    - name: Get job templates from Controller
      ansible.builtin.uri:
        url: "https://control-https/api/controller/v2/job_templates/"
        method: GET
        user: admin
        password: "{{ aap_admin_password }}"
        force_basic_auth: true
        validate_certs: false
      register: r_job_templates
      when: aap_admin_password is defined

    - name: Find job templates using a non-existent EE
      ansible.builtin.set_fact:
        broken_templates: >-
          {{
            r_job_templates.json.results |
            selectattr('execution_environment', 'defined') |
            selectattr('execution_environment', 'ne', None) |
            rejectattr('execution_environment', 'in', valid_ee_ids) | list
          }}
      when:
        - r_job_templates.json.results is defined
        - valid_ee_ids is defined

    - name: Get default EE (first valid EE)
      ansible.builtin.set_fact:
        default_ee_id: "{{ valid_ee_ids | first }}"
      when:
        - valid_ee_ids is defined
        - valid_ee_ids | length > 0

    - name: Fix broken job templates to use valid EE
      ansible.builtin.uri:
        url: "https://control-https/api/controller/v2/job_templates/{{ item.id }}/"
        method: PATCH
        user: admin
        password: "{{ aap_admin_password }}"
        force_basic_auth: true
        body_format: json
        body: "{{ {'execution_environment': default_ee_id | int} }}"
        validate_certs: false
        status_code: [200, 201]
      loop: "{{ broken_templates | default([]) }}"
      when:
        - broken_templates is defined
        - broken_templates | length > 0
        - default_ee_id is defined

    - name: Show completion message
      ansible.builtin.copy:
        content: |
          [INFO]
          [OK] Module 6 solved

          Fixed:
          - Updated {{ broken_templates | default([]) | length }} job template(s) to use a valid Execution Environment

          Job templates should now launch successfully. Click Next to validate.
        dest: "{{ job_info_dir }}/validation_failure.out"
      delegate_to: localhost
      when: job_info_dir is defined
```

- [ ] **Step 3: Verify syntax**

```bash
ansible-playbook --syntax-check runtime-automation/main.yml \
  -e module_directory=module-06 -e module_stage=validation
ansible-playbook --syntax-check runtime-automation/main.yml \
  -e module_directory=module-06 -e module_stage=solve
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add runtime-automation/module-06/
git commit -m "feat: add FTL validation + solve for module-06 (Controller Job Issues)"
```

---

## Task 8: Module 07 — Lightspeed Issues

**Scenario:** `chatbot-configuration-secret` in `aap-rh1` has wrong `model_name` value, causing Lightspeed API pods to CrashLoopBackOff.  
**Namespace:** `aap-rh1`

**Files:**
- Create: `runtime-automation/module-07/validation.yml`
- Create: `runtime-automation/module-07/solve.yml`

- [ ] **Step 1: Create validation.yml**

```yaml
# runtime-automation/module-07/validation.yml
---
- name: Validate Module 7 — Lightspeed Issues
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Get namespace aap-rh1
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_namespace

    - name: Get chatbot configuration secret
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/secrets/chatbot-configuration-secret"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_chatbot_secret

    - name: Decode model name from secret
      ansible.builtin.set_fact:
        model_name: "{{ r_chatbot_secret.json.data.model_name | b64decode }}"
      when:
        - r_chatbot_secret.status == 200
        - r_chatbot_secret.json.data.model_name is defined

    - name: Get all pods in aap-rh1
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/pods"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_all_pods

    - name: Filter for Lightspeed API pods
      ansible.builtin.set_fact:
        lightspeed_pods: >-
          {{
            r_all_pods.json.items | selectattr('metadata.name', 'match', '.*lightspeed-api-.*') | list
            if r_all_pods.status == 200
            else []
          }}

    - name: Check Running and CrashLoopBackOff counts
      ansible.builtin.set_fact:
        running_count: "{{ lightspeed_pods | selectattr('status.phase', 'equalto', 'Running') | list | length }}"
        crashing_count: >-
          {{
            lightspeed_pods | selectattr('status.containerStatuses', 'defined') |
            map(attribute='status.containerStatuses') | flatten |
            selectattr('state.waiting.reason', 'defined') |
            selectattr('state.waiting.reason', 'equalto', 'CrashLoopBackOff') | list | length
          }}

    - name: Evaluate checks
      ansible.builtin.set_fact:
        namespace_exists: "{{ r_namespace.status == 200 }}"
        correct_model: "{{ model_name | default('') == 'granite-3-2-8b-instruct' }}"
        lightspeed_running: >-
          {{
            (running_count | int) > 0 and
            (crashing_count | int) == 0
          }}

    - name: Set validation output
      ansible.builtin.set_fact:
        validation_output: |
          Module 7: Lightspeed Issues
          ============================
          {{ '[OK]' if namespace_exists else '[X] ' }} Namespace aap-rh1 exists
          {{ '[OK]' if correct_model else '[X] ' }} Chatbot secret model_name = granite-3-2-8b-instruct (current: {{ model_name | default('not set') }})
          {{ '[OK]' if lightspeed_running else '[X] ' }} Lightspeed API pod Running ({{ running_count }} running, {{ crashing_count }} crashing)
        validation_passed: >-
          {{
            namespace_exists and
            correct_model and
            lightspeed_running
          }}

    - name: Write failure details
      ansible.builtin.copy:
        content: "{{ validation_output }}"
        dest: "{{ job_info_dir }}/validation_failure.out"
      delegate_to: localhost
      when:
        - job_info_dir is defined
        - not validation_passed | bool

    - name: Fail if not passed
      ansible.builtin.fail:
        msg: "Complete the missing tasks and try again."
      when: not validation_passed | bool
```

- [ ] **Step 2: Create solve.yml**

```yaml
# runtime-automation/module-07/solve.yml
---
- name: Solve Module 7 — Lightspeed Issues
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Get chatbot configuration secret
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/secrets/chatbot-configuration-secret"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 404]
      register: r_chatbot_secret

    - name: Decode current model name
      ansible.builtin.set_fact:
        current_model: "{{ r_chatbot_secret.json.data.model_name | b64decode }}"
      when:
        - r_chatbot_secret.status == 200
        - r_chatbot_secret.json.data.model_name is defined

    - name: Patch chatbot secret with correct model name
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/secrets/chatbot-configuration-secret"
        method: PATCH
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
          Content-Type: "application/strategic-merge-patch+json"
        body_format: json
        body:
          data:
            model_name: "{{ 'granite-3-2-8b-instruct' | b64encode }}"
        validate_certs: false
        status_code: [200, 201]
      when:
        - r_chatbot_secret.status == 200
        - current_model | default('') != 'granite-3-2-8b-instruct'

    - name: Get Lightspeed operator manager pods
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/pods?labelSelector=app.kubernetes.io%2Fname%3Dansible-lightspeed-operator"
        method: GET
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
      register: r_lightspeed_mgr_pods

    - name: Delete Lightspeed operator pod to force reconciliation
      ansible.builtin.uri:
        url: "https://kubernetes.default.svc/api/v1/namespaces/aap-rh1/pods/{{ item.metadata.name }}"
        method: DELETE
        headers:
          Authorization: "Bearer {{ lookup('file', '/var/run/secrets/kubernetes.io/serviceaccount/token') }}"
        validate_certs: false
        status_code: [200, 202, 404]
      loop: "{{ r_lightspeed_mgr_pods.json.items | default([]) }}"
      when:
        - r_lightspeed_mgr_pods.json.items | default([]) | length > 0
        - current_model | default('') != 'granite-3-2-8b-instruct'

    - name: Show completion message
      ansible.builtin.copy:
        content: |
          [INFO]
          [OK] Module 7 solved

          Fixed:
          - Updated chatbot-configuration-secret model_name to granite-3-2-8b-instruct
          - Restarted Lightspeed operator to reconcile

          Lightspeed API pod should restart with correct model. Click Next to validate.
        dest: "{{ job_info_dir }}/validation_failure.out"
      delegate_to: localhost
      when: job_info_dir is defined
```

- [ ] **Step 3: Verify syntax**

```bash
ansible-playbook --syntax-check runtime-automation/main.yml \
  -e module_directory=module-07 -e module_stage=validation
ansible-playbook --syntax-check runtime-automation/main.yml \
  -e module_directory=module-07 -e module_stage=solve
```

Expected: no errors.

- [ ] **Step 4: Commit**

```bash
git add runtime-automation/module-07/
git commit -m "feat: add FTL validation + solve for module-07 (Lightspeed Issues)"
```

---

## Task 9: Layer 1 — Syntax Gate CI

**Files:**
- Create: `.github/workflows/ftl-syntax-check.yml`

- [ ] **Step 1: Install ansible-lint and yamllint locally and verify they work**

```bash
pip3 install ansible-lint yamllint
yamllint runtime-automation/main.yml
ansible-lint runtime-automation/main.yml
```

Expected: no errors (or understand any existing warnings before encoding them in CI).

- [ ] **Step 2: Create .yamllint config** (controls what yamllint enforces)

Create `runtime-automation/.yamllint`:
```yaml
---
extends: default
rules:
  line-length:
    max: 160
  truthy:
    allowed-values: ['true', 'false']
  comments:
    min-spaces-from-content: 1
```

- [ ] **Step 3: Create the GitHub Actions workflow**

```yaml
# .github/workflows/ftl-syntax-check.yml
---
name: FTL Syntax Gate

on:
  push:
    branches: [main]
    paths:
      - 'runtime-automation/**'
      - 'ui-config.yml'
      - '.github/workflows/ftl-syntax-check.yml'
  pull_request:
    paths:
      - 'runtime-automation/**'
      - 'ui-config.yml'

jobs:
  syntax-check:
    name: Lint and Syntax Check
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install tools
        run: pip install ansible-core ansible-lint yamllint

      - name: yamllint — runtime-automation
        run: yamllint -c runtime-automation/.yamllint runtime-automation/

      - name: yamllint — ui-config.yml
        run: yamllint ui-config.yml

      - name: ansible-lint — all playbooks
        run: |
          ansible-lint runtime-automation/main.yml \
            runtime-automation/module-*/validation.yml \
            runtime-automation/module-*/solve.yml

      - name: Syntax check — main.yml routes
        run: |
          for module in module-02 module-03 module-04 module-05 module-06 module-07; do
            for stage in validation solve; do
              echo "=== syntax check: $module/$stage ==="
              ansible-playbook --syntax-check runtime-automation/main.yml \
                -e module_directory=$module \
                -e module_stage=$stage
            done
          done

      - name: Verify all stubs are executable
        run: |
          for stub in runtime-automation/module-*/*-control.sh; do
            if [ ! -x "$stub" ]; then
              echo "ERROR: $stub is not executable"
              exit 1
            fi
          done
          echo "All stubs are executable"
```

- [ ] **Step 4: Push and verify CI passes**

```bash
git add .github/workflows/ftl-syntax-check.yml runtime-automation/.yamllint
git commit -m "feat: add Layer 1 FTL syntax gate CI"
git push origin main
```

Watch the Actions tab. Expected: all steps green.

---

## Task 10: Layer 2 — Integration Test CI

**Files:**
- Create: `.github/workflows/ftl-integration-test.yml`

**Pre-condition:** A RHDP lab must be provisioned before running this workflow. The workflow takes the lab's Showroom URL and bastion SSH credentials as inputs.

- [ ] **Step 1: Create the workflow**

```yaml
# .github/workflows/ftl-integration-test.yml
---
name: FTL Integration Test

on:
  workflow_dispatch:
    inputs:
      showroom_url:
        description: 'Showroom URL (e.g. https://showroom-abc123-1.apps.ocpvdev01.rhdp.net)'
        required: true
        type: string
      bastion_host:
        description: 'Bastion hostname (e.g. bastion.abc123.example.opentlc.com)'
        required: true
        type: string
      bastion_user:
        description: 'Bastion SSH user'
        required: true
        default: 'devops'
        type: string

jobs:
  integration-test:
    name: E2E Integration Test — All Scenarios
    runs-on: ubuntu-latest
    env:
      SHOWROOM: ${{ inputs.showroom_url }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.BASTION_SSH_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan -H ${{ inputs.bastion_host }} >> ~/.ssh/known_hosts

      - name: Verify Showroom runner is reachable
        run: |
          echo "Checking Showroom runner API..."
          curl -sk $SHOWROOM/runner/api/config | python3 -m json.tool
          echo "Runner API is reachable"

      - name: Check detected stages
        run: |
          stages=$(curl -sk $SHOWROOM/runner/api/config)
          echo "Detected stages: $stages"
          for module in module-02 module-03 module-04 module-05 module-06 module-07; do
            for stage in validation solve; do
              if echo "$stages" | grep -q "${module}/${stage}"; then
                echo "[OK] $module/$stage detected"
              else
                echo "[X]  $module/$stage NOT detected — check .sh stubs"
                exit 1
              fi
            done
          done

      - name: Run integration tests for all scenarios
        run: |
          PASS=0
          FAIL=0
          SUMMARY="| Scenario | Break | Validate(fail) | Solve | Validate(pass) |\n|----------|-------|----------------|-------|----------------|\n"

          for scenario_num in 2 3 4 5 6 7; do
            module="module-0${scenario_num}"
            scenario="scenario-${scenario_num}"
            echo ""
            echo "============================================"
            echo "Testing $module"
            echo "============================================"

            BREAK_OK="✅"
            VFAIL_OK="❌"
            SOLVE_OK="❌"
            VPASS_OK="❌"

            # Step 1: Break the scenario via bastion
            echo "Breaking $scenario..."
            ssh -i ~/.ssh/id_rsa ${{ inputs.bastion_user }}@${{ inputs.bastion_host }} \
              "cd rh1-aap-troubleshooting && echo '$scenario_num' | ansible-playbook site.yml -t break" && \
              BREAK_OK="✅" || BREAK_OK="❌"

            # Step 2: Trigger validation — expect FAIL
            echo "Running validation (expect FAIL)..."
            JOB=$(curl -sk -X POST $SHOWROOM/runner/api/$module/validation)
            JOB_ID=$(echo $JOB | python3 -c "import sys,json; print(json.load(sys.stdin).get('job_id',''))")
            sleep 30
            RESULT=$(curl -sk $SHOWROOM/runner/api/job/$JOB_ID)
            STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin).get('status',''))")
            if [ "$STATUS" = "failed" ]; then
              echo "[OK] Validation correctly detected broken state (FAIL)"
              VFAIL_OK="✅"
            else
              echo "[X]  Validation did not fail as expected (status: $STATUS)"
            fi

            # Step 3: Trigger solve
            echo "Running solve..."
            JOB=$(curl -sk -X POST $SHOWROOM/runner/api/$module/solve)
            JOB_ID=$(echo $JOB | python3 -c "import sys,json; print(json.load(sys.stdin).get('job_id',''))")
            sleep 60
            RESULT=$(curl -sk $SHOWROOM/runner/api/job/$JOB_ID)
            STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin).get('status',''))")
            if [ "$STATUS" = "successful" ]; then
              echo "[OK] Solve completed successfully"
              SOLVE_OK="✅"
            else
              echo "[X]  Solve failed (status: $STATUS)"
            fi

            # Step 4: Wait for pods and re-validate — expect PASS
            echo "Waiting 120s for pods to stabilize..."
            sleep 120
            echo "Running validation (expect PASS)..."
            JOB=$(curl -sk -X POST $SHOWROOM/runner/api/$module/validation)
            JOB_ID=$(echo $JOB | python3 -c "import sys,json; print(json.load(sys.stdin).get('job_id',''))")
            sleep 60
            RESULT=$(curl -sk $SHOWROOM/runner/api/job/$JOB_ID)
            STATUS=$(echo $RESULT | python3 -c "import sys,json; print(json.load(sys.stdin).get('status',''))")
            if [ "$STATUS" = "successful" ]; then
              echo "[OK] Validation passed after solve"
              VPASS_OK="✅"
              PASS=$((PASS + 1))
            else
              echo "[X]  Validation did not pass after solve (status: $STATUS)"
              FAIL=$((FAIL + 1))
            fi

            # Step 5: Reset scenario for next run
            ssh -i ~/.ssh/id_rsa ${{ inputs.bastion_user }}@${{ inputs.bastion_host }} \
              "cd rh1-aap-troubleshooting && echo '$scenario_num' | ansible-playbook site.yml -t fix" || true

            SUMMARY="${SUMMARY}| $module | $BREAK_OK | $VFAIL_OK | $SOLVE_OK | $VPASS_OK |\n"
          done

          echo ""
          echo "============================================"
          echo "INTEGRATION TEST SUMMARY"
          echo "============================================"
          printf "$SUMMARY"
          echo ""
          echo "Passed: $PASS / 6"

          # Write GitHub Actions job summary
          {
            echo "## FTL Integration Test Results"
            echo ""
            printf "$SUMMARY"
            echo ""
            echo "**Passed: $PASS / 6**"
          } >> $GITHUB_STEP_SUMMARY

          if [ $FAIL -gt 0 ]; then
            echo "FAILED: $FAIL scenario(s) did not pass"
            exit 1
          fi
```

- [ ] **Step 2: Add the required secret to the repository**

In GitHub → Settings → Secrets and variables → Actions, add:
- `BASTION_SSH_KEY` — the private SSH key for bastion access (PEM format, the full key contents)

- [ ] **Step 3: Document the pre-Summit runbook**

Add this block as a comment at the top of the workflow file (already included above). The pre-Summit steps are:
1. Order RHDP lab for this content
2. Wait for provisioning to complete
3. Get the Showroom URL and bastion hostname from the provisioning email
4. Go to GitHub Actions → FTL Integration Test → Run workflow
5. Fill in `showroom_url`, `bastion_host`, `bastion_user` inputs
6. Run and verify all 6 scenarios pass

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/ftl-integration-test.yml
git commit -m "feat: add Layer 2 FTL integration test workflow"
```

---

## Self-Review Checklist

After completing all tasks, run:

```bash
# 1. Verify all playbooks parse
for module in module-02 module-03 module-04 module-05 module-06 module-07; do
  for stage in validation solve; do
    ansible-playbook --syntax-check runtime-automation/main.yml \
      -e module_directory=$module -e module_stage=$stage
  done
done

# 2. Verify all stubs exist and are executable
ls -la runtime-automation/module-*/solve-control.sh
ls -la runtime-automation/module-*/validation-control.sh

# 3. Verify no collection modules are used
grep -r "community\.\|kubernetes\.core\.\|redhat\." runtime-automation/ && \
  echo "ERROR: collection module found" || echo "OK: no collection modules"

# 4. Verify every playbook has play wrapper
for f in runtime-automation/module-*/validation.yml runtime-automation/module-*/solve.yml; do
  if ! grep -q "hosts: localhost" "$f"; then
    echo "ERROR: missing hosts: localhost in $f"
  fi
done

# 5. Verify ui-config.yml covers all 6 modules
for i in 02 03 04 05 06 07; do
  grep -q "module-$i" ui-config.yml && echo "OK: module-$i in ui-config" || echo "MISSING: module-$i"
done
```

All checks must pass before declaring implementation complete.
