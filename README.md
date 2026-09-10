# Preset Darkroom

Preset Darkroom is a trial-rendering platform for photographers selling RAW-editing presets. It lets prospective buyers upload a photo, apply a creator's preset, and preview the result a limited number of times before deciding whether to purchase.

## The problem

Photographers can create distinctive editing presets, but buyers often have to purchase without knowing how a preset will behave on their own images. Gallery examples cannot account for a buyer's lighting, camera, skin tones, or composition.

Preset Darkroom lets buyers try a preset on their own RAW photo a limited number of times before purchase. Sellers gain a safer conversion path, while the platform preserves trial limits and watermarks previews.

## Architecture

The engine separates the public API from compute-heavy RAW rendering with an asynchronous job workflow. The Gateway accepts a preview request, records a job, and appends it to Redis Streams for a private Worker to render. It returns promptly with a job identifier, so a slow RAW render never holds an HTTP request open.

```text
Browser / Frontend
        |
        | HTTPS (RESTful) — upload image and create / query preview job
        v
Gateway (Core API) ------------------> Job database
  - REST API and request validation       - job status and ownership
  - Redis-backed trial rate limiting      - output reference and errors
  - preview-job creation and status API
        |
        | Redis Streams — enqueue render job
        v
Redis Streams
        |
        | Consumer group — worker consumes job
        v
Worker(s)
  - RAW image pipeline
  - preset / 3D LUT application
  - watermarking and preview encoding
        |
        | HTTPS read/write
        v
Private object/shared storage
  uploads/                 outputs/
```

The frontend uses HTTPS REST endpoints to create and retrieve preview jobs. `POST /api/v1/previews` returns `202 Accepted` with a job ID and status URL; `GET /api/v1/previews/{job_id}` returns `queued`, `processing`, `complete`, or `failed`. When complete, the Gateway verifies that the authenticated buyer owns the job and returns a short-lived signed HTTPS URL for the preview object. Browsers never connect to Workers, Redis, or the storage mount directly.

## Service roles

| Service | Role | How it communicates |
| --- | --- | --- |
| **Gateway** | Public Core API. It validates input, applies Redis-backed trial limits, creates preview jobs, and returns job status. | Receives **HTTPS REST** traffic; writes job records; uses `XADD` to append rendering messages; creates signed preview URLs after ownership checks. |
| **Job database** | Source of truth for preview jobs: owner, input, preset, status, output reference, attempts, and errors. | Written by Gateway and Workers through a database connection over TLS; read by Gateway status endpoints. |
| **Redis Streams** | Buffers rendering work and enables asynchronous retry. | Gateway publishes to `preview:render` with `XADD`; Workers in the `preview-workers` consumer group use `XREADGROUP`, `XACK`, and `XAUTOCLAIM`. |
| **Worker** | Private compute service that processes RAW files, applies LUTs, watermarks previews, and saves output. | Consumes Redis Stream messages, reads/writes storage over **HTTPS**, and updates job state in the database. |

Workers acknowledge an entry only after the job result is durably written. Stale pending jobs are reclaimed with `XAUTOCLAIM`; jobs that exhaust retry attempts are moved to `preview:dead-letter`. Storage objects remain private: the Worker writes a non-guessable path such as `outputs/{job_id}/preview.webp`, while the Gateway returns an ownership-checked signed URL. For local development only, the Gateway may serve mounted outputs through a development route.

## Project layout

```text
raw-preset-engine/
├── gateway/                         # Public Core API gateway
│   ├── app/
│   │   ├── core/                    # Configuration, Redis, database, and storage clients
│   │   ├── middleware/              # Tracing and rate limiting
│   │   ├── controllers/             # Job creation and status orchestration
│   │   ├── routers/                 # Versioned REST routes
│   │   └── main.py                  # Gateway application entry point
│   ├── Dockerfile
│   └── requirements.txt
├── worker/                          # Private asynchronous rendering service
│   ├── engine/                      # RAW, LUT, and watermark pipeline
│   ├── presets/                     # 3D LUT preset files
│   ├── job_consumer.py              # Redis Streams consumer entry point
│   ├── Dockerfile
│   └── requirements.txt
├── migrations/                      # Preview-job database schema
├── shared_storage/                  # Local object-storage mock
│   ├── uploads/
│   └── outputs/
└── docker-compose.yml               # Gateway, worker, Redis, database, and storage
```

## Architecture Decisions and Rationale

| Decision | Rationale |
| --- | --- |
| **HTTPS REST for the public API** | Browser-native, versioned endpoints for submitting a preview job and retrieving its state. |
| **Redis Streams for rendering jobs** | `XADD`, consumer groups, `XACK`, and `XAUTOCLAIM` provide a lightweight, at-least-once job workflow without holding an HTTP request open. |
| **No Thrift on the render path** | Redis Streams decouples the Gateway from Workers, enabling independent scaling, retries, and recovery from Worker failures. |
| **Private object storage with signed URLs** | The Gateway enforces ownership then returns a short-lived HTTPS URL, avoiding public previews and large image proxying. |
| **Python Worker stack** | Use `rawpy` (LibRaw) for 16-bit RAW decoding, `numpy` for vectorized processing, `colour-science` for `.cube` parsing and tetrahedral interpolation, `Pillow` for watermarking/WebP encoding, `redis-py` for Streams, and `boto3` for S3-compatible storage. |
| **Colour-managed LUT processing** | Decode with camera white balance and no automatic brightness; convert to the preset's declared working space; apply tetrahedral interpolation; convert to display sRGB; downscale, watermark, and encode WebP. Initially accept only labelled display-referred sRGB `.cube` presets. |
| **Idempotent jobs and trial reservations** | A retry must not duplicate a preview or consume another trial. Finalize a trial only after successful rendering; release it on permanent failure. |
