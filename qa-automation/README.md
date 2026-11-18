# Advanced App Platform Demo — QA Automation

Automated testing playbooks for the OpenShift Advanced App Platform demo
(`ocp4-adv-app-platform-demo`). Modeled on the
[konflux-tssc-lab qa-automation](https://github.com/rhpds/konflux-tssc-lab/tree/main/qa-automation)
pattern.

## Philosophy: test the outcome, not the clicks

The Showroom content is a **presenter demo script** (browser-driven: OpenShift
console, DevSpaces IDE, Developer Hub, Argo CD, GitLab UI, Roo Code AI). These
playbooks do **not** replay those UI actions. Instead, each module test drives
the **equivalent API / git / `oc` action** and asserts the resulting cluster
state — the same fact the presenter confirms visually.

Pure talk-track steps (MTA story, business-value narration) and visual-only
callouts have no deterministic machine outcome and are intentionally skipped.
Non-deterministic LLM steps (Roo Code fix, LLM classification text) are reduced
to invariants (pipeline passes; inbox has zero "Review Required").

## Structure

```
qa-automation/
├── healthcheck.yml       # Read-only environment readiness (runs at provision time)
├── e2e.yml               # Orchestrator — imports all module playbooks in order
├── e2e-module-01.yml     # Developer experience (Parasol dev app + DevSpaces)
├── e2e-module-02.yml     # CI/CD pipeline — push → Tekton → Argo → live endpoint
├── e2e-module-03.yml     # Platform operations (Mesh, Kiali, HPA, External Secrets)
├── e2e-module-04.yml     # Developer Hub (RHDH + catalog + software template)
├── e2e-module-05.yml     # Secure development (secured pipeline, ACS, RHTAS)
├── e2e-module-06.yml     # Trusted supply chain (TPA SBOM API)
├── e2e-module-07.yml     # AI applications (LLM email routing outcome)
└── README.md             # This file
```

## Healthcheck vs E2E

| Aspect | Healthcheck | E2E |
|--------|-------------|-----|
| **When** | At instance order/provision time | After deploy / on-demand |
| **Purpose** | Is the instance ready for a group? | Do the demo workflows actually work? |
| **Scope** | Namespaces, Argo apps Healthy/Synced, routes, GitLab repo, DevSpaces, Vault | Per-module outcomes (pipelines, deploys, endpoints, APIs) |
| **Mutates?** | No | Modules 02/07 push commits when `qa_run_mutating=true` |
| **Duration** | ~1–2 min | ~20–40 min (full suite; pipeline builds dominate) |

## Topology note

This lab is **per-group** (one shared pre-provisioned instance per group of 5)
with **fixed namespaces** (`parasol-insurance-{build,dev,prod}`, plus
`-secured-` variants) — not multi-tenant. So tests target cluster-wide demo
state rather than per-user resources.

## Authentication (dual mode)

Every playbook supports both entry points used by the Publishing House / Showroom:

**CLI / admin mode:**
```bash
ansible-playbook qa-automation/e2e.yml \
  -e "ocp_api_url=https://api.cluster-xxx:6443" \
  -e "ocp_admin_password=xxx" \
  -e "common_password=openshift"
```

**Showroom runner mode** (auto): reads `openshift_api_url` from `user_data.yml`
and authenticates with the `runtime-automation` ServiceAccount token in the
showroom namespace. Override the location with `-e showroom_namespace=...` and
`-e showroom_sa_secret=...` if different.

## Running

```bash
# Readiness check (safe, read-only)
ansible-playbook qa-automation/healthcheck.yml -e ocp_api_url=... -e ocp_admin_password=...

# Full outcome suite, including the mutating pipeline/AI drives
ansible-playbook qa-automation/e2e.yml \
  -e ocp_api_url=... -e ocp_admin_password=... \
  -e common_password=openshift -e qa_run_mutating=true

# Single module
ansible-playbook qa-automation/e2e-module-02.yml -e ocp_api_url=... -e ocp_admin_password=...
```

### Key variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `ocp_api_url` / `openshift_api_url` | — | Cluster API URL (CLI / runner) |
| `ocp_admin_password` | — | Admin password (CLI mode); omit for runner mode |
| `common_password` | `openshift` | Shared demo password (GitLab root OAuth, etc.) |
| `qa_run_mutating` | `false` | Allow tests to push commits / merge MRs / apply CRs |
| `litellm_api_base_url` | `""` | External LiteLLM endpoint for the Module 7 prereq check |
| `showroom_namespace` | `showroom` | Namespace of the runner SA token (runner mode) |

## How each module maps demo action → asserted outcome

| Module | Demo action (UI) | Automated outcome |
|--------|------------------|-------------------|
| 01 | Open app in Topology, open DevSpaces | Parasol dev Deployments available; CheCluster `Active`; dashboard answers |
| 02 | Commit in IDE → pipeline → Roo fix → Argo | Commit clean endpoint via GitLab API → PipelineRun `Succeeded` → Argo dev `Synced/Healthy` → `GET /api/claims/stats` 200 |
| 03 | Kiali graph, HPA scaling, Vault secrets | Istio `Ready`; Kiali route answers; HPA exists; ExternalSecrets `Ready` |
| 04 | Developer Hub + self-service template | RHDH Deployment up; catalog API answers; "secured development" template present |
| 05 | Secure pipeline, ACS, SBOM, TAS, Conforma | Secured pipeline + namespace exist; latest secured run `Succeeded`; ACS Central answers; RHTAS `Securesign` present |
| 06 | Browse SBOMs / CVEs in TPA | TPA UI answers; `/api/v2/sbom` responds |
| 07 | RHDH → DevSpaces → git overlay → push | LLM ExternalSecret `Ready`; prepared branch exists; (mutating) merge `llm-classification`→main → Argo sync → inbox has no "Review Required" |

## Error-handling pattern

Lighter modules (01, 03, 04, 05, 06) use the **error-collection** pattern:
run every check with `ignore_errors`, append messages to `e2e_failures`, and
fail once at the end with a consolidated report. The core pipeline module (02)
uses `until`-retries and fails fast, since later assertions depend on earlier
steps succeeding.

## Known assumptions / things to adjust

Some resource names/paths are best-effort and parameterized at the top of each
playbook — adjust if the environment differs:

- Route names (e.g. `rhdh/backstage-developer-hub`, `stackrox/central`,
  `istio-system/kiali`).
- Parasol app REST paths (`/api/claims/stats`, `/api/emails`) depend on the
  `parasol-insurance` application.
- The prepared `llm-classification` branch must exist in GitLab for Module 7's
  mutating path.
- API versions (`sailoperator.io/v1`, `rhtas.redhat.com/v1alpha1`,
  `external-secrets.io/v1beta1`, `org.eclipse.che/v2`) match the demo's operator
  set; bump if operators are upgraded.
