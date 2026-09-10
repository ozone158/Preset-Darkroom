# Preset Darkroom — Core-First Build Roadmap

Build from the fragile rendering core outward:

```text
Core Engine → Redis Streams Consumer → Gateway → Persistent State & Security → Deployment
```

Do not ask an AI to generate the Gateway, PostgreSQL, Redis, and Docker Compose stack at the start. Validate the RAW and 3D-LUT pipeline first; it is the highest-risk part of the product.

## 1. Prove the rendering core with one script

Do not start a web service, Redis, database, or Docker. Prepare one representative `.CR3` or `.ARW` RAW file and one known-good `.cube` file. Create one standalone script: `test_render.py`.

### Build

- [ ] Read the RAW image with `rawpy`.
- [ ] Use `raw.postprocess(use_camera_wb=True, no_auto_bright=True)` to establish the exposure baseline.
- [ ] **Before applying the LUT**, resize the image so its long edge is at most 1920 pixels.
- [ ] Apply the `.cube` file with `colour-science` using tetrahedral interpolation.
- [ ] Add a semi-transparent diagonal watermark with Pillow.
- [ ] Save `preview.webp`.

### Why this comes first

Applying tetrahedral interpolation to a full 45-megapixel RAW image can exhaust memory or stall the CPU. Preview resolution is enough for a buyer's trial, so downsample before the LUT rather than after it.

### Works when

`python test_render.py sample.CR3 preset.cube` creates a colour-correct, watermarked `preview.webp` within an acceptable time on the development machine.

### AI prompt guardrail

> Before applying the 3D LUT, downsample the image to a maximum long edge of 1920 pixels. Never pass the full RAW-resolution ndarray to tetrahedral LUT interpolation.

## 2. Wrap the working core in a Redis Streams Worker

Only after the standalone script works, start a local Redis instance:

```bash
docker run -p 6379:6379 redis:alpine
```

Create `worker/job_consumer.py` and move the rendering code from `test_render.py` into an importable module.

### Build

- [ ] Create the `preview-workers` consumer group for the `preview:render` stream.
- [ ] Read messages with `XREADGROUP`.
- [ ] Call the proven rendering module with the job's RAW path and preset path.
- [ ] Write output to `shared_storage/outputs/{job_id}/preview.webp`.
- [ ] Acknowledge successful work with `XACK`.
- [ ] Write basic status to Redis: `queued`, `processing`, `complete`, or `failed`.

### Manual test

```bash
XADD preview:render * job_id 101 raw_path /mock/sample.CR3 preset /mock/test.cube
```

### Works when

The Worker consumes the message, renders the preview, saves `shared_storage/outputs/101/preview.webp`, marks the job `complete`, and acknowledges the Stream entry.

### Failure rule

A corrupt or truncated RAW file is not retryable. Mark it `failed` with a useful error message; do not let the Worker retry it forever.

## 3. Add a thin Gateway and polling loop

Introduce FastAPI only after the Worker loop is proven. Keep state in Redis for now; do not add PostgreSQL yet.

### Build

- [ ] Add `POST /api/v1/previews`.
- [ ] Accept an uploaded file and store it in `shared_storage/uploads/`.
- [ ] Generate a job ID and set its Redis Hash status to `queued`.
- [ ] Append the job to `preview:render` using `XADD`.
- [ ] Return `202 Accepted`, `job_id`, and a status URL immediately.
- [ ] Add `GET /api/v1/previews/{job_id}`.
- [ ] Return the Redis Hash status and the local preview URL after completion.

### Works when

A cURL request or Postman upload creates a job that moves from `queued` to `complete` while the caller polls its status endpoint and can open the generated preview.

## 4. Persist state and add trial reservations

Add PostgreSQL only when the upload-to-preview loop is reliable. This is where concurrent trial-limit bugs are most likely, so keep the implementation small and test it deliberately.

### Build

- [ ] Add a `jobs` table and a user trial-allowance table.
- [ ] Save every created job to PostgreSQL.
- [ ] Reserve one trial atomically in Redis when the Gateway accepts a preview request, using `DECR` with validation or a Lua script.
- [ ] On successful rendering, persist that the reservation was consumed.
- [ ] When retries are exhausted and the job moves to `preview:dead-letter`, restore the buyer's trial allowance.
- [ ] Make each job idempotent so retrying it cannot consume another trial.

### Works when

Concurrent requests cannot exceed a user's allowance, a successful job consumes exactly one trial, and a permanently failed job restores exactly one trial.

## 5. Add production edges last

Only after the local end-to-end workflow is stable, replace local infrastructure with production-ready equivalents.

### Build

- [ ] Abstract local file access behind a storage interface.
- [ ] Move uploads and previews to S3 or MinIO through its SDK.
- [ ] Return five-minute presigned URLs after the Gateway verifies job ownership.
- [ ] Add `docker-compose.yml` for Gateway, Worker, PostgreSQL, Redis, and MinIO.
- [ ] Add a Worker background loop that uses `XAUTOCLAIM` to recover abandoned pending jobs.
- [ ] Add retry limits and send exhausted jobs to `preview:dead-letter`.

### Works when

The whole stack starts with Docker Compose; a buyer can submit a job, receive a private time-limited preview URL, and an interrupted Worker job is recovered or fails cleanly.

## Non-negotiable guardrails

- Downsample to a 1920px maximum long edge before applying a 3D LUT.
- Treat invalid/corrupt RAW files as permanent failures, not retryable failures.
- Acknowledge a Redis Stream message only after the output and final state are durable.
- Keep storage private; return a signed URL only after verifying job ownership.
- Build and test one layer at a time. Do not generate the entire stack in one AI request.
