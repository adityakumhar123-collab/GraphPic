# r1_about.md — Architectural Design (Final)

> _Antigravity's reply to i1_about.md — all decisions agreed and locked._  
> _This is the canonical low-level design document for Graphpic._

---

## 1. All Decisions — Locked ✅

| Topic | Decision |
|-------|----------|
| Auth | **JWT** — stateless, fits API + mobile-responsive web |
| QR code | Encodes **full URL** (`/clients/{vault_id}`) — client lands directly on selfie page |
| Frontend location | `apps/web/` inside the monorepo |
| Qdrant strategy | **Single collection** `face_embeddings`, filtered per vault via `vault_id` payload field |
| Mongo TTL | `query_jobs.expires_at` — TTL index, **10 min**, aligned with presigned URL expiry |
| Vault public URL | `vault._id` doubles as the public token — no separate `public_token` field |
| QR code storage | **Not stored in S3** — generated dynamically on demand in the browser |
| Worker strategy | **Python workers talk to Redis directly** — Node.js only produces jobs |
| Client-side preprocessing | **None** — raw selfie sent to API; worker handles detection + embedding |
| S3 key — permanent | `{owner_id}/{vault_id}/{image_id}.ext` |
| S3 key — ephemeral | `temp/{owner_id}/{vault_id}/{job_id}/{image_id}.ext` |
| Temp cleanup | **Mongo TTL (10 min) + S3 Lifecycle Rule (15 min)** — belt and suspenders |
| Multi-face dedup | If **any** embedding for a given `image_id` matches → return that photo exactly once |
| Vault lifecycle | `status: "active" \| "archived"` — archived vaults reject new client queries |
| Poll interval | **Every 2 seconds** |
| No face detected | `query_jobs.status → "failed"`, `error → "no_face_detected"` — UI prompts user to retake |

---

## 2. MongoDB Schemas

### `users` collection
```js
{
  _id: ObjectId,
  email: String,        // unique, indexed
  username: String,     // unique, indexed
  password: String,     // bcrypt hash
  vaults: [
    {
      event_name: String,
      vault_id: ObjectId   // ref → vaults._id
    }
  ],
  created_at: Date,
  updated_at: Date
}
```

### `vaults` collection
```js
{
  _id: ObjectId,             // IS the public token — used directly in /clients/:id URL
  owner_id: ObjectId,        // ref → users._id
  event_name: String,
  status: String,            // "active" | "archived"
                             //   active   → accepts client queries
                             //   archived → rejects client queries; data preserved
  image_count: Number,       // counter, incremented per upload
  created_at: Date,
  updated_at: Date
}
```

### `images` collection
```js
{
  _id: ObjectId,
  vault_id: ObjectId,        // ref → vaults._id  [indexed]
  owner_id: ObjectId,        // ref → users._id
  s3_key: String,            // "{owner_id}/{vault_id}/{_id}.ext"
  original_filename: String,
  file_size: Number,         // bytes
  mime_type: String,
  status: String,            // "pending" | "processing" | "done" | "failed"
  face_count: Number,        // 0 = no faces detected; image kept but unsearchable
  uploaded_at: Date,
  processed_at: Date
}
```

### `query_jobs` collection
```js
{
  _id: ObjectId,
  vault_id: ObjectId,
  status: String,                  // "pending" | "processing" | "done" | "failed"
  result_presigned_urls: [String], // presigned GET URLs to matched images (10 min expiry)
  error: String,                   // e.g. "no_face_detected" | "processing_error"
  created_at: Date,
  expires_at: Date                 // TTL index — 10 min; auto-deletes doc after expiry
}
```

> **TTL Index:** `db.query_jobs.createIndex({ expires_at: 1 }, { expireAfterSeconds: 0 })`

---

## 3. Qdrant Schema

**Collection name:** `face_embeddings`  
**Vector size:** 512 (ArcFace)  
**Distance:** Cosine

```json
{
  "id": "<uuid-v4>",
  "vector": [/* 512 floats */],
  "payload": {
    "owner_id":   "<ObjectId string>",
    "vault_id":   "<ObjectId string>",
    "image_id":   "<ObjectId string>",  // ref → images._id — used for dedup on query
    "s3_key":     "{owner_id}/{vault_id}/{image_id}.ext",
    "face_index": 0                     // 0-based index of this face within the source image
  }
}
```

**Search filter (per query):**
```json
{
  "must": [
    { "key": "vault_id", "match": { "value": "<vault_id>" } }
  ]
}
```

**Similarity threshold:** cosine distance ≤ **0.4** (tune during testing)

**How `face_index` works:**
```python
faces = insightface_app.get(img)     # list of Face objects from InsightFace
for i, face in enumerate(faces):
    embedding = face.embedding        # 512-dim numpy array
    # store with face_index = i
```
It's a debug/audit field. Core dedup logic only uses `image_id`.

---

## 4. S3 Bucket Layout

```
graphpic-bucket/
│
├── {owner_id}/
│   └── {vault_id}/
│       └── {image_id}.{ext}                     ← permanent photographer uploads
│
└── temp/
    └── {owner_id}/
        └── {vault_id}/
            ├── selfies/
            │   └── {job_id}.{ext}               ← client selfie (worker deletes after processing)
            └── query_results/
                └── {job_id}/
                    └── {image_id}.{ext}         ← ephemeral matched images (S3 Lifecycle: 15 min)
```

**S3 Lifecycle Rule:** Applied to `temp/` prefix — auto-deletes all objects after **15 minutes**.  
This is the safety net on top of the Mongo TTL; works even if the worker crashes mid-job.

---

## 5. Redis Queues (BullMQ)

| Queue | Producer | Consumer | Payload |
|-------|----------|----------|---------|
| `image-processing` | Poller (SQS) | `image_worker` (Python) | `{ s3_key, image_id, vault_id, owner_id }` |
| `face-query` | API server | `query_worker` (Python) | `{ job_id, vault_id, selfie_s3_key }` |

### `image-processing` worker — step by step
```
Input: { s3_key, image_id, vault_id, owner_id }

1. Download image from S3 via presigned GET URL
2. Run InsightFace → detect all faces → list of Face objects
3. IF face list is empty:
      Update images: { status: "done", face_count: 0 }
      (image stays in S3; just unsearchable)
      Done.
4. FOR each face (index i):
      a. Crop face region from image
      b. Generate ArcFace 512-dim embedding
      c. Upsert point to Qdrant:
            id         = uuid4()
            vector     = embedding
            payload    = { owner_id, vault_id, image_id, s3_key, face_index: i }
5. Update images: { status: "done", face_count: N }
6. ON any error: Update images: { status: "failed" }, log error
```

### `face-query` worker — step by step
```
Input: { job_id, vault_id, selfie_s3_key }

1. Update query_jobs: { status: "processing" }
2. Download selfie from S3
3. Run InsightFace → detect faces
4. IF no faces detected:
      Update query_jobs: { status: "failed", error: "no_face_detected" }
      Delete selfie from S3
      Done. (UI will receive status:"failed", error:"no_face_detected" → show "try again" prompt)
5. Take the primary face (highest confidence / largest bounding box)
6. Generate ArcFace 512-dim embedding
7. Qdrant similarity search:
      filter: { vault_id: vault_id }
      threshold: cosine distance ≤ 0.4
      limit: e.g. 100 results
8. Extract unique image_ids from results (dedup — any matching face = include that photo)
9. For each unique image_id:
      Generate presigned GET URL (10 min expiry) using s3_key from Qdrant payload
10. Delete selfie from S3 (no longer needed)
11. Update query_jobs:
      { status: "done",
        result_presigned_urls: [...],
        expires_at: now + 10min }
12. ON any error: Update query_jobs: { status: "failed", error: "processing_error" }
```

---

## 6. Upload Flow (Photographer)

```
Browser              API Server           S3              SQS          Poller       Redis       image_worker
  │                      │                 │               │              │            │               │
  ├─ POST /vaults/:id/upload-init          │               │              │            │               │
  │  { filenames[], sizes[], mimes[] } ──► │               │              │            │               │
  │                      ├─ INSERT N image docs (status:pending)          │            │               │
  │                      ├─ generate N presigned POST URLs ──────────────►│            │               │
  │◄─ { presigned_urls[] } ──────────────┤│               │              │            │               │
  │                      │                 │               │              │            │               │
  ├─ PUT × N (direct to S3, parallel) ───────────────────►│              │            │               │
  │                      │                 ├─ S3 event ───►│              │            │               │
  │                      │                 │               ├─ SQS msg ───►│            │               │
  │                      │                 │               │              ├─ push job ►│               │
  │                      │                 │               │              │            ├─ dequeue ────►│
  │                      │                 │               │              │            │               ├─ download image
  │                      │                 │               │              │            │               ├─ detect faces
  │                      │                 │               │              │            │               ├─ gen embeddings
  │                      │                 │               │              │            │               ├─ upsert Qdrant
  │                      │                 │               │              │            │               └─ update images.status→done
```

---

## 7. Client Query Flow

```
Client Browser        API Server         Redis        query_worker        Qdrant        Mongo
  │                       │                │               │                 │              │
  ├─ POST /clients/:vault_id/query         │               │                 │              │
  │  (raw selfie blob) ──────────────────►│               │                 │              │
  │                       ├─ upload selfie → temp/{owner_id}/{vault_id}/selfies/{job_id}   │
  │                       ├─ INSERT query_jobs { status: "pending" } ───────────────────────►│
  │                       ├─ push job to face-query queue ►│               │              │ │
  │◄─ 200 { job_id } ────┤│               │               │               │              │ │
  │                       │                │               │               │              │ │
  ├─ GET /jobs/:job_id ──►│               │               │               │              │ │
  │◄─ { status:"pending"} │                │               │               │              │ │
  │  (polls every 2s)     │                ├─ dequeue ────►│               │              │ │
  │                       │                │               ├─ download selfie from S3      │ │
  │                       │                │               ├─ detect face                  │ │
  │                       │                │               │                               │ │
  │                       │         [no face found]        │                               │ │
  │                       │                │               ├─ UPDATE query_jobs {failed, "no_face_detected"} ►│
  │◄─ { status:"failed",  │               │               │               │              │ │
  │    error:"no_face_    │               │               │               │              │ │
  │    detected" }        │               │               │               │              │ │
  │  → show "try again"   │               │               │               │              │ │
  │                       │                │               │               │              │ │
  │                       │         [face found]           │               │              │ │
  │                       │                │               ├─ gen embedding                │ │
  │                       │                │               ├─ search(vault_id filter) ────►│ │
  │                       │                │               │◄─ matched results ────────────┤ │
  │                       │                │               ├─ dedup by image_id            │ │
  │                       │                │               ├─ gen presigned GET URLs       │ │
  │                       │                │               ├─ delete selfie from S3        │ │
  │                       │                │               ├─ UPDATE query_jobs {done, urls, expires_at} ──►│
  │◄─ { status:"done",    │               │               │               │              │ │
  │    urls:[] }          │               │               │               │              │ │
  ├─ fetch + display images via presigned URLs             │               │              │ │
  │  (TTL + S3 Lifecycle handle cleanup automatically)     │               │              │ │
```

---

## 8. Final Directory Structure ✅

```
graphpic/
│
├── package.json                        ← root — workspace config, shared dev scripts (lint, format)
├── pnpm-workspace.yaml                 ← declares apps/* and packages/* as pnpm workspaces
├── .env.example                        ← canonical list of ALL env vars across every app
├── .gitignore
├── README.md
├── docker-compose.yml                  ← local dev: spins up mongo, redis, qdrant + all apps
│
├── config/                             ← non-secret, environment-level config
│   ├── default.json                    ← shared defaults (ports, queue names, TTL durations, thresholds)
│   ├── development.json                ← dev overrides
│   └── production.json                 ← prod overrides
│
├── packages/                           ← shared internal libraries — no app-specific logic lives here
│   │
│   ├── core/                           ← @graphpic/core — consumed by api, poller, workers (JS side)
│   │   ├── package.json
│   │   ├── db/
│   │   │   ├── index.js                ← exports connectMongo() — initialises Mongoose connection
│   │   │   └── models/
│   │   │       ├── User.js             ← Mongoose User schema + model
│   │   │       ├── Vault.js            ← Mongoose Vault schema + model (status field included)
│   │   │       ├── Image.js            ← Mongoose Image schema + model (per-image, not embedded)
│   │   │       └── QueryJob.js         ← Mongoose QueryJob schema + TTL index + model
│   │   └── services/
│   │       ├── s3.js                   ← createPresignedPost(), getPresignedGet(), deleteObject()
│   │       ├── queue.js                ← BullMQ queue factory — getQueue(name) with shared Redis config
│   │       └── qdrant.js               ← qdrantClient singleton (JS client — used by API if ever needed)
│   │
│   └── shared-types/                   ← @graphpic/types — JSDoc typedefs shared across all JS apps
│       └── package.json
│
├── apps/
│   │
│   ├── api/                            ← @graphpic/api — Express.js REST API (Node.js)
│   │   ├── package.json
│   │   ├── Dockerfile
│   │   └── src/
│   │       ├── index.js                ← entry point: boot Express, connect Mongo, start listening
│   │       ├── app.js                  ← Express app factory: apply middleware, mount all routers
│   │       ├── routes/
│   │       │   ├── index.js            ← central router — mounts all sub-routers at their base paths
│   │       │   ├── auth.routes.js      ← POST /auth/register  POST /auth/login  POST /auth/logout
│   │       │   ├── vault.routes.js     ← GET|POST /vaults  DELETE /vaults/:id
│   │       │   │                         POST /vaults/:id/upload-init  PATCH /vaults/:id/archive
│   │       │   └── client.routes.js    ← POST /clients/:vault_id/query   GET /jobs/:job_id
│   │       ├── controllers/
│   │       │   ├── auth.controller.js  ← register (hash pw, save user), login (verify, sign JWT), logout
│   │       │   ├── vault.controller.js ← create vault, list vaults, batch presigned URL generation, archive
│   │       │   └── client.controller.js← upload selfie to S3, insert query_job, enqueue face-query job,
│   │       │                             poll and return query_job status + presigned result URLs
│   │       ├── middleware/
│   │       │   ├── auth.middleware.js  ← verify JWT, attach req.user; skip for /clients/* routes
│   │       │   ├── validate.js         ← Zod schema validation wrapper for req.body / req.params
│   │       │   └── errorHandler.js     ← global Express error handler — formats all errors consistently
│   │       └── lib/
│   │           └── presignedUpload.js  ← takes file metadata array, calls s3.createPresignedPost() in batch,
│   │                                     inserts image docs, returns { presigned_urls, image_ids }
│   │
│   ├── workers/                        ← Python face-processing workers (talk to Redis directly)
│   │   ├── Dockerfile                  ← single Docker image; CMD runs both workers as separate processes
│   │   ├── shared/                     ← code shared between image_worker and query_worker
│   │   │   ├── s3_ops.py               ← download_from_s3(), generate_presigned_get(), delete_from_s3()
│   │   │   ├── mongo_ops.py            ← update_image_status(), update_query_job()  (pymongo)
│   │   │   └── config.py               ← loads env vars — Redis URL, S3 bucket, Qdrant host, thresholds
│   │   ├── image_worker/               ← consumes `image-processing` queue
│   │   │   ├── worker.py               ← entry: BullMQ-compatible Redis consumer loop; dispatches jobs
│   │   │   ├── face_pipeline.py        ← load image → InsightFace detection → ArcFace embeddings (all faces)
│   │   │   ├── qdrant_ops.py           ← upsert_embeddings(): writes all face points for one image to Qdrant
│   │   │   └── requirements.txt        ← insightface, onnxruntime-gpu, opencv-python, numpy, qdrant-client,
│   │   │                                  pymongo, boto3, redis
│   │   └── query_worker/               ← consumes `face-query` queue
│   │       ├── worker.py               ← entry: BullMQ-compatible Redis consumer loop; dispatches jobs
│   │       ├── face_pipeline.py        ← load selfie → InsightFace → ArcFace (primary face only)
│   │       ├── search.py               ← qdrant_search(vault_id, embedding, threshold) → unique image_ids
│   │       ├── result_ops.py           ← generate presigned GET URLs for matched images, write to query_jobs
│   │       └── requirements.txt
│   │
│   ├── poller/                         ← @graphpic/poller — SQS → Redis bridge (Node.js)
│   │   ├── package.json
│   │   ├── Dockerfile
│   │   └── src/
│   │       ├── index.js                ← entry: starts the long-poll loop against SQS
│   │       └── sqs.js                  ← receiveMessages(), deleteMessage(), parseS3Event()
│   │                                     — extracts s3_key, maps to image_id+vault_id, pushes to BullMQ
│   │
│   └── web/                            ← @graphpic/web — React + Vite (serves both photographer and client UI)
│       ├── package.json
│       ├── vite.config.js
│       ├── index.html
│       ├── public/
│       └── src/
│           ├── main.jsx                ← React DOM entry
│           ├── App.jsx                 ← react-router-dom setup: / , /vaults/:id , /clients/:vault_id
│           ├── pages/
│           │   ├── Auth.jsx            ← login + register forms; on success stores JWT in memory/httpOnly cookie
│           │   ├── Dashboard.jsx       ← photographer: vault list, create vault button, QR display per vault
│           │   ├── VaultDetail.jsx     ← photographer: single vault view, drag-drop bulk uploader, upload progress
│           │   └── ClientView.jsx      ← client: camera selfie capture → polling state → results gallery
│           │                             shows "No face detected — try again" on failed jobs
│           ├── components/
│           │   ├── Uploader.jsx        ← drag-drop zone; calls useUpload hook; shows per-file progress
│           │   ├── QRDisplay.jsx       ← generates QR code dynamically from vault URL (qrcode.react or similar)
│           │   ├── FaceCapture.jsx     ← opens webcam via getUserMedia, captures frame, sends raw blob to API
│           │   └── ResultsGallery.jsx  ← renders matched images from presigned GET URLs in a responsive grid
│           ├── hooks/
│           │   ├── useUpload.js        ← calls /upload-init, batches direct S3 PUTs with concurrency control,
│           │   │                         tracks per-file progress state
│           │   └── useJobPoller.js     ← polls GET /jobs/:id every 2s; resolves on "done", rejects on "failed";
│           │                             surfaces error code so UI can show the right message
│           ├── api/
│           │   └── client.js           ← fetch API wrapper with base URL, auto-attaches JWT header & handles 401
│           └── store/
│               └── index.js            ← Redux Toolkit store — auth slice (user, token), vault slice, active job slice
│
└── manuals/
    ├── docs/                           ← final/generated project documentation (API spec, deployment guide, etc.)
    └── instructions/                   ← working design notes
        ├── i1_about.md                 ← original project description from developer
        └── r1_about.md                 ← this file — full agreed architecture
```

---

## 9. Design Decisions — Complete Reference

| Decision | Choice | Reason |
|----------|--------|--------|
| Auth | **JWT** | Stateless; works across API + mobile-responsive web |
| Worker language | **Python** | InsightFace / ArcFace / ONNX Runtime are Python-native |
| Worker → queue | **Direct Redis** (Option A) | Node.js only produces; Python only consumes — clean boundary |
| Queue library | **BullMQ + Redis** | Retries, concurrency, delayed jobs, job state all built-in |
| S3 upload | **Direct browser → S3** via presigned POST | Server never touches image bytes — saves bandwidth + memory |
| Qdrant namespace | **Single collection + payload filter** | No per-vault collection lifecycle; simpler ops |
| Image metadata | **Separate `images` collection** | Avoids 16 MB Mongo doc limit; enables per-image status tracking |
| Vault public URL | **`vault._id` directly** | No redundant token field; ObjectId is safe to expose publicly |
| Vault lifecycle | **`status: active/archived`** | Photographer can close events without deleting data |
| QR generation | **Dynamic (client-side)** | No S3 write needed; generated from vault URL on the fly |
| Temp cleanup | **Mongo TTL (10 min) + S3 Lifecycle (15 min)** | Works even if client ghosts or worker crashes |
| Multi-face handling | **One embedding per face in Qdrant, dedup by `image_id` on query** | Correct multi-person photos; no duplicate results |
| No face detected | **`status: "failed"`, `error: "no_face_detected"`** → UI shows "try again" | Clear UX; avoids silent empty results |
| Client polling | **Every 2s, 10-min result expiry** | Fast enough UX; conservative enough on server load |

---

_— Antigravity_

