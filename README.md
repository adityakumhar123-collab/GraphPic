# Graphpic

> Face-recognition photo retrieval for photographers and their clients.

## What it does

Photographers bulk-upload event photos. Clients scan a QR code, take a selfie, and instantly get back every photo from that event in which their face appears.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18 + Vite |
| API | Node.js / Express 4 |
| Database | MongoDB (Mongoose) |
| Vector DB | Qdrant |
| Object Storage | Amazon S3 |
| Queue | BullMQ + Redis |
| Message Bridge | AWS SQS → Redis Poller |
| Face ML | InsightFace + ArcFace (Python) |

---

## Local Development Setup

### Prerequisites
- Node.js 20+
- pnpm (`npm i -g pnpm`)
- Python 3.11+
- Docker + Docker Compose

### 1. Clone and install JS dependencies

```bash
git clone https://github.com/<owner>/graphpic.git
cd graphpic
pnpm install
```

### 2. Set up environment variables

```bash
cp .env.example .env
# Fill in your AWS credentials, MongoDB URI, Redis URL, JWT secret
```

### 3. Start infrastructure (MongoDB, Redis, Qdrant)

```bash
docker compose up mongodb redis qdrant -d
```

### 4. Install Python dependencies

```bash
cd apps/workers
pip install -r image_worker/requirements.txt
```

### 5. Start all JS services

```bash
# From repo root — starts API, web, and poller concurrently
pnpm dev
```

### 6. Start Python workers (separate terminal)

```bash
python apps/workers/image_worker/worker.py &
python apps/workers/query_worker/worker.py &
```

---

## Project Structure

See [`manuals/instructions/r1_about.md`](manuals/instructions/r1_about.md) for the full architecture.  
See `manuals/instructions/Dev{1-4}_instruct.md` for team work assignments.

---

## Team

| Dev | Role |
|-----|------|
| Dev 1 (Owner) | Infra, Python Workers, Poller |
| Dev 2 | Auth (JWT), Shared Core (models + services) |
| Dev 3 | Photographer CRUD, Bulk Upload, Photographer UI |
| Dev 4 | Client Query Pipeline, API Wiring, Client UI |
