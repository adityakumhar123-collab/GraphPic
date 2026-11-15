# Dev3_instruct.md — Photographer CRUD
## Role: Vault Management · Bulk Upload · Photographer Frontend

---

## Your Owned Files & Directories

```
graphpic/
├── apps/
│   ├── api/
│   │   └── src/
│   │       ├── lib/                       ← entire dir, yours
│   │       │   └── presignedUpload.js
│   │       ├── routes/
│   │       │   └── vault.routes.js        ← this file only, yours
│   │       └── controllers/
│   │           └── vault.controller.js    ← this file only, yours
│   └── web/
│       └── src/
│           ├── pages/
│           │   ├── Dashboard.jsx          ← this file only, yours
│           │   └── VaultDetail.jsx        ← this file only, yours
│           ├── components/
│           │   ├── Uploader.jsx           ← this file only, yours
│           │   └── QRDisplay.jsx          ← this file only, yours
│           └── hooks/
│               └── useUpload.js           ← this file only, yours
```

**You have exclusive ownership of `apps/api/src/lib/`.**  
In shared directories (`routes/`, `controllers/`, `pages/`, `components/`, `hooks/`) you each own specific named files — no overlaps.

---

## What Each File Does

### `apps/api/src/lib/presignedUpload.js`
The core of the bulk upload flow — called from `vault.controller.js`:

```js
// Input: { files: [{ filename, fileSize, mimeType }], vaultId, ownerId }
//
// For each file:
//   1. Generate imageId (new mongoose.Types.ObjectId())
//   2. Build s3_key = `${ownerId}/${vaultId}/${imageId}.${ext}`
//   3. Call s3.createPresignedPost(s3_key, mimeType, maxSizeMB=20)
//   4. INSERT images doc: { _id: imageId, vault_id, owner_id, s3_key,
//                           original_filename, file_size, mime_type,
//                           status: "pending", uploaded_at: now }
//
// After all files:
//   5. Vault.$inc({ image_count: files.length })
//
// Returns: [{ imageId, presignedUrl, fields }]
```

---

### `apps/api/src/routes/vault.routes.js`
All routes here require `verifyJWT` (imported from middleware — Dev 2 writes it):

```
GET    /vaults                 → vault.controller.listVaults
POST   /vaults                 → vault.controller.createVault
DELETE /vaults/:id             → vault.controller.deleteVault
PATCH  /vaults/:id/archive     → vault.controller.archiveVault
POST   /vaults/:id/upload-init → vault.controller.uploadInit
```

---

### `apps/api/src/controllers/vault.controller.js`

| Handler | Logic |
|---------|-------|
| `listVaults` | `Vault.find({ owner_id: req.user._id })` → return array |
| `createVault` | Validate `event_name` → insert vault `{ owner_id, event_name, status:"active", image_count:0 }` → push `{ event_name, vault_id }` into `user.vaults` → return vault |
| `deleteVault` | Verify ownership → delete vault doc → `Image.deleteMany({ vault_id })` → return 204 |
| `archiveVault` | Verify ownership → `Vault.findByIdAndUpdate(id, { status:"archived", updated_at:now })` → return vault |
| `uploadInit` | Verify ownership + `vault.status === "active"` → validate `req.body.files[]` → call `presignedUpload()` → return `{ presigned_urls }` |

---

### `apps/web/src/pages/Dashboard.jsx`
Photographer home after login:
- Fetch and render all vaults (`GET /vaults`)
- **Create vault** button → modal → `event_name` input → `POST /vaults`
- Vault cards: event name, image count, status badge (`active` / `archived`), link to VaultDetail
- Per-card actions: **Archive** (`PATCH /vaults/:id/archive`), **Delete** (`DELETE /vaults/:id`)
- Show QR icon per vault → opens QRDisplay modal

### `apps/web/src/pages/VaultDetail.jsx`
Single vault detail view:
- Vault name, status badge, image count
- Renders `<Uploader vaultId={vault._id} />` for bulk upload
- Renders `<QRDisplay vaultId={vault._id} />` — the client-facing link/QR
- Upload progress summary: X uploaded, Y processing, Z failed

### `apps/web/src/components/Uploader.jsx`
Drag-and-drop bulk file uploader:
- Accepts `image/*` only
- On drop/select → calls `useUpload(vaultId, files)`
- Shows per-file progress bar (idle → uploading → done / failed)
- Shows summary footer: N uploaded, M failed

### `apps/web/src/components/QRDisplay.jsx`
```jsx
// Props: vaultId (string)
// 1. Build clientUrl = `${import.meta.env.VITE_APP_URL}/clients/${vaultId}`
// 2. Render QR code with `react-qr-code` or `qrcode.react`
// 3. Show copy-link button alongside
// 4. Optionally a download QR as PNG button
```

### `apps/web/src/hooks/useUpload.js`
```js
// useUpload(vaultId)
// Called with: files (FileList or File[])
//
// Steps:
// 1. POST /vaults/:vaultId/upload-init  with { files: [{ filename, fileSize, mimeType }] }
// 2. Receive [{ imageId, presignedUrl, fields }]
// 3. For each presigned URL:
//    - PUT file directly to S3 using the presigned fields
//    - Concurrency cap: 5 parallel uploads (Promise.allSettled batching)
//    - Track per-file state: "idle" | "uploading" | "done" | "failed"
// 4. Return { fileStatuses, overallProgress (0–100) }
```

---

## Are the Files Exhaustive?

For your scope: **yes**, with these notes:

- You may want a small `apps/web/src/components/VaultCard.jsx` extracted from `Dashboard.jsx` for the individual vault card UI — this would be a new file in your component territory, totally fine.
- If input validation is complex, you can add a `apps/api/src/schemas/vault.schema.js` for Zod schemas — a new file, acceptable.
- For image delete (cascading Qdrant cleanup), that's a future concern (vectors in Qdrant should also be deleted by `vault_id` filter) — not required in v1 but keep a TODO in `deleteVault`.
- No additional files strictly needed.

---

## Git & GitHub Workflow

### Step 1 — Clone after Dev 1 pushes skeleton

```bash
git clone https://github.com/<dev1-username>/graphpic.git
cd graphpic
pnpm install
```

### Step 2 — Coordinate with Dev 2 early

You need these from Dev 2 (`packages/core/`) before or while you code:
- `Vault` model — schema shape (especially field names)
- `Image` model — fields you insert in `presignedUpload.js`
- `s3.createPresignedPost()` — function signature
- `verifyJWT` — import path `../../middleware/auth.middleware`

**Work in parallel using agreed interfaces.** Use placeholder imports and swap when Dev 2's branch merges.

### Step 3 — Create your feature branch

```bash
git checkout -b feature/vault-crud-and-upload
```

### Step 4 — Commit as you go

```bash
git add apps/api/src/lib/presignedUpload.js
git add apps/api/src/routes/vault.routes.js
git add apps/api/src/controllers/vault.controller.js
git add apps/web/src/pages/Dashboard.jsx
git add apps/web/src/pages/VaultDetail.jsx
git add apps/web/src/components/Uploader.jsx
git add apps/web/src/components/QRDisplay.jsx
git add apps/web/src/hooks/useUpload.js

git commit -m "feat(vault): list, create, archive, delete vault endpoints"
git commit -m "feat(upload): presigned URL batch generation and image doc insert"
git commit -m "feat(web): Dashboard and VaultDetail photographer pages"
git commit -m "feat(web): Uploader component with direct S3 PUT and progress tracking"
git commit -m "feat(web): QRDisplay component"
```

### Step 5 — Stay in sync

```bash
git fetch origin
git merge origin/main
# Your files are distinct — conflicts are extremely unlikely
```

### Step 6 — Push and open PR

```bash
git push origin feature/vault-crud-and-upload
```

On GitHub:
1. **Compare & pull request** → `feature/vault-crud-and-upload → main`
2. Title: `feat: vault CRUD, bulk upload init, photographer frontend`
3. Note in description: depends on Dev 2's core models and middleware
4. Request review from Dev 1; do not self-merge

### Step 7 — After merge

```bash
git checkout main && git fetch origin && git merge origin/main
git branch -d feature/vault-crud-and-upload
```

---

## Dependencies

| You depend on | Who owns it |
|--------------|-------------|
| `Vault`, `Image` Mongoose models | Dev 2 — `packages/core/db/models/` |
| `s3.createPresignedPost()`, `s3.getPresignedGet()` | Dev 2 — `packages/core/services/s3.js` |
| `getQueue()` | Dev 2 — not directly needed (upload → S3 event → poller → queue, you don't push) |
| `verifyJWT` middleware | Dev 2 — `apps/api/src/middleware/auth.middleware.js` |

| Others depend on you | What they need |
|---------------------|----------------|
| Dev 4 | Nothing directly — your vault routes are mounted by Dev 4's `routes/index.js` |

---

## Env Vars You Need

Already in `.env.example`:
```env
S3_BUCKET_NAME=
AWS_REGION=
VITE_API_URL=http://localhost:3000
VITE_APP_URL=http://localhost:5173
```
