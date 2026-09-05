---
name: Schedule a Wizehire interview
description: >-
  Book a phone, video, or in-person interview against a Wizehire application through the Scout Service.
  This is a WRITE with no undo — the candidate is notified and the API exposes no cancel operation.
api: openapi/wizehire-scout-service-openapi.yml
base_url: https://scout.wizehire.com
operations:
  - get_apply_details_v1_candidates__apply_id__get
  - schedule_interview_v1_interviews_schedule_post
generated: '2026-09-04'
method: generated
source: openapi/wizehire-scout-service-openapi.yml
---

# Schedule a Wizehire interview

## Read this first — this action cannot be taken back

`POST /v1/interviews/schedule` proxies through to the Wizehire ATS and books a real interview. Wizehire's
help centre documents interview invitations and reminders going out to job seekers, so a candidate is
contacted as a result. **The Scout Service exposes no cancel, reschedule, void, or delete operation**, and
Wizehire publishes no cancellation window. Undoing this requires a human in the Wizehire web app.

There is also **no idempotency key** anywhere in this contract, and **no dry-run mode**. If you retry a
request whose response you did not see, you may book a second interview. Send it once.

**Confirm with the user before calling it.** Restate the candidate, the application id, and the interview
type, and wait for an explicit yes.

## Before you start

- **Auth.** `Authorization: Bearer <token>` from an authenticated Wizehire session. Required on both
  operations below.
- **What you need.** An `apply_id` (**integer**) and an `interview_type`.

## Steps

1. **Verify the application first.**
   `GET /v1/candidates/{apply_id}` — operationId `get_apply_details_v1_candidates__apply_id__get`.
   Confirm you have the right person before you book anything. A wrong `apply_id` here means contacting
   the wrong candidate, and you cannot undo it afterwards.

2. **Confirm with the user.** Name the candidate and the interview type. Wait for a yes.

3. **Book it.**
   `POST /v1/interviews/schedule` — operationId `schedule_interview_v1_interviews_schedule_post`.

   Request body (`ScheduleInterviewRequest`, both fields required):

   ```json
   { "apply_id": 12345, "interview_type": "video" }
   ```

   `interview_type` is an enum with exactly three values — **`phone`**, **`video`**, **`in_person`**.
   Anything else returns 422. Note the underscore in `in_person`.

   The request body carries **no date or time field**. Timing is settled by Wizehire's own scheduling
   flow, not by this call. Do not tell the user you booked a specific slot; you did not, and the contract
   gives you no way to.

4. **Report what you actually know.** The 200 response schema is empty in the contract and returns no
   interview identifier, so you cannot confirm which interview record was created. Report that the request
   succeeded, and nothing more.

## Error handling

- **422** — validation. Common causes: `apply_id` sent as a string, or a bad `interview_type`. Read
  `detail[].loc`, fix, and resend **once**.
- **Any other non-200** — the contract declares nothing else, so do not interpret it. **Do not retry a
  non-422 failure**: without idempotency, a retry on an ambiguous failure risks a duplicate booking.
  Surface the status code to the user and let a human resolve it in the Wizehire app.
