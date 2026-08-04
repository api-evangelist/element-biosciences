---
name: Triage a failed ElemBio Cloud analysis execution
description: Find failed workflow executions (Bases2Fastq, Cells2Stats), read their logs and error messages, and tie them back to the source run.
api: openapi/element-biosciences-cloud-api-openapi-original.yml
operations:
  - ExecutionService_ListExecutions
  - ExecutionService_GetExecution
  - ExecutionService_GetExecutionLogs
  - ExecutionService_ListExecutionFiles
  - RunService_GetRun
scopes:
  - executions:read
  - executions:logs
  - runs:read
---

# Triage a failed analysis execution

An Execution is a launched bioinformatics workflow — Bases2Fastq demultiplexing,
Cells2Stats cytoprofiling — that consumes one or more Runs and produces outputs.

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

1. **Find the failures.** Call `ExecutionService_ListExecutions`
   (`GET /v1/executions`) with
   `filter=status:EXECUTION_STATUS_FAILED time_created>=7d`, paginating.
   The full status set is `EXECUTION_STATUS_UNSPECIFIED`, `_INITIALIZED`,
   `_UNTRIGGERED`, `_PENDING`, `_RUNNING`, `_COMPLETED`, `_FAILED`, `_CANCELED`.
   Note `_UNTRIGGERED` separately — that is a workflow that never started, which is a
   different problem from one that ran and failed.

2. **Read the execution record.** Call `ExecutionService_GetExecution`
   (`GET /v1/executions/{id}`). The fast signal is `errorMessage` plus
   `statusDescription`. Also capture `workflow`, `platform`, `startedBy`,
   `timeStarted` and `timeCompleted` (duration is often the tell — a failure seconds
   after start is configuration, hours in is data or resources).

3. **Get the logs.** Call `ExecutionService_GetExecutionLogs`
   (`GET /v1/executions/{execution_id}/logs`). This needs the `executions:logs` scope,
   which is separate from `executions:read` — if you get a 403 with
   `INSUFFICIENT_SCOPE` here but reads worked, that is exactly the missing scope to
   report.

4. **Tie it back to the source run.** `execution.inputs[]` carries `RunReference`
   entries (`id`, `name`, `instrumentName`, `kitType`, `runType`). Call
   `RunService_GetRun` on the input run id and check *its* `status` and `metrics` — a
   large share of analysis failures are really upstream run-quality problems (low
   `metrics.sequencing.q30`, low `total_reads`, a run that ended
   `RUN_STATUS_STOPPED` rather than `RUN_STATUS_COMPLETED`).

5. **Check for partial output.** Call `ExecutionService_ListExecutionFiles`
   (`GET /v1/executions/{execution_id}/files`). A failed execution that still wrote
   files usually failed late; an empty output usually means it failed at setup.

6. **Compare against a good run.** If you need a baseline, list executions with
   `filter=status:EXECUTION_STATUS_COMPLETED` for the same workflow and diff the input
   run metrics.

## Reporting

State the failure in this order: what failed, the `errorMessage`, what the logs show,
the source run's status and quality metrics, and whether output was partially written.
Distinguish clearly between an infrastructure failure, a configuration failure, and an
upstream data-quality failure. If you cannot tell from the available evidence, say so —
do not invent a root cause.
