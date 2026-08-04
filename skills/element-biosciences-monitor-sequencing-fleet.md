---
name: Monitor the Element Biosciences sequencing fleet
description: Check the health of every registered AVITI instrument and surface the runs currently in flight or recently failed, using the ElemBio Cloud API.
api: openapi/element-biosciences-cloud-api-openapi-original.yml
operations:
  - InstrumentService_ListInstruments
  - InstrumentService_GetInstrument
  - RunService_ListRuns
scopes:
  - instruments:read
  - runs:read
---

# Monitor the sequencing fleet

Use this to answer "how is the lab doing right now" — which instruments are online,
what is sequencing, and what failed recently.

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

1. **Confirm the key can see instruments.** Call `AuthService_GetAuth`
   (`GET /v1/auth`) and check the returned scopes include `instruments:read` and
   `runs:read`. If either is missing, stop and report exactly which permission the key
   needs — re-issuing the key is a console action a human must take.

2. **List the fleet.** Call `InstrumentService_ListInstruments`
   (`GET /v1/instruments`), following `nextPageToken`. Each instrument carries
   `serialNumber`, `name`, `model`, `status`, `softwareVersion` and
   `timeLastConnected`.

3. **Flag instruments that need attention.** `status` is one of
   `INSTRUMENT_STATUS_ACTIVE`, `INSTRUMENT_STATUS_INACTIVE`,
   `INSTRUMENT_STATUS_DECOMMISSIONED` (or `_UNSPECIFIED`). Report anything not
   `ACTIVE`, and any `ACTIVE` instrument whose `timeLastConnected` is stale relative to
   now — a connected-but-silent instrument is the interesting case.

4. **Pull the runs in flight.** Call `RunService_ListRuns` (`GET /v1/runs`) with
   `filter=status:RUN_STATUS_SEQUENCING,RUN_STATUS_RUNNING`. Join each run to its
   instrument through `run.instrument.serialNumber`.

5. **Pull recent failures.** Call `RunService_ListRuns` again with
   `filter=status:RUN_STATUS_FAILED,RUN_STATUS_STOPPED time_updated>=7d`.

6. **Drill in on one instrument when asked.** Call `InstrumentService_GetInstrument`
   (`GET /v1/instruments/{serial_number}`) — note the path parameter is the serial
   number, not an opaque id. Then scope runs to it with
   `filter=instrument.serial_number:<serial>`.

## Reporting

Give a fleet table (serial, name, model, status, last connected), then in-flight runs,
then recent failures. Do not speculate about *why* a run failed from this data alone —
that requires the execution logs; use the triage skill for that.
