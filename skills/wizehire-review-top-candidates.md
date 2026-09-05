---
name: Review top candidates for a Wizehire job
description: >-
  Pull the highest-fit candidates for a job from the Wizehire Scout Service, open the full application
  detail for the ones worth a closer look, and hand a shortlist back to the recruiter. Read-only — this
  skill never schedules, messages, or changes anything.
api: openapi/wizehire-scout-service-openapi.yml
base_url: https://scout.wizehire.com
operations:
  - get_top_applies_v1_candidates_top__job_key__get
  - get_apply_details_v1_candidates__apply_id__get
generated: '2026-09-04'
method: generated
source: openapi/wizehire-scout-service-openapi.yml
---

# Review top candidates for a Wizehire job

## Before you start

- **Auth.** Every operation here needs `Authorization: Bearer <token>`. The token comes from an
  authenticated Wizehire session — the Scout Service backs the Scout Chrome extension, and there is no
  self-serve developer console that issues one. If you do not have a token, stop and ask the user for it;
  do not try to obtain one.
- **What you need.** A `job_key` (string). If the user gives you a job title instead, ask for the key —
  the API has no job search or job list operation, so you cannot look one up.
- **The health check is free.** `GET /healthz` (`healthz`) needs no token and returns
  `{status, environment}`. Use it to confirm the service is up before blaming a token.

## Steps

1. **Get the shortlist.**
   `GET /v1/candidates/top/{job_key}` — operationId `get_top_applies_v1_candidates_top__job_key__get`.
   Returns the top candidates for the job by fit score.

   The 200 response schema in the contract is empty (`{}`), so the exact field names are not specified.
   Read what comes back; do not assume a shape. If you need `apply_id` values and cannot find them,
   say so rather than guessing at a key name.

2. **Open the ones worth reading.**
   For each candidate you want detail on:
   `GET /v1/candidates/{apply_id}` — operationId `get_apply_details_v1_candidates__apply_id__get`.
   `apply_id` is an **integer**, not a string. Sending a string returns 422.

   There is no batch operation. One request per application.

3. **Report back.** Summarise for the recruiter. Do not rank, score, or reject candidates on your own
   authority — Wizehire operates under NYC Local Law 144 as an automated employment decision tool and
   publishes a bias audit for its own scoring. Your job here is retrieval and summary; the hiring
   decision is the recruiter's.

## Error handling

- **422** is the only error the contract declares. Body is `{"detail":[{"loc":[...],"msg":"...","type":"..."}]}`.
  Read `detail[].loc` to find the bad field. It is deterministic — **never retry the same payload**.
- **401 / 403 / 404 / 429 / 5xx are all undeclared.** The contract describes none of them, so you cannot
  distinguish an expired token from a deleted application from a throttle by shape alone. Read the raw
  status code, and when it is not 200 or 422, surface it to the user verbatim instead of interpreting it.
- **No rate limits are published.** Wizehire documents none and the API returns no `RateLimit-*` or
  `Retry-After` headers. Be conservative: serialise your requests and back off on any non-200.

## Do not

- Do not call `POST /v1/interviews/schedule` from this skill. That books a real interview and the
  candidate is notified. Use the scheduling skill, and only when the user has asked for it.
- Do not page. There are no pagination parameters; asking for `?limit=` or `?page=` will not do anything.
