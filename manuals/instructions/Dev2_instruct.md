# Dev2_instruct.md — Auth + Shared Core
## Role: JWT Authentication · All Mongoose Models · Shared Services

---

## Your Owned Files & Directories

```
graphpic/
├── packages/
│   └── core/                      ← entire dir, yours — no one else modifies this
│       ├── package.json
│       ├── db/
│       │   ├── index.js
│       │   └── models/
│       │       ├── User.js
│       │       ├── Vault.js
│       │       ├── Image.js
│       │       └── QueryJob.js
│       └── services/
│           ├── s3.js
│           ├── queue.js
│           └── qdrant.js
├── apps/
│   ├── api/
│   │   └── src/
│   │       ├── middleware/        ← entire dir, yours
│   │       │   ├── auth.middleware.js
│   │       │   ├── validate.js
│   │       │   └── errorHandler.js
│   │       ├── routes/
│   │       │   └── auth.routes.js      ← this file only, yours
│   │       └── controllers/
│   │           └── auth.controller.js  ← this file only, yours
│   └── web/
│       └── src/
│           ├── pages/
│           │   └── Auth.jsx            ← this file only, yours
│           ├── api/
│           │   └── client.js           ← entire dir, yours
│           └── store/
│               └── index.js            ← entire dir, yours
```

**`packages/core/` is entirely yours. The other three devs import from it but never modify it.**

---

## What Each File Does

### `packages/core/db/index.js`
Exports `connectMongo(uri)` — initialises Mongoose connection. Called once at API startup.

### `packages/core/db/models/User.js`
```js
// Schema:
{
  _id: ObjectId,
  email: String,      // unique, indexed
  username: String,   // unique, indexed
  password: String,   // bcrypt hash — hashed in pre-save hook, never raw
  vaults: [{ event_name: String, vault_id: ObjectId }],
  created_at: Date,
  updated_at: Date
}
// Pre-save hook: bcrypt.hash(password, 12)
// Instance method: user.comparePassword(candidate) → Boolean
```

### `packages/core/db/models/Vault.js`
```js
{
  _id: ObjectId,         // IS the public token used in /clients/:id URL
  owner_id: ObjectId,    // ref → users._id
  event_name: String,
  status: String,        // "active" | "archived"
  image_count: Number,   // counter, $inc on each upload batch
  created_at: Date,
  updated_at: Date
}
```

### `packages/core/db/models/Image.js`
```js
{
  _id: ObjectId,
  vault_id: ObjectId,    // indexed
  owner_id: ObjectId,
  s3_key: String,        // "{owner_id}/{vault_id}/{_id}.ext"
  original_filename: String,
  file_size: Number,
  mime_type: String,
  status: String,        // "pending" | "processing" | "done" | "failed"
  face_count: Number,    // 0 = no faces found
  uploaded_at: Date,
  processed_at: Date
}
```

### `packages/core/db/models/QueryJob.js`
```js
{
  _id: ObjectId,
  vault_id: ObjectId,
  status: String,                   // "pending" | "processing" | "done" | "failed"
  result_presigned_urls: [String],
  error: String,                    // "no_face_detected" | "processing_error"
  created_at: Date,
  expires_at: Date                  // TTL index — auto-delete after 10 min
}
// TTL index: schema.index({ expires_at: 1 }, { expireAfterSeconds: 0 })
```

### `packages/core/services/s3.js`
| Export | Signature | Does |
|--------|-----------|------|
| `createPresignedPost` | `(key, mimeType, maxSizeMB)` | Returns `{ url, fields }` for direct browser upload |
| `getPresignedGet` | `(key, expirySeconds)` | Returns a time-limited GET URL for a private S3 object |
| `deleteObject` | `(key)` | Deletes one object from S3 |

### `packages/core/services/queue.js`
```js
// Exports:
export const IMAGE_PROCESSING_QUEUE = "image-processing"
export const FACE_QUERY_QUEUE       = "face-query"
export const getQueue = (name) => new Queue(name, { connection: redisConfig })
```
Shared BullMQ queue factory — all apps use this to produce jobs; Python workers consume from Redis directly using the same queue names.

### `packages/core/services/qdrant.js`
Exports a singleton `qdrantClient` — initialised `QdrantClient` using `QDRANT_HOST` env var. Used by the API if it ever needs to query Qdrant directly (optional for now).

---

### `apps/api/src/middleware/auth.middleware.js`
- `verifyJWT(req, res, next)` — reads `Authorization: Bearer <token>`, verifies with `jsonwebtoken`, attaches `req.user = { _id, email, username }`
- Skips for `/auth/*` and `/clients/*` routes (those are unauthenticated)

### `apps/api/src/middleware/validate.js`
- `validate(zodSchema)` — middleware factory; validates `req.body`, returns `400 { error, issues }` on failure

### `apps/api/src/middleware/errorHandler.js`
- Global Express error handler (4-arg: `err, req, res, next`)
- Returns consistent `{ error: message, code }` JSON for all thrown errors

---

### `apps/api/src/routes/auth.routes.js`
```
POST /auth/register   → auth.controller.register
POST /auth/login      → auth.controller.login
POST /auth/logout     → auth.controller.logout
```

### `apps/api/src/controllers/auth.controller.js`
| Handler | Logic |
|---------|-------|
| `register` | Validate input → check uniqueness → create User → sign JWT → return `{ token, user }` |
| `login` | Find by email → `user.comparePassword()` → sign JWT → return `{ token, user }` |
| `logout` | Return 200 (JWT is stateless; invalidation is client-side — note this in code comments) |

---

### `apps/web/src/pages/Auth.jsx`
- Tabbed form: Login / Register
- On success: stores JWT, dispatches login action to Redux store, redirects to `/`
- On error: inline field-level validation messages

### `apps/web/src/api/client.js`
- Native `fetch` wrapper function (e.g. `apiClient(endpoint, options)`) with `baseURL = import.meta.env.VITE_API_URL`
- Request interceptor logic: reads token from Redux store (`store.getState().auth.token`) → adds `Authorization: Bearer <token>`
- Response handler: parses JSON, on 401 response status → dispatches `logout()` action to Redux store → redirects to `/auth`

### `apps/web/src/store/index.js`
Redux Toolkit store (`configureStore`) with slices:
```js
// authSlice (yours — fully implement):
// { name: 'auth', initialState: { user: null, token: null }, reducers: { setCredentials, logout } }

// vaultSlice (stub — Dev 3 will populate actions/reducers):
// { name: 'vaults', initialState: { list: [] }, reducers: { setVaults } }

// jobSlice (stub — Dev 4 will populate actions/reducers):
// { name: 'job', initialState: { activeJobId: null, status: null }, reducers: { setJob, clearJob } }
```
> Coordinate with Dev 3 and Dev 4 on what state/actions they need in the store before you finalise it.

---

## Are the Files Exhaustive?

For your scope: **yes**, with these notes:

- You may want to add `packages/core/db/models/index.js` to re-export all models from one place — nice DX, not strictly required.
- If you add input validation schemas (Zod), you can put them inside the respective controller files or create `apps/api/src/schemas/auth.schema.js` — a `schemas/` dir would be a new directory (acceptable, just mention it).
- No additional files needed for the auth flow.

---

## Git & GitHub Workflow

### Step 1 — Clone after Dev 1 pushes skeleton

Wait for Dev 1's message that the skeleton is on GitHub, then:

```bash
git clone https://github.com/<dev1-username>/graphpic.git
cd graphpic
pnpm install
```

### Step 2 — Create your feature branch

```bash
git checkout -b feature/auth-and-core
```

### Step 3 — Commit as you go

```bash
# Stage only YOUR files
git add packages/core/
git add apps/api/src/middleware/
git add apps/api/src/routes/auth.routes.js
git add apps/api/src/controllers/auth.controller.js
git add apps/web/src/pages/Auth.jsx
git add apps/web/src/api/client.js
git add apps/web/src/store/index.js

git commit -m "feat(core): mongoose models — User, Vault, Image, QueryJob"
git commit -m "feat(core): shared services — s3, queue, qdrant"
git commit -m "feat(auth): JWT register and login endpoints"
git commit -m "feat(middleware): verifyJWT, validate, errorHandler"
git commit -m "feat(web): Auth page, fetch client, Redux store"
```

### Step 4 — Stay in sync

```bash
git fetch origin
git merge origin/main
```

### Step 5 — Push and open PR

```bash
git push origin feature/auth-and-core
```

On GitHub:
1. **Compare & pull request** → `feature/auth-and-core → main`
2. Title: `feat: auth endpoints, JWT middleware, all core models and services`
3. **Ship this PR early** — Dev 3 and Dev 4 are blocked on your models and services
4. Request review from Dev 1; do not self-merge

### Step 6 — After merge

```bash
git checkout main && git fetch origin && git merge origin/main
git branch -d feature/auth-and-core
```

---

## Dependencies

| You depend on | Who |
|--------------|-----|
| Repo skeleton | Dev 1 — wait for initial push |

| Others depend on you | What |
|---------------------|------|
| Dev 1 (Workers) | Queue name constants, Mongo schema field names |
| Dev 3 (CRUD) | `Vault`, `Image` models; `s3.createPresignedPost()`, `s3.getPresignedGet()`, `getQueue()` |
| Dev 4 (Client query) | `QueryJob` model; `qdrantClient`; `verifyJWT` middleware; `auth.routes.js` |

**Ship `packages/core/` first** — it unblocks everyone.

---

## Env Vars You Need

```env
MONGODB_URI=mongodb://localhost:27017/graphpic
JWT_SECRET=supersecretkey
JWT_EXPIRY=7d
```
