# Dev4_instruct.md — Client Query Pipeline + API Wiring
## Role: Selfie → Job → Results Pipeline · API Entry & Router · Client Frontend

---

## Your Owned Files & Directories

```
graphpic/
├── apps/
│   ├── api/
│   │   └── src/
│   │       ├── index.js                    ← this file, yours (API entry point)
│   │       ├── app.js                      ← this file, yours (Express factory + route mounting)
│   │       ├── routes/
│   │       │   ├── index.js                ← this file, yours (mounts ALL routers)
│   │       │   └── client.routes.js        ← this file, yours
│   │       └── controllers/
│   │           └── client.controller.js    ← this file, yours
│   └── web/
│       └── src/
│           ├── main.jsx                    ← this file, yours
│           ├── App.jsx                     ← this file, yours (react-router setup)
│           ├── pages/
│           │   └── ClientView.jsx          ← this file, yours
│           ├── components/
│           │   ├── FaceCapture.jsx         ← this file, yours
│           │   └── ResultsGallery.jsx      ← this file, yours
│           └── hooks/
│               └── useJobPoller.js         ← this file, yours
```

**You own the two API app-level files (`index.js`, `app.js`) and `routes/index.js`.**  
This makes you the **API integrator** — your `routes/index.js` mounts all three router files from all devs.

---

## What Each File Does

### `apps/api/src/index.js`
Application entry point:
```js
// 1. Load env vars (dotenv.config())
// 2. connectMongo(process.env.MONGODB_URI)  ← packages/core/db
// 3. const app = require('./app')
// 4. app.listen(process.env.PORT || 3000, () => console.log('API running'))
```

### `apps/api/src/app.js`
Express app factory:
```js
// 1. const app = express()
// 2. Middleware: cors(), express.json(), morgan('dev')
// 3. Mount root router: app.use('/api', require('./routes'))
// 4. Mount errorHandler (Dev 2's) as LAST middleware: app.use(errorHandler)
// 5. module.exports = app
```

### `apps/api/src/routes/index.js`
Central route mount — the only file that imports all three routers:
```js
const authRoutes   = require('./auth.routes')    // Dev 2
const vaultRoutes  = require('./vault.routes')   // Dev 3
const clientRoutes = require('./client.routes')  // yours

router.use('/auth',    authRoutes)
router.use('/vaults',  vaultRoutes)   // protected by verifyJWT inside vault.routes
router.use('/clients', clientRoutes)  // public
router.use('/jobs',    clientRoutes)  // polling endpoint lives on same controller
```

---

### `apps/api/src/routes/client.routes.js`
No auth middleware — these routes are fully public (clients have no accounts):
```
POST /clients/:vault_id/query  → client.controller.submitQuery
GET  /jobs/:job_id             → client.controller.pollJob
```

---

### `apps/api/src/controllers/client.controller.js`

**`submitQuery(req, res)`**
```
1. Look up vault by params.vault_id (Vault model)
2. If not found → 404
3. If vault.status === "archived" → 403 { error: "vault_archived" }
4. Upload raw selfie blob server-side (multer middleware, memory storage):
     s3_key = `temp/${vault.owner_id}/${vault_id}/selfies/${job_id}.jpg`
     Use s3.uploadBuffer(key, buffer, mimeType)  ← add this helper to s3.js
       OR use a presigned POST + server forwards — simpler: direct server upload for one image
5. INSERT query_jobs {
     vault_id,
     status: "pending",
     created_at: now,
     expires_at: new Date(now + 10 * 60 * 1000)
   }
6. Push to face-query BullMQ queue:
     getQueue(FACE_QUERY_QUEUE).add('query', { job_id, vault_id, selfie_s3_key: s3_key })
7. Return 200 { job_id }
```

**`pollJob(req, res)`**
```
1. QueryJob.findById(params.job_id)
2. If not found → 404
3. Return { status, result_presigned_urls, error }
   UI interprets:
   "pending" / "processing" → keep polling (every 2s)
   "done"    → show ResultsGallery with result_presigned_urls
   "failed"  → check error:
       "no_face_detected"  → show "No face found — please try again" prompt
       "processing_error"  → show generic error message
```

> **Note:** You may need to ask Dev 2 to add an `uploadBuffer(key, buffer, mimeType)` helper to `packages/core/services/s3.js` for the selfie server-side upload — coordinate this.

---

### `apps/web/src/main.jsx`
Standard React entry:
```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode><App /></React.StrictMode>
)
```

### `apps/web/src/App.jsx`
React Router v6 setup with a `ProtectedRoute` wrapper:
```jsx
// Routes:
//   /auth              → <Auth />         (Dev 2) — public
//   /                  → <Dashboard />    (Dev 3) — protected
//   /vaults/:id        → <VaultDetail />  (Dev 3) — protected
//   /clients/:vault_id → <ClientView />   (yours) — public

// ProtectedRoute: reads token from Redux store (Dev 2's store)
//   if no token → <Navigate to="/auth" />
//   otherwise   → render children
```

---

### `apps/web/src/pages/ClientView.jsx`
Public client-facing page — the full flow:
1. Show `<FaceCapture onCapture={handleCapture} />`
2. `handleCapture(blob)`:
   - POST blob to `/api/clients/:vault_id/query` (multipart)
   - Receive `{ job_id }`
   - Start polling via `useJobPoller(job_id)`
3. Show spinner + "Searching your photos…" while polling
4. On `status: "done"` → render `<ResultsGallery urls={result_presigned_urls} />`
5. On `status: "failed"`:
   - `error === "no_face_detected"` → "We couldn't detect a face in your photo. Please try again." + retake button
   - other → "Something went wrong. Please try again."

### `apps/web/src/components/FaceCapture.jsx`
```jsx
// 1. navigator.mediaDevices.getUserMedia({ video: { facingMode: "user" } })
// 2. Render <video> preview (auto-play, muted)
// 3. "Capture" button → draw video frame to <canvas> → canvas.toBlob(blob => onCapture(blob))
// 4. Show retake button after capture (re-starts preview)
// 5. Handle permission denied gracefully with fallback message
```

### `apps/web/src/components/ResultsGallery.jsx`
```jsx
// Props: urls: string[]
// If urls.length === 0 → "No matching photos found in this event."
// Otherwise: responsive CSS grid of <img> cards
// Each card: image preview + download anchor (href=url, download attr)
// Images load from S3 presigned GET URLs (10-min expiry)
```

### `apps/web/src/hooks/useJobPoller.js`
```js
// useJobPoller(jobId)
// 1. useEffect: setInterval(poll, 2000)
// 2. poll = async () => {
//      const data = await apiClient(`/jobs/${jobId}`)
//      if (data.status === "done" || data.status === "failed") {
//        clearInterval(intervalId)
//        setResult(data)
//      }
//    }
// 3. Cleanup: clearInterval on unmount
// Returns: { status, urls, error, isLoading }
```

---

## Are the Files Exhaustive?

For your scope: **yes**, with these notes:

- You may need a `multer` configuration either inline in `client.controller.js` or extracted to `apps/api/src/lib/multerConfig.js` — the latter would be a new file in `lib/` which is **Dev 3's directory**. Best to put multer config inline in your controller or ask Dev 3 to leave `lib/` open for one more file.
- If you add a `ProtectedRoute` component in React, it can live inline in `App.jsx` or be extracted to `apps/web/src/components/ProtectedRoute.jsx` — your call.
- `apps/web/src/App.jsx` imports pages from Dev 2 and Dev 3 — these are just imports, no file ownership conflict.
- No additional files strictly needed.

---

## Git & GitHub Workflow

### Step 1 — Clone after Dev 1 pushes skeleton

```bash
git clone https://github.com/<dev1-username>/graphpic.git
cd graphpic
pnpm install
```

### Step 2 — Understand your position: you integrate last

Your `app.js` and `routes/index.js` import from all other devs. The clean approach:

- **Write your own files first** (client.routes.js, client.controller.js, ClientView.jsx, FaceCapture.jsx, ResultsGallery.jsx, useJobPoller.js, App.jsx, main.jsx) — these have minimal cross-dev dependencies.
- **Stub the cross-dev imports** in `routes/index.js` and `app.js` while other branches are still open:
  ```js
  // routes/index.js — stubs while teammates work
  const authRoutes  = require('./auth.routes')   // Dev 2 — will exist after their merge
  const vaultRoutes = require('./vault.routes')  // Dev 3 — will exist after their merge
  ```
- **Finalise wiring** after Dev 2 and Dev 3 merge their PRs → merge main into your branch → verify imports resolve.

### Step 3 — Create your feature branch

```bash
git checkout -b feature/client-query-pipeline
```

### Step 4 — Commit as you go

```bash
git add apps/api/src/index.js
git add apps/api/src/app.js
git add apps/api/src/routes/index.js
git add apps/api/src/routes/client.routes.js
git add apps/api/src/controllers/client.controller.js
git add apps/web/src/main.jsx
git add apps/web/src/App.jsx
git add apps/web/src/pages/ClientView.jsx
git add apps/web/src/components/FaceCapture.jsx
git add apps/web/src/components/ResultsGallery.jsx
git add apps/web/src/hooks/useJobPoller.js

git commit -m "feat(client): selfie submission and query job endpoints"
git commit -m "feat(api): Express entry point and central route mounting"
git commit -m "feat(web): ClientView page with camera capture and job polling"
git commit -m "feat(web): FaceCapture component with getUserMedia"
git commit -m "feat(web): ResultsGallery and useJobPoller hook"
git commit -m "feat(web): App.jsx with react-router and ProtectedRoute"
```

### Step 5 — Stay in sync and finalise wiring

```bash
git fetch origin
git merge origin/main
# After Dev 2 and Dev 3 merge: verify that imports in routes/index.js resolve correctly
# Run the app locally to confirm: pnpm --filter @graphpic/api dev
```

### Step 6 — Push and open PR

```bash
git push origin feature/client-query-pipeline
```

On GitHub:
1. **Compare & pull request** → `feature/client-query-pipeline → main`
2. Title: `feat: client query pipeline, selfie capture, polling UI, API wiring`
3. **Important note in description:** "This PR should be merged after Dev 2 and Dev 3 — `routes/index.js` mounts all routers."
4. Request review from Dev 1; do not self-merge

### Step 7 — After merge

```bash
git checkout main && git fetch origin && git merge origin/main
git branch -d feature/client-query-pipeline
```

---

## Dependencies

| You depend on | Who owns it | What you need |
|--------------|-------------|---------------|
| `Vault` model (vault lookup) | Dev 2 — `packages/core/db/models/Vault.js` | `status` field check |
| `QueryJob` model | Dev 2 — `packages/core/db/models/QueryJob.js` | insert + poll |
| `s3.js` (selfie upload) | Dev 2 — `packages/core/services/s3.js` | May need `uploadBuffer()` added |
| `getQueue`, `FACE_QUERY_QUEUE` | Dev 2 — `packages/core/services/queue.js` | Push job to queue |
| `verifyJWT` middleware | Dev 2 — middleware (for `app.js` protected routes) | Import path |
| `errorHandler` | Dev 2 — middleware | Mount last in `app.js` |
| `auth.routes.js` | Dev 2 | Import in `routes/index.js` |
| `vault.routes.js` | Dev 3 | Import in `routes/index.js` |

| Others depend on you | What they need |
|---------------------|----------------|
| Nobody — you integrate last | Your `app.js` ties everything together |

**Recommended merge order:**
```
Dev 1 (skeleton) → Dev 2 (core + auth) → Dev 3 (vault CRUD) → Dev 4 (you — wiring + client)
```

---

## Env Vars You Need

Already in `.env.example`:
```env
PORT=3000
MONGODB_URI=
REDIS_URL=
S3_BUCKET_NAME=
AWS_REGION=
VITE_API_URL=http://localhost:3000
VITE_APP_URL=http://localhost:5173
```
