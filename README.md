# GoQueue ⚙️

A distributed task processing system in Go with Redis Streams, concurrent workers, bounded retries, and crash recovery.

## Overview

GoQueue separates task submission from execution. The API accepts jobs, Redis coordinates delivery, and independent workers process tasks asynchronously.

The design focuses on explicit job state, controlled concurrency, and recovery from failures.

## Architecture

GoQueue consists of three Go services backed by Redis:

| Component | Responsibility |
|-----------|----------------|
| **API** | Validate requests, enqueue jobs, and serve job status and results |
| **Workers** | Execute tasks with bounded concurrency and execution deadlines |
| **Scheduler** | Move scheduled retries back into the ready queue |
| **Redis** | Store stream messages, job state, retry schedules, and results |

Workers share a Redis Streams consumer group to distribute work across processes.

## Processing Model

- **Asynchronous submission** — return a job ID after persisting the job and publishing its queue entry.
- **Bounded concurrency** — limit active executions per worker process.
- **Explicit acknowledgment** — acknowledge deliveries after recording their outcome.
- **Retry backoff** — schedule temporary failures with exponential backoff and jitter.
- **Crash recovery** — reclaim abandoned pending deliveries.
- **Ownership checks** — reject state updates from stale worker attempts.
- **Submission deduplication** — associate repeated requests with the same idempotency key.
- **Dead-letter storage** — retain permanently failed jobs for inspection.

## Job Lifecycle

A job moves through these states:

| State | Meaning |
|-------|---------|
| `queued` | Waiting for execution |
| `running` | An execution attempt is active |
| `retry_wait` | Waiting for a scheduled retry |
| `succeeded` | Execution completed and the result is available |
| `dead` | A permanent failure occurred or the attempt limit was reached |

## API

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/jobs` | Submit a task |
| GET | `/jobs/{id}` | Retrieve job state and attempt details |
| GET | `/jobs/{id}/result` | Retrieve the completed result |
| GET | `/healthz` | Process liveness |
| GET | `/readyz` | Redis connectivity |

The initial task type is `generate_report`, which converts structured sales data into a CSV report.

## Reliability Model

The system is designed for retryable delivery with potentially repeated execution. Attempts are bounded, and handlers must tolerate repetition.

Redis state transitions use atomic operations. Execution ownership checks prevent stale workers from overwriting newer attempts, while external side effects require their own idempotency mechanism.

Durability depends on Redis persistence configuration. The architecture targets a single Redis instance; high availability is outside its current scope.

## Stack

**Go · Redis Streams · Lua · Docker Compose**