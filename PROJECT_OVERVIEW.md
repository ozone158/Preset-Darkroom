# Preset Darkroom — Project Overview

## What it is

Preset Darkroom is a trial-preview platform for photographers and other creators who sell RAW photo-editing presets. It lets a prospective buyer upload one of their own photos, choose a preset, and receive a watermarked preview of how that preset affects their image before purchasing.

The product solves a common problem in preset sales: gallery examples cannot show how a preset will behave with a specific buyer's camera, lighting, skin tones, composition, or exposure. A limited trial preview gives buyers confidence while protecting the creator's work.

## Who it serves

### Buyers

Buyers can test a preset on their own RAW image without buying blindly. They receive a smaller, watermarked preview rather than a reusable full-resolution export.

### Preset sellers

Sellers can offer a more convincing purchase path while retaining control over trial limits, preview quality, watermarking, and access to original preset files.

### Platform operators

Operators can manage presets, trial allowances, rendering capacity, failures, and storage without exposing internal Workers to the public internet.

## Main user journey

1. A buyer chooses a preset and uploads a RAW image.
2. The public API validates the request and checks the buyer's available trial allowance.
3. The API creates a preview job and places a small message on a Redis Stream.
4. A private Worker receives the job and renders a preview.
5. The Worker saves a watermarked WebP preview to private storage and marks the job complete.
6. The buyer's browser polls the job-status endpoint.
7. When the preview is ready, the API verifies ownership and returns a short-lived signed URL.
8. The buyer views the preview and decides whether to purchase the preset.

## Architecture at a glance

```text
Buyer browser
    |
    | HTTPS REST
    v
Gateway API -----> PostgreSQL job state
    |
    | Redis Streams: preview:render
    v
Private Worker(s)
    |
    | HTTPS / S3-compatible storage API
    v
Private uploads and preview outputs
```

The Gateway is the only public service. It handles browser requests, validation, trial limits, job creation, job status, and signed preview URLs. Workers are private and perform the CPU- and memory-intensive image processing.

Redis Streams connects the Gateway to Workers asynchronously. This matters because RAW rendering can take longer than a normal web request. The API responds quickly with `202 Accepted`, while Workers render in the background. A consumer group distributes jobs across Workers; acknowledgements and stale-job recovery prevent work from disappearing when a Worker stops unexpectedly.

## Preview rendering

The Worker is designed around a controlled, preview-sized image pipeline:

1. Read the RAW file with `rawpy`, backed by LibRaw.
2. Use camera white balance and disable automatic brightness to produce a predictable exposure baseline.
3. Downsample to a maximum long edge of 1920 pixels before expensive LUT processing.
4. Convert pixels to the colour space and transfer curve expected by the preset.
5. Apply the preset's `.cube` 3D LUT using tetrahedral interpolation through `colour-science`.
6. Convert to display sRGB, add a visible semi-transparent watermark, and encode a WebP preview with Pillow.

The downsampling step happens before LUT interpolation deliberately. Applying a 3D LUT to a full-resolution 45-megapixel RAW image can cause extreme CPU use or out-of-memory failures, while a 1920px preview is appropriate for a trial experience.

For the first version, the platform should accept only labelled, display-referred sRGB `.cube` files. Each preset should declare its expected input colour space and transfer curve; applying a LUT in the wrong colour domain leads to inaccurate colour, especially in skin tones and highlights.

## Data and storage

Uploaded RAW files and generated previews are private. A Worker writes previews under a job-specific, non-guessable object key, for example:

```text
uploads/{job_id}/source.CR3
outputs/{job_id}/preview.webp
```

The browser never receives a raw storage path. Once a job is complete, the Gateway confirms that the requesting buyer owns the job and issues a short-lived signed HTTPS URL for the output. In development, local folders can mimic object storage; production should use S3, MinIO, or another S3-compatible private object store.

## Trial limits and reliability

Trial limits protect the value of a seller's presets. When a preview request is accepted, the Gateway atomically reserves one trial in Redis. It finalizes that reservation only after successful rendering. If a job permanently fails, the platform restores the trial allowance.

Jobs are idempotent: running the same job more than once must not create duplicate previews or consume extra trials. Workers acknowledge Redis Stream entries only after output and job state are durably recorded. A background recovery loop uses `XAUTOCLAIM` to recover abandoned pending jobs; exhausted retries move to `preview:dead-letter` for review.

Corrupt or truncated RAW files are permanent failures, not retryable ones. Retrying them wastes Worker capacity and can create a processing loop.

## Technology choices

| Area | Initial choice | Purpose |
| --- | --- | --- |
| Public API | FastAPI | Uploads, job creation, job status, signed URLs |
| Background work | Redis Streams and consumer groups | Asynchronous, at-least-once rendering jobs |
| Job persistence | PostgreSQL | Durable job, ownership, and trial records |
| Image decoding | `rawpy` / LibRaw | Camera RAW support and demosaicing |
| LUT and colour work | `numpy` and `colour-science` | Efficient pixel arrays and tetrahedral 3D-LUT interpolation |
| Watermark/output | Pillow | Watermarking and WebP/JPEG encoding |
| Object storage | MinIO locally; S3-compatible storage in production | Private uploads, outputs, and signed URLs |
| Local orchestration | Docker Compose | Repeatable local development environment |

Thrift is intentionally not used on the preview-rendering path. Redis Streams allows the Gateway and Workers to scale and recover independently; a synchronous RPC call would keep the public request coupled to rendering time.

## Current build approach

Development proceeds from the inside out:

1. Prove RAW decoding, downsampling, LUT application, watermarking, and WebP output in one standalone script.
2. Put that proven renderer behind a Redis Streams Worker loop.
3. Add the smallest possible FastAPI Gateway with upload and polling endpoints.
4. Add PostgreSQL persistence and transactional trial reservations.
5. Add signed object-storage delivery, Docker Compose, retry recovery, and production hardening.

This order reduces risk because the rendering core is the most fragile and expensive part of the system. Building the web and infrastructure layers first would hide image-pipeline performance and colour problems until much later.

## Success criteria for the first version

- A buyer uploads a supported RAW image and chooses a preset.
- The API responds immediately with a preview-job ID.
- A Worker creates a colour-correct, 1920px-or-smaller, watermarked WebP preview.
- The buyer can retrieve only their own completed preview through a time-limited URL.
- Trial limits are enforced fairly, including when jobs fail or retry.
- Failed, stalled, and corrupt-file jobs reach a clear terminal state rather than remaining stuck.
