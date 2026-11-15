# Dev1_instruct.md — You / Repo Owner
## Role: Infra Setup · Python Workers · Poller

---

## Your Owned Files & Directories

```
graphpic/
├── package.json                   ← yours
├── pnpm-workspace.yaml            ← yours
├── .env.example                   ← yours
├── .gitignore                     ← yours
├── README.md                      ← yours
├── docker-compose.yml             ← yours
├── config/                        ← entire dir, yours
│   ├── default.json
│   ├── development.json
│   └── production.json
├── apps/
│   ├── workers/                   ← entire dir, yours — no one else touches this
│   │   ├── Dockerfile
│   │   ├── shared/
│   │   │   ├── config.py
│   │   │   ├── s3_ops.py
│   │   │   └── mongo_ops.py
│   │   ├── image_worker/
│   │   │   ├── worker.py
│   │   │   ├── face_pipeline.py
│   │   │   ├── qdrant_ops.py
│   │   │   └── requirements.txt
│   │   └── query_worker/
│   │       ├── worker.py
│   │       ├── face_pipeline.py
│   │       ├── search.py
│   │       ├── result_ops.py
│   │       └── requirements.txt
│   └── poller/                    ← entire dir, yours — no one else touches this
│       ├── package.json
│       ├── Dockerfile
│       └── src/
│           ├── index.js
│           └── sqs.js
```

---

## What Each File Does

### Root & Config
| File | Purpose |
|------|---------|
| `package.json` | Root workspace config — `"workspaces": ["apps/*","packages/*"]`, shared dev scripts |
| `pnpm-workspace.yaml` | Declares pnpm workspace packages |
| `.env.example` | Canonical list of every env var across all apps — teammates copy this to `.env` |
| `.gitignore` | Ignore `node_modules/`, `.env`, `__pycache__/`, `*.pyc`, `dist/` |
| `README.md` | Project overview and local dev setup instructions |
| `docker-compose.yml` | Spins up MongoDB, Redis, Qdrant + all 4 apps for local dev |
| `config/default.json` | Shared defaults: port numbers, queue names, TTL values, similarity threshold |
| `config/development.json` | Dev overrides (local hosts, debug flags) |
| `config/production.json` | Prod overrides (use env vars, disable debug) |

### `apps/poller/`
| File | Purpose |
|------|---------|
| `src/sqs.js` | `receiveMessages()`, `deleteMessage()`, `parseS3Event()` — extracts `s3_key`, `image_id`, `vault_id` from SQS notification |
| `src/index.js` | Long-poll loop: receive SQS messages → push job to BullMQ `image-processing` queue |
| `package.json` | Poller package — depends on `@graphpic/core`, `@aws-sdk/client-sqs`, `bullmq` |
| `Dockerfile` | Node.js image for the poller service |

### `apps/workers/shared/`
| File | Purpose |
|------|---------|
| `config.py` | Loads all env vars — Redis URL, S3 bucket, Qdrant host, similarity threshold, presigned URL expiry |
| `s3_ops.py` | `download_from_s3(key)`, `generate_presigned_get(key, expiry)`, `delete_from_s3(key)` — boto3 |
| `mongo_ops.py` | `update_image_status(image_id, status, face_count)`, `update_query_job(job_id, status, urls, error)` — pymongo |

### `apps/workers/image_worker/`
| File | Purpose |
|------|---------|
| `face_pipeline.py` | Load image (OpenCV) → InsightFace detect all faces → ArcFace 512-dim embedding per face |
| `qdrant_ops.py` | `upsert_embeddings(image_id, vault_id, owner_id, s3_key, embeddings_list)` → writes N points to Qdrant |
| `worker.py` | Redis consumer loop for `image-processing` queue; orchestrates: download → pipeline → qdrant upsert → mongo update |
| `requirements.txt` | `insightface onnxruntime-gpu opencv-python numpy qdrant-client pymongo boto3 redis` |

### `apps/workers/query_worker/`
| File | Purpose |
|------|---------|
| `face_pipeline.py` | Same InsightFace + ArcFace pipeline but processes one selfie; returns single embedding (highest confidence face) |
| `search.py` | `qdrant_search(vault_id, embedding, threshold)` → returns list of unique `image_id`s (deduped by `image_id` payload field) |
| `result_ops.py` | Generates presigned GET URLs for matched `image_id`s; calls `update_query_job()` with done status + URLs |
| `worker.py` | Redis consumer loop for `face-query` queue; orchestrates: download selfie → pipeline → search → result_ops → delete selfie |
| `requirements.txt` | Same as image_worker |

### `apps/workers/Dockerfile`
Single Docker image — `CMD` launches both `image_worker/worker.py` and `query_worker/worker.py` as separate processes.

---

## Things NOT in Your Files (coordinate with team)

You will need these interfaces from Dev 2 — agree on them before writing workers:

| What | Who | Agreement needed |
|------|-----|-----------------|
| Queue name constants | Dev 2 (`queue.js`) | `"image-processing"` and `"face-query"` — confirm strings |
| `images` Mongo schema | Dev 2 (`Image.js`) | Fields your `mongo_ops.py` will write to: `status`, `face_count`, `processed_at` |
| `query_jobs` Mongo schema | Dev 2 (`QueryJob.js`) | Fields your `mongo_ops.py` will write to: `status`, `result_presigned_urls`, `error`, `expires_at` |
| Qdrant collection name | Agreed in architecture | `face_embeddings` — 512-dim cosine |

---

## Are the Files Exhaustive?

For your scope: **yes**, with these notes:

- You may add a `apps/workers/shared/__init__.py` and `apps/workers/image_worker/__init__.py` etc. to make Python treat them as packages — standard Python practice.
- You may add a `apps/workers/.env` or rely on the root `.env` passed through Docker — decide during implementation.
- `docker-compose.yml` may reference external service images (mongo:7, redis:7, qdrant/qdrant) that don't need extra files.
- No additional files needed for the poller unless you add unit tests (then add `apps/poller/tests/` dir).

---

## Git & GitHub Workflow

### Phase 1 — Push the skeleton (do this first, unblocks teammates)

```bash
cd c:\Users\LENOVO\MY_STUFFS\Code_and_Development\Graphpic

git init
git add .
git commit -m "chore: initial project skeleton — all directories and empty files"

# Create the GitHub repo in browser, then:
git remote add origin https://github.com/<your-username>/graphpic.git
git branch -M main
git push -u origin main
```

**Share the repo URL with the other three immediately.**  
This is the ONLY direct push to `main` ever. All future work — including yours — goes through feature branches.

---

### Phase 2 — Your own feature branch

```bash
git checkout -b feature/workers-and-poller

# Work on your files...

# Commit as you go:
git add apps/workers/ apps/poller/ config/ docker-compose.yml .env.example package.json pnpm-workspace.yaml .gitignore README.md
git commit -m "feat(poller): SQS long-poll and BullMQ job push"
git commit -m "feat(workers): image_worker face pipeline and qdrant upsert"
git commit -m "feat(workers): query_worker selfie search and result generation"
git commit -m "chore: docker-compose with mongo, redis, qdrant, all apps"

git push origin feature/workers-and-poller
```

---

### Phase 3 — Stay in sync with main when teammates merge

```bash
git fetch origin
git merge origin/main
# Your dirs are isolated — conflicts are extremely unlikely
```

---

### Phase 4 — Open your own PR

1. Push: `git push origin feature/workers-and-poller`
2. Go to your GitHub repo → **Compare & pull request**
3. Base: `main` ← Compare: `feature/workers-and-poller`
4. Title: `feat: Python workers (image + query), poller, docker-compose`
5. Since you own the repo you can self-review and merge

---

### Phase 5 — Review and merge teammates' PRs

When a teammate opens a PR:
1. **Pull Requests** tab on GitHub
2. Review only files in their assigned directories
3. Approve → **Squash and merge**

**Recommended merge order:**
```
Dev 1 skeleton → Dev 2 (core + auth) → Dev 3 (vault CRUD) → Dev 4 (client + wiring)
```

---

## Env Vars You Own (put in `.env.example`)

```env
# AWS
AWS_REGION=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
S3_BUCKET_NAME=
SQS_QUEUE_URL=

# Redis
REDIS_URL=

# Qdrant
QDRANT_HOST=
QDRANT_PORT=6333
QDRANT_COLLECTION=face_embeddings

# Worker tuning
SIMILARITY_THRESHOLD=0.4
PRESIGNED_URL_EXPIRY_SECONDS=600

# MongoDB (also used by workers)
MONGODB_URI=

# Node
PORT=3000
NODE_ENV=development

# Auth (Dev 2 fills these)
JWT_SECRET=
JWT_EXPIRY=7d
```
