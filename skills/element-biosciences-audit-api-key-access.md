---
name: Audit what an ElemBio Cloud API key can reach
description: Verify an API key's tenant and scopes, then enumerate the runs, executions, instruments and storage connections it is actually able to see.
api: openapi/element-biosciences-cloud-api-openapi-original.yml
operations:
  - AuthService_GetAuth
  - RunService_ListRuns
  - ExecutionService_ListExecutions
  - InstrumentService_ListInstruments
  - StorageConnectionService_ListStorageConnections
  - StorageConnectionService_GetStorageConnection
scopes: []
---

# Audit what an API key can reach

Run this before wiring a key into a LIMS, dashboard, or agent, and whenever someone
asks "what does this key have access to?".

## Rules that apply to every call

- Base URL is `https://cloud-api.usw2.elembio.io`. Every path is under `/v1`.
- Send the API key on every request: `x-api-key: <key>`. There is no OAuth flow.
- Errors return `{code, message, details[]}`. Branch on `details[0].reason`, not on the
  message text. `INSUFFICIENT_SCOPE` means the key is valid but under-permissioned —
  do not retry it, report which scope is needed. `INTERNAL_ERROR` is retryable with
  backoff. Capture the `X-Request-ID` response header on any failure.
- List endpoints are cursor-paginated: pass `pageSize` (max 1000) and follow
  `nextPageToken` until it comes back empty. Never assume one page is the whole set.
- The `filter` parameter uses `keyword:value` terms ANDed by spaces, `,` for
  alternatives, and `>= <= > < !=` for numeric and date fields. Dates accept ISO-8601
  or relative offsets like `7d` / `1mo`. Filter keywords are snake_case even though
  response fields are lowerCamelCase.
- Every operation in this API is a GET. Nothing here mutates state, so all steps are
  safe to retry.

## Steps

1. **Introspect the key.** Call `AuthService_GetAuth` (`GET /v1/auth`). This is the only
   operation that needs no scope. It returns the `tenantId` the key belongs to and the
   scopes it carries. Record both verbatim.

2. **Map scopes to capability.** The documented permissions are:

   | Permission | Grants |
   |---|---|
   | `Runs:Read` | List and get run metadata |
   | `Runs:Download` | Browse and download run files |
   | `Executions:Read` | List and get execution metadata |
   | `Executions:Download` | Browse and download execution files |
   | `Executions:Logs` | View execution logs |
   | `Instruments:Read` | List and get instrument metadata |
   | `Storage:Read` | List and get storage connection metadata |
   | `Storage:Download` | Browse and download storage files |

   The unsuffixed forms (`Runs`, `Executions`, `Instruments`, `Storage`) are
   unrestricted access to that resource family. Flag any of those four as
   over-permissioned unless the user justifies them — Element's own guidance is to
   grant the minimum required.

3. **Enumerate the visible surface.** For each scope the key holds, call the matching
   list operation and count what comes back, paginating fully:
   `RunService_ListRuns`, `ExecutionService_ListExecutions`,
   `InstrumentService_ListInstruments`,
   `StorageConnectionService_ListStorageConnections`.

   List endpoints transparently restrict results to what the key is scoped to, so the
   counts *are* the blast radius.

4. **Check storage reach specifically.** `Storage:Download` can be narrowed to a single
   connection (`storage:download:{connection_id}`) or a path prefix. Call
   `StorageConnectionService_GetStorageConnection` on each connection returned and record
   its `uri` and `type` — this is the customer-owned bucket the key can pull from, and
   the most consequential thing on the list.

5. **Probe the boundary, carefully.** To confirm a scope is genuinely absent, call one
   operation it would gate and confirm a 403 with `INSUFFICIENT_SCOPE`. Only use read
   and list operations for this. Never call the `*/credentials` operations during an
   audit — they vend live S3 session credentials, and issuing real credentials is not
   something an audit should do as a side effect.

## Reporting

Report the tenant, the exact scope list, the count of each resource the key can see, the
storage connections and their URIs, and any scope you consider broader than the stated
use requires. Note the key's expiry if the user can supply it from the console — keys
expire between 1 and 365 days, defaulting to 30, and a key nearing expiry inside an
automated pipeline is an outage waiting to happen.
