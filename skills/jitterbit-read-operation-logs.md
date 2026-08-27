---
name: jitterbit-read-operation-logs
description: Retrieve Jitterbit Integration Studio operation logs and drill into a single execution instance, and retrieve API Manager API logs asynchronously.
api: Jitterbit Harmony Platform APIs
api_url: https://developer.jitterbit.com/harmony-platform-apis/
spec: openapi/jitterbit-harmony-platform-openapi.yml
operations:
  - authenticate
  - getOperationLogs
  - getOperationLogDetails
generated: '2026-08-27'
method: generated
source: openapi/jitterbit-harmony-platform-openapi.yml + https://docs.jitterbit.com/api-manager/api-manager-reference/api-manager-log-service-api/
---

# Read Jitterbit operation and API logs

Two different log surfaces, on two different hosts, with two different
envelopes. Know which one you want.

## A. Integration Studio operation logs (Harmony platform API)

1. **Authenticate** — `authenticate` (`POST /login`); pass the token as the
   `authToken` header.

2. **List operation logs** — `getOperationLogs`
   (`PUT /operation-logs`) with `organizationId` and a `text/plain` request body
   carrying the JSON filter.

   Note the shape: this is a **read modelled as PUT**. Do not assume the
   safe-method semantics an agent would normally apply to a listing call.

3. **Drill into one execution** — `getOperationLogDetails`
   (`GET /operation-logs`) with `operationInstanceGuid`, taken from the listing.

There is no pagination on either operation.

## B. API Manager API logs (Jitterbit Cloud RESTful Service, Beta)

A separate host and a separate token flow. Requires a user with the Admin role.

1. **Get a token** — `PUT {regionHost}/jitterbit-cloud-restful-service/user/login`
   with a JSON body `{"email": "...", "password": "..."}`. Region hosts:
   `https://na-east.jitterbit.com`, `https://emea-west.jitterbit.com`,
   `https://apac.jitterbit.com`.

   If the organization has two-factor authentication on, this first call fails
   with `errorCode: VALIDATE_TFA_LOGIN_EMAIL` and mails a code. Send a second
   request to `/user/login/tfacode` with `email`, `password`, `code` and
   `deviceId`.

2. **Request the log file** — `PUT {regionHost}/jitterbit-cloud-restful-service/api/analytics/debuglogs-async`
   with headers `Content-Type: application/json`, `Accept: application/json` and
   `authToken: <token>`, and a body carrying `timeRangeFrom`, `timeRangeTo` and
   `orgId` (all required), plus optional `ascendSort`, `retrieveLogMessages`,
   `clientTimeZone`, `queryString` and `csvFormat`.

   `queryString` filters on `time`, `apiname`, `envname`, `authprofile`,
   `requestid`, `requestmethod`, `requesturi`, `responsetime`, `sourceip`,
   `sourceapp`, `statuscode` and `message`, with `=`, `<>`, `!=`, `>`, `<`,
   `>=`, `<=`, semicolon-delimited.

3. **Poll and download** — the response returns `{"key": "...", "status": "..."}`.
   Poll `GET {regionHost}/jitterbit-cloud-restful-service/api/analytics/log-async/{orgId}/{key}`.

   Statuses: `RECEIVED` → `PROCESSING` → `COMPLETE`. Also `INVALID` (key not
   found), `ERROR` (generation failed) and `NO_DATA` (filter matched nothing).

## Rules

- **The generated file lives 24 hours; the reference key lives 23.** An
  identical request will not regenerate a file during that window unless the
  previous attempt returned `ERROR` or `NO_DATA`.
- **A failure during `RECEIVED` or `PROCESSING` locks the request until the key
  expires.** The documented workaround is to change any parameter, which makes
  the request distinct. Do not hammer the same body.
- **Rate limits apply.** The cloud API gateway ceiling is 200 requests per
  minute per organization, and exhaustion returns 429 with no `Retry-After`.
  Poll on a backoff, not in a tight loop.
