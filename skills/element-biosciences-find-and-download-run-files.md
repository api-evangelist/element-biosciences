---
name: Find a completed run and download its files
description: Locate a completed sequencing or multiomics run by quality criteria and retrieve its output files using vended short-lived S3 credentials.
api: openapi/element-biosciences-cloud-api-openapi-original.yml
operations:
  - RunService_ListRuns
  - RunService_GetRun
  - RunService_ListRunFiles
  - RunService_GetRunDownloadCredentials
scopes:
  - runs:read
  - runs:download
---

# Find a completed run and download its files

The most common ElemBio Cloud automation: find the run the scientist means, then get
its data onto disk or into a pipeline.

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

1. **Narrow to candidate runs.** Call `RunService_ListRuns` (`GET /v1/runs`) with a
   filter built from what the user actually specified. Useful keywords:
   - `type:sequencing` or `type:multiomics`
   - `status:RUN_STATUS_COMPLETED`
   - `time_completed>=7d` (or an ISO-8601 date)
   - `instrument.serial_number:<serial>`
   - `name:<substring>` or bare text to search all string fields
   - quality gates: `metrics.sequencing.q30>=90`, `metrics.sequencing.total_reads>=...`

   Example: `filter=type:sequencing status:RUN_STATUS_COMPLETED time_completed>=7d metrics.sequencing.q30>=90`

2. **Disambiguate before downloading.** If more than one run matches, present the
   candidates (`id`, `name`, `timeCompleted`, instrument, key metrics) and let the user
   choose. Do not guess — sequencing runs are large and downloading the wrong one is
   expensive.

3. **Read the full record.** Call `RunService_GetRun` (`GET /v1/runs/{id}`) for the
   chosen run. Confirm `status` is `RUN_STATUS_COMPLETED` before treating the outputs as
   final; `RUN_STATUS_SEQUENCED` means sequencing finished but the run has not fully
   completed.

4. **List the files.** Call `RunService_ListRunFiles` (`GET /v1/runs/{run_id}/files`),
   paginating. Each `File` has `path`, `name`, `size`, `lastModified`, `uri`,
   `downloadUrl` and `storageClass`. Sum `size` and tell the user the transfer volume
   before starting.

5. **Get download credentials.** Call `RunService_GetRunDownloadCredentials`
   (`GET /v1/runs/{run_id}/credentials`). The response is an `S3Credentials` object:
   `region`, `bucket`, `prefix`, `accessKeyId`, `secretAccessKey`, `sessionToken`,
   `expiration`.

   **Treat this response as a secret.** Never print it, log it, or write it into a
   transcript. Use it in-process with an S3 client, and re-request it rather than
   caching past `expiration`.

6. **Transfer.** Pull the objects directly from S3 under `bucket`/`prefix`. If the user
   would rather stream than copy, point them at the CLI instead:
   `elembio mount run <run> <mountpoint>` exposes the same files as a local folder.

## Notes

- `storageClass` may indicate archived objects that need restoring before they can be
  read — surface that rather than failing mid-transfer.
- A 404 with reason `RUN_NOT_FOUND` can mean the run does not exist *or* that the key is
  not scoped to it: list endpoints silently restrict to scoped resources. Say both.
