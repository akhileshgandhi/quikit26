# # syntax=docker/dockerfile:1.7
# #
# # Admin — per-tenant admin UI
# # Port: 3005
# #
# # Build from the monorepo ROOT:
# #   docker build -f apps/admin/Dockerfile -t gcr.io/<project>/admin:<tag> .
# #
# # See docs/engineering/GCP_DEPLOYMENT.md for the full deployment guide.

# FROM node:20-slim AS base
# RUN apt-get update && apt-get install -y --no-install-recommends \
#       ca-certificates openssl libssl-dev \
#   && rm -rf /var/lib/apt/lists/*
# ENV NEXT_TELEMETRY_DISABLED=1

# # ------------------------------------------------------------------ deps
# FROM base AS deps
# WORKDIR /app
# COPY package.json package-lock.json turbo.json ./
# COPY apps/quikit/package.json       ./apps/quikit/
# COPY apps/quikscale/package.json    ./apps/quikscale/
# COPY apps/admin/package.json        ./apps/admin/
# COPY packages/ui/package.json       ./packages/ui/
# COPY packages/auth/package.json     ./packages/auth/
# COPY packages/shared/package.json   ./packages/shared/
# COPY packages/database/package.json ./packages/database/
# COPY packages/redis/package.json    ./packages/redis/
# COPY packages/database/prisma ./packages/database/prisma
# RUN npm ci --no-audit --no-fund

# # ------------------------------------------------------------------ builder
# FROM base AS builder
# WORKDIR /app

# # Copy manifests for npm ci
# COPY package.json package-lock.json turbo.json ./
# COPY apps/quikit/package.json       ./apps/quikit/
# COPY apps/quikscale/package.json    ./apps/quikscale/
# COPY apps/admin/package.json        ./apps/admin/
# COPY packages/ui/package.json       ./packages/ui/
# COPY packages/auth/package.json     ./packages/auth/
# COPY packages/shared/package.json   ./packages/shared/
# COPY packages/database/package.json ./packages/database/
# COPY packages/redis/package.json    ./packages/redis/
# COPY packages/database/prisma ./packages/database/prisma

# # Reinstall dependencies to ensure all are available in this stage
# RUN npm ci --no-audit --no-fund

# # Copy source (everything not excluded by .dockerignore)
# COPY . .

# # Create empty public directories if they don't exist (apps may not have public assets)
# RUN mkdir -p apps/admin/public apps/quikit/public apps/quikscale/public

# # Regenerate Prisma client into the re-copied node_modules (postinstall
# # ran in deps/, but source has since been overlaid). Idempotent.
# RUN npx prisma generate --schema=packages/database/prisma/schema.prisma
# ARG DATABASE_URL="postgresql://dummy:dummy@localhost:5432/dummy"
# ARG NEXTAUTH_URL="http://localhost:3005"
# ARG NEXT_PUBLIC_SENTRY_DSN=""
# ENV DATABASE_URL=$DATABASE_URL
# ENV NEXTAUTH_URL=$NEXTAUTH_URL
# ENV NEXT_PUBLIC_SENTRY_DSN=$NEXT_PUBLIC_SENTRY_DSN
# RUN npx turbo build --filter=admin

# # ------------------------------------------------------------------ runner
# FROM base AS runner
# WORKDIR /app
# ENV NODE_ENV=production
# ENV HOSTNAME=0.0.0.0
# ENV PORT=3005
# RUN addgroup --system --gid 1001 nodejs \
#   && adduser  --system --uid 1001 nextjs
# COPY --from=builder --chown=nextjs:nodejs /app/apps/admin/.next/standalone ./
# COPY --from=builder --chown=nextjs:nodejs /app/apps/admin/.next/static ./apps/admin/.next/static
# COPY --from=builder --chown=nextjs:nodejs /app/apps/admin/public ./apps/admin/public

# COPY --from=builder --chown=nextjs:nodejs /app/node_modules ./node_modules

# COPY --from=builder --chown=nextjs:nodejs /app/node_modules/.prisma ./node_modules/.prisma
# COPY --from=builder --chown=nextjs:nodejs /app/node_modules/@prisma/client ./node_modules/@prisma/client
# USER nextjs
# EXPOSE 3005
# CMD ["node", "server.js"]
# syntax=docker/dockerfile:1.7

# -------------------------------------------------- base
FROM node:20-slim AS base
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates openssl libssl-dev \
  && rm -rf /var/lib/apt/lists/*
ENV NEXT_TELEMETRY_DISABLED=1

# -------------------------------------------------- deps
FROM base AS deps
WORKDIR /app

# Copy only manifests for caching
COPY package.json package-lock.json turbo.json ./
COPY apps/admin/package.json ./apps/admin/
COPY apps/quikit/package.json ./apps/quikit/
COPY apps/quikscale/package.json ./apps/quikscale/
COPY packages/ui/package.json ./packages/ui/
COPY packages/auth/package.json ./packages/auth/
COPY packages/shared/package.json ./packages/shared/
COPY packages/database/package.json ./packages/database/
COPY packages/redis/package.json ./packages/redis/
COPY packages/database/prisma ./packages/database/prisma

# Install all deps
RUN npm ci --no-audit --no-fund

# -------------------------------------------------- builder
FROM base AS builder
WORKDIR /app

# Copy installed deps
COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/package.json ./package.json
COPY --from=deps /app/package-lock.json ./package-lock.json
COPY --from=deps /app/turbo.json ./turbo.json

# Copy full source
COPY . .

# Ensure public dirs exist
RUN mkdir -p apps/admin/public

# Prisma generate
RUN npx prisma generate --schema=packages/database/prisma/schema.prisma

# Build envs (safe defaults)
ARG DATABASE_URL="postgresql://dummy:dummy@localhost:5432/dummy"
ARG NEXTAUTH_URL="http://localhost:3005"
ARG NEXT_PUBLIC_SENTRY_DSN=""
ENV DATABASE_URL=$DATABASE_URL
ENV NEXTAUTH_URL=$NEXTAUTH_URL
ENV NEXT_PUBLIC_SENTRY_DSN=$NEXT_PUBLIC_SENTRY_DSN

# Build only admin app
RUN npx turbo build --filter=admin

# -------------------------------------------------- runner
# -------------------------------------------------- runner
FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV HOSTNAME=0.0.0.0
ENV PORT=3005

RUN addgroup --system --gid 1001 nodejs \
  && adduser --system --uid 1001 nextjs

COPY --from=builder --chown=nextjs:nodejs /app/apps/admin/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/apps/admin/.next/static ./apps/admin/.next/static
COPY --from=builder --chown=nextjs:nodejs /app/apps/admin/public ./apps/admin/public

COPY --from=builder --chown=nextjs:nodejs /app/node_modules/.prisma ./node_modules/.prisma
COPY --from=builder --chown=nextjs:nodejs /app/node_modules/@prisma/client ./node_modules/@prisma/client
COPY --from=builder --chown=nextjs:nodejs /app/packages/database ./packages/database

USER nextjs
EXPOSE 3005
CMD ["sh", "-c", "if [ -f apps/admin/server.js ]; then node apps/admin/server.js; else node server.js; fi"]