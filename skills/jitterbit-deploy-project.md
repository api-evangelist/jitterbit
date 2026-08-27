---
name: jitterbit-deploy-project
description: Deploy an Integration Studio project into a Harmony environment using the Jitterbit Harmony platform APIs, taking a recoverable export first.
api: Jitterbit Harmony Platform APIs
api_url: https://developer.jitterbit.com/harmony-platform-apis/
spec: openapi/jitterbit-harmony-platform-openapi.yml
operations:
  - authenticate
  - getProject
  - exportProject
  - deployProject
generated: '2026-08-27'
method: generated
source: openapi/jitterbit-harmony-platform-openapi.yml + conventions/jitterbit-conventions.yml
---

# Deploy a Jitterbit Integration Studio project

Deploying overwrites the deployed state of a project. Jitterbit publishes no
rollback-to-previous-deployment operation, so the export in step 2 is the only
recovery path you get. Do not skip it.

## Before you start

You need three identifiers that the API cannot give you. All three come out of
the Harmony web console:

- `organizationId` — top right of the Harmony portal header, next to the org name.
- `environmentId` — Management Console → Environments.
- `projectGuid` — the browser address bar while the project is open in Integration Studio, e.g. `9a87b6c5-a1b2-34c5-6d7e-123456a7b89c`.

Pick the base URL for your organization's region:
`https://harmony-api.na-east.jitterbit.com/{endpoint}`,
`https://harmony-api.emea-west.jitterbit.com/{endpoint}` or
`https://harmony-api.apac-southeast.jitterbit.com/{endpoint}`. The `endpoint`
variable in the published contract defaults to `dev`; use `api` for production.

## Steps

1. **Authenticate** — `authenticate` (`POST /login`) with `username` and
   `password` as query parameters. Read `authenticationToken` out of the
   response body. Every later call carries it as the `authToken` request header.
   The token is valid for 14400 seconds (4 hours). There is no refresh
   operation — re-run this step when it expires.

   This will not work for an organization on Harmony SSO. The operation's own
   description says the credentials "must be associated with Harmony account
   credentials and not an organization using Harmony SSO".

2. **Export first** — `exportProject` (`GET /project-import-export`) with
   `projectGuid` and the four include flags (`includeEmails`,
   `includeSchedules`, `includeCredentials`, `includeProjectVariables`). Keep
   the returned JSON. If the deploy goes wrong, `importProject` on this file is
   how you get back.

   `includeCredentials` may be disabled by the organization's *Allow credentials
   to be exported* policy. If it is, record that the restore will need
   credentials re-entered by hand.

3. **Confirm what you are about to change** — `getProject` (`GET /project`) with
   `organizationId`, `environmentId` and `projectGuid`. Optional `AcceptVersion`
   header. Verify you are pointed at the project you think you are.

4. **Deploy** — `deployProject` (`PUT /project`) with a JSON request body
   carrying the `projectGuid`.

5. **Check the result properly.** A 200 does not mean success. The Harmony
   envelope reports application failures in the body:

   ```json
   {"success": false, "uri": "...", "error": {"errorMessage": "...", "errorCode": "...", "errorId": "..."}}
   ```

   Read `success` first. On `false`, keep `error.errorId` — that is the handle
   Jitterbit support asks for. Ignore any `guid-*` field; the documentation says
   so explicitly.

## Rules

- **No idempotency key exists.** If a deploy call times out, you cannot safely
  replay it and be sure it fires once. Re-read the project state with
  `getProject` before retrying.
- **No dry run exists.** There is no validate-only or preview parameter on
  `deployProject`.
- **No rate-limit headers exist.** You will find out you are over the line when
  a `429` arrives, with no `Retry-After`. Back off exponentially with jitter.
- **The contract declares only 200 responses.** Handle 4xx and 5xx defensively;
  nothing in the spec describes them.
