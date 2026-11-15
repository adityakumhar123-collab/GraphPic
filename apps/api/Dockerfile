FROM node:20-alpine
WORKDIR /app

# Copy workspace manifests first (better layer caching)
COPY package.json pnpm-workspace.yaml ./
COPY packages/core/package.json ./packages/core/
COPY packages/shared-types/package.json ./packages/shared-types/
COPY apps/api/package.json ./apps/api/

# Install pnpm and deps
RUN npm install -g pnpm && pnpm install --frozen-lockfile

# Copy source
COPY packages/ ./packages/
COPY apps/api/ ./apps/api/

EXPOSE 3000
CMD ["node", "apps/api/src/index.js"]
