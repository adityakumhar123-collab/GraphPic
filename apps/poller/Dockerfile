FROM node:20-alpine
WORKDIR /app

COPY package.json pnpm-workspace.yaml ./
COPY packages/core/package.json ./packages/core/
COPY packages/shared-types/package.json ./packages/shared-types/
COPY apps/poller/package.json ./apps/poller/

RUN npm install -g pnpm && pnpm install --frozen-lockfile

COPY packages/ ./packages/
COPY apps/poller/ ./apps/poller/

CMD ["node", "apps/poller/src/index.js"]
