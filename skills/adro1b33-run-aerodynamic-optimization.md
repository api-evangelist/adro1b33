---
name: run-aerodynamic-optimization
description: >-
  Take a vehicle geometry from upload through a CFD optimization job on ADRO's AOX platform and
  retrieve the optimized result, checking the credit cost before committing to the run.
api: AOX Platform API
base_url: https://api.aoxlabs.com
spec: openapi/adro1b33-aox-openapi.yaml
generated: '2026-09-07'
method: generated
source: >-
  Grounded in openapi/adro1b33-aox-openapi.yaml. Every operationId below was verified present in the
  spec. ADRO publishes no tutorial or getting-started guide, so the ORDER of steps is inferred from
  the contract's own structure (init/complete upload handshake, estimate-before-submit, poll-monitor)
  and is not a provider-published workflow.
operations:
  - projects_create
  - projects_source_uploads_init_create
  - projects_source_uploads_complete_create
  - projects_source_uploads_abort_create
  - projects_register_stl_create
  - projects_mesh_predict_retrieve
  - projects_patches_ref_create
  - credits_estimate_retrieve
  - credits_wallet_retrieve
  - job_unified_create
  - job_monitor_retrieve
  - job_retrieve
  - job_cancel_create
  - job_valid_dsns_retrieve
  - job_result_mesh_retrieve
  - job_optimized_stl_retrieve
  - job_export_step_retrieve
  - credits_usage_jobs_retrieve_2
---

# Run an aerodynamic optimization on AOX

The AOX API is unversioned, session-authenticated and **trailing-slash strict** — a request to
`/projects` without the final slash will not match. Bodies are camelCase; query and path parameters
are snake_case.

## 0. Before you start

- Establish a session (`POST /accounts/login/`). The contract declares no securityScheme; see
  `authentication/adro1b33-authentication.yml`.
- Confirm the active team (`GET /accounts/teams/`, `POST /accounts/team-switch/`). Every project, job
  and credit charge is scoped to a team.
- **There is no idempotency mechanism.** No operation accepts an `Idempotency-Key`. If a POST times
  out, do not blind-retry it — re-read the collection first (`GET /projects/`, `GET /projects/{id}/jobs/`)
  and check whether the resource already exists. A blind retry creates a second project or a second
  paid job.

## 1. Create the project

`POST /projects/` → `projects_create`. Keep both the numeric `id` and the `uid` from the response:
the API addresses some resources by one and some by the other.

## 2. Upload the geometry

A three-call handshake, all keyed on the numeric `project_id`:

1. `POST /projects/{project_id}/source-uploads/init/` → `projects_source_uploads_init_create`
2. Upload the bytes to the location the init response returns.
3. `POST /projects/{project_id}/source-uploads/complete/` → `projects_source_uploads_complete_create`

If any part fails, call `POST /projects/{project_id}/source-uploads/abort/`
(`projects_source_uploads_abort_create`) rather than leaving a half-finished upload. For an STL
already in storage, `POST /projects/{project_id}/register-stl/` (`projects_register_stl_create`) is
the direct path — and it is the **only** operation in the whole contract that documents its 400 and
404 responses, so treat those as the shape you handle everywhere else too.

Non-STL geometry goes through `POST /stl/convert/` (`stl_convert_create`) or the async
`POST /stl/convert-async/`; a broken mesh goes through `POST /stl/repair/` (`stl_repair_create`).

## 3. Set up the optimization

- `GET /projects/{id}/mesh-predict/` (`projects_mesh_predict_retrieve`) predicts the mesh before you
  pay for one.
- `POST /projects/{id}/geometry-rotation/` (`projects_geometry_rotation_create`) orients the model.
- `POST /projects/{id}/patches/ref/` (`projects_patches_ref_create`) attaches the deformation regions
  the optimizer is allowed to move. Merge related patches with `POST /patches/merge/`
  (`patches_merge_create`); this is reversible with `POST /patches/{id}/unmerge/`.

## 4. Price the run before you commit

`GET /credits/estimate/` → `credits_estimate_retrieve`, and `GET /credits/wallet/` →
`credits_wallet_retrieve` for the balance. **Do this every time.** Runs are metered in credits against
a per-team wallet, the plan caps which solvers you may use at all, and there is no free retry: a job
you cancel mid-run has already consumed compute.

## 5. Submit

`POST /job/unified/` → `job_unified_create`. Keep the returned job `id` **and** `uid`.

## 6. Poll — do not wait on a webhook, there is none

`GET /job/{id}/monitor/` → `job_monitor_retrieve`. The contract's own description tells the front end
to poll this every 5–10 seconds. There are no webhooks, no callbacks and no AsyncAPI document; in-app
notifications (`GET /job-notifications/`) are pull, not push.

**A 200 does not mean success.** Read `status`, `stage` and `progress`, and check `errorSummary` and
`errorDetail` on `GET /job/{id}/` (`job_retrieve`) — failure detail lives in the resource, not in an
HTTP error. `GET /job/{id}/execution-logs/` and `GET /job/{id}/history/` carry the detail.

To stop a run: `POST /job/{id}/cancel/` → `job_cancel_create`. ADRO's refund policy
(https://aoxlabs.com/refund) enumerates the cancellation points — before execution starts, during
Meshing or Lattice Setup, during Simulating, during Optimizing, during PostProcessing, and after
successful completion — but does **not** publish the credit outcome at each point in machine-readable
form. Do not promise a user a refund you cannot verify.

## 7. Collect results

- `GET /job/{id}/valid-dsns/` → `job_valid_dsns_retrieve` (which result sets exist)
- `GET /job/{id}/result-mesh/` → `job_result_mesh_retrieve` (presigned mesh URL per DSN)
- `GET /job/{id}/streamlines/`, `GET /job/{id}/flowfield/` for the flow visualization
- `GET /job/{id}/optimized-stl/` → `job_optimized_stl_retrieve` (the optimized geometry)
- `GET /job/{id}/export-step/` → `job_export_step_retrieve` (STEP for CAD hand-back)
- `GET /credits/usage/jobs/{job_uid}/` → `credits_usage_jobs_retrieve_2` (what it actually cost)

## Errors

Not RFC 9457. Two envelopes: `{"detail": "..."}` for 401/403/404/405, and `{"field": ["..."]}` for
400 validation. **Messages come back in Korean regardless of `Accept-Language`** — match on status
code and field name, never on message text. A 500 returns HTML, not JSON, so guard your parser. Full
catalog: `errors/adro1b33-problem-types.yml`.
