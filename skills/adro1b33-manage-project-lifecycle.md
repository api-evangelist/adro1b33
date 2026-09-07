---
name: manage-project-lifecycle
description: >-
  Delete, preview and restore AOX projects, jobs and LES runs safely — reading the restore window off
  the resource before destroying anything.
api: AOX Platform API
base_url: https://api.aoxlabs.com
spec: openapi/adro1b33-aox-openapi.yaml
generated: '2026-09-07'
method: generated
source: >-
  Grounded in openapi/adro1b33-aox-openapi.yaml (soft-delete/preview/restore/purge operations and the
  Project.purgeAfter / canRestore / deletion fields). All operationIds verified present in the spec.
operations:
  - projects_list
  - projects_retrieve
  - projects_deletion_preview_retrieve
  - projects_destroy
  - projects_restore_create
  - projects_purge_now_create
  - job_deletion_preview_retrieve
  - job_delete_create
  - job_restore_create
  - job_purge_now_create
  - les_runs_deletion_preview_retrieve
  - les_runs_delete_create
  - les_runs_restore_create
  - les_runs_purge_now_create
  - deletion_executions_retrieve
  - teams_retained_results_retrieve
  - teams_retained_results_download_url_create
---

# Delete and restore AOX resources without losing work

AOX deletion is two-phase, and this is the best-designed part of the contract. Use it.

## The pattern

| Phase | Project | Job | LES run |
|---|---|---|---|
| Preview | `GET /projects/{id}/deletion-preview/` | `GET /job/{id}/deletion-preview/` | `GET /les/runs/{uid}/deletion-preview/` |
| Soft delete | `DELETE /projects/{id}/` | `POST /job/{id}/delete/` | `POST /les/runs/{uid}/delete/` |
| Restore | `POST /projects/{id}/restore/` | `POST /job/{id}/restore/` | `POST /les/runs/{uid}/restore/` |
| Irreversible | `POST /projects/{id}/purge-now/` | `POST /job/{id}/purge-now/` | `POST /les/runs/{uid}/purge-now/` |

Note the inconsistency: a project is soft-deleted with HTTP `DELETE`, while a job and a LES run are
soft-deleted with `POST .../delete/`.

## Always preview first

Call the deletion-preview operation before every delete. It returns the blast radius and the
scheduled purge date. It is a `GET` — it changes nothing.

## Read the window off the resource

`GET /projects/{id}/` returns `deletedAt`, `purgeAfter`, `canRestore` and a `deletion` block.
`purgeAfter` is the moment restore stops working; `canRestore` states in the response whether it is
still available. **ADRO does not publish the retention period anywhere**, so never quote a user a
number of days — read `purgeAfter` and report the actual date.

Track an executed purge with `GET /deletion-executions/{execution_uid}/`
(`deletion_executions_retrieve`), which reports `status`, `phase`, `scheduledFor` and the bytes
removed.

## Never call purge-now on a user's behalf without explicit confirmation

`purge-now` bypasses the restore window entirely. It is the one operation in this workflow with no
reversal. Ask, and quote what the preview said will be destroyed.

## Deleting a running job

`POST /job/{id}/delete/` cancels a `QUEUED` or `RUNNING` job first, then soft-deletes it. The compute
already consumed is not returned.

## After a subscription ends

A team whose subscription has expired is not cut off from its own results:
`GET /teams/{team_uid}/retained-results/` lists approved result artifacts and
`POST /teams/{team_uid}/retained-results/{artifact_uid}/download-url/` issues a download URL. The
retention period for those artifacts is not published either.
