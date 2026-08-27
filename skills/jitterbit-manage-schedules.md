---
name: jitterbit-manage-schedules
description: List, create, update, enable/disable and delete Integration Studio operation schedules through the Jitterbit Harmony platform APIs.
api: Jitterbit Harmony Platform APIs
api_url: https://developer.jitterbit.com/harmony-platform-apis/
spec: openapi/jitterbit-harmony-platform-openapi.yml
operations:
  - authenticate
  - getSchedules
  - createSchedule
  - updateSchedule
  - enableDisableSchedule
  - deleteSchedules
generated: '2026-08-27'
method: generated
source: openapi/jitterbit-harmony-platform-openapi.yml + conventions/jitterbit-conventions.yml
---

# Manage Jitterbit operation schedules

Schedules control when an Integration Studio operation runs. Toggling one is
fully reversible; deleting one is not.

## Steps

1. **Authenticate** — `authenticate` (`POST /login`), then pass
   `authenticationToken` as the `authToken` header on every call below.

2. **List** — `getSchedules` (`GET /schedules`) with `environmentId` and
   `projectGuid`. There is no pagination on this operation; the full set comes
   back in one response.

3. **Create** — `createSchedule` (`POST /schedules`) with a JSON body. The
   schedule is assigned to the specified project. Use `updateSchedule` for an
   existing schedule; the contract separates the two.

4. **Update** — `updateSchedule` (`PUT /schedules`) with a `text/plain` request
   body carrying JSON. That content type is what the published contract
   declares — it is not a typo in this skill.

5. **Enable or disable** — `enableDisableSchedule` (`PUT /schedules-toggle`)
   with `environmentId`, `scheduleGuid` and `enable`. **Prefer this over
   deletion.** It is the only fully reversible write on this surface: flip
   `enable` back and you are where you started.

6. **Delete, only when you mean it** — `deleteSchedules`
   (`DELETE /schedules`) with `environmentId` and `scheduleId`.
   No restore, no trash, no undo is published for schedules. If the schedule may
   be wanted again, read it with `getSchedules` and keep the definition, or just
   disable it instead.

## Gotchas grounded in the contract

- **Two identifier types for one entity.** `deleteSchedules` takes an integer
  `scheduleId`; `enableDisableSchedule` takes a UUID `scheduleGuid`. That
  inconsistency is in Jitterbit's contract. Carry both.
- **Failure is in the body.** Check `success` before trusting a 200. On failure,
  read `error.errorMessage`, `error.errorCode` and `error.errorId`.
- **No idempotency key.** A retried `createSchedule` can create a second
  schedule. List first, then decide.
- **Region-bound.** The token and the host must be the same region: na-east,
  emea-west or apac-southeast.
