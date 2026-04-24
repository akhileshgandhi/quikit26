# syntax=docker/dockerfile:1.7
#
# QuikScale — flagship OKR / KPI app
# Port: 3004
#
# Build from the monorepo ROOT:
#   docker build -f apps/quikscale/Dockerfile -t gcr.io/<project>/quikscale:<tag> .
#
# See docs/engineering/GCP_DEPLOYMENT.md for the full deployment guide.

FROM node:20-slim AS base
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates openssl libssl-dev \
  && rm -rf /var/lib/apt/lists/*
ENV NEXT_TELEMETRY_DISABLED=1

# ------------------------------------------------------------------ deps
FROM base AS deps
WORKDIR /app
COPY package.json package-lock.json turbo.json ./
COPY apps/quikit/package.json       ./apps/quikit/
COPY apps/quikscale/package.json    ./apps/quikscale/
COPY apps/admin/package.json        ./apps/admin/
COPY packages/ui/package.json       ./packages/ui/
COPY packages/auth/package.json     ./packages/auth/
COPY packages/shared/package.json   ./packages/shared/
COPY packages/database/package.json ./packages/database/
COPY packages/redis/package.json    ./packages/redis/
COPY packages/database/prisma ./packages/database/prisma
RUN npm ci --no-audit --no-fund

# ------------------------------------------------------------------ builder
FROM base AS builder
WORKDIR /app

# Copy manifests for npm ci
COPY package.json package-lock.json turbo.json ./
COPY apps/quikit/package.json       ./apps/quikit/
COPY apps/quikscale/package.json    ./apps/quikscale/
COPY apps/admin/package.json        ./apps/admin/
COPY packages/ui/package.json       ./packages/ui/
COPY packages/auth/package.json     ./packages/auth/
COPY packages/shared/package.json   ./packages/shared/
COPY packages/database/package.json ./packages/database/
COPY packages/redis/package.json    ./packages/redis/
COPY packages/database/prisma ./packages/database/prisma

# Reinstall dependencies to ensure all are available in this stage
RUN npm ci --no-audit --no-fund

# Copy source (everything not excluded by .dockerignore)
COPY . .

# Create empty public directories if they don't exist (apps may not have public assets)
RUN mkdir -p apps/admin/public apps/quikit/public apps/quikscale/public

# Regenerate Prisma client into the re-copied node_modules (postinstall
# ran in deps/, but source has since been overlaid). Idempotent.
RUN npx prisma generate --schema=packages/database/prisma/schema.prisma
ARG DATABASE_URL="postgresql://dummy:dummy@localhost:5432/dummy"
ARG NEXTAUTH_URL="http://localhost:3004"
ARG NEXT_PUBLIC_SENTRY_DSN=""
ENV DATABASE_URL=$DATABASE_URL
ENV NEXTAUTH_URL=$NEXTAUTH_URL
ENV NEXT_PUBLIC_SENTRY_DSN=$NEXT_PUBLIC_SENTRY_DSN
RUN npx turbo build --filter=quikscale

# ------------------------------------------------------------------ runner
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV HOSTNAME=0.0.0.0
ENV PORT=3004
RUN addgroup --system --gid 1001 nodejs \
  && adduser  --system --uid 1001 nextjs
COPY --from=builder --chown=nextjs:nodejs /app/apps/quikscale/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/apps/quikscale/.next/static ./apps/quikscale/.next/static
COPY --from=builder --chown=nextjs:nodejs /app/apps/quikscale/public ./apps/quikscale/public
COPY --from=builder --chown=nextjs:nodejs /app/node_modules/.prisma ./node_modules/.prisma
COPY --from=builder --chown=nextjs:nodejs /app/node_modules/@prisma/client ./node_modules/@prisma/client
COPY --from=builder --chown=nextjs:nodejs /app/packages/database ./packages/database
USER nextjs
EXPOSE 3004
CMD ["sh", "-c", "if [ -f apps/quikscale/server.js ]; then node apps/quikscale/server.js; else node server.js; fi"]
