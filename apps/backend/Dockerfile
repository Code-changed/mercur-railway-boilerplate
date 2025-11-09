FROM node:20-alpine AS base

# The web Dockerfile is copy-pasted into our main docs at /docs/handbook/deploying-with-docker.
# Make sure you update this Dockerfile, the Dockerfile in the web workspace and copy that over to Dockerfile in the docs.

FROM base AS builder
# Check https://github.com/nodejs/docker-node/tree/b4117f9333da4138b03a546ec926ef50a31506c3#nodealpine to understand why libc6-compat might be needed.
RUN apk update
RUN apk add --no-cache libc6-compat
# Set working directory
WORKDIR /app

# Install pnpm
RUN corepack enable && corepack prepare pnpm@latest --activate

COPY package.json pnpm-workspace.yaml pnpm-lock.yaml ./
COPY packages ./packages
COPY apps/backend/package.json ./apps/backend/

# Install dependencies to get turbo
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm turbo prune api --docker

# Add lockfile and package.json's of isolated subworkspace
FROM base AS installer
RUN apk update
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Install pnpm
RUN corepack enable && corepack prepare pnpm@latest --activate

# First install dependencies (as they change less often)
COPY --from=builder /app/out/pnpm-lock.yaml ./pnpm-lock.yaml
COPY --from=builder /app/out/pnpm-workspace.yaml ./pnpm-workspace.yaml
COPY --from=builder /app/out/json/ .
RUN pnpm install --frozen-lockfile

# Build the project and its dependencies
COPY --from=builder /app/out/full/ .

# Uncomment and use build args to enable remote caching
# ARG TURBO_TEAM
# ENV TURBO_TEAM=$TURBO_TEAM

# ARG TURBO_TOKEN
# ENV TURBO_TOKEN=$TURBO_TOKEN

RUN pnpm turbo build --filter=api

FROM base AS runner
WORKDIR /app

# Don't run production as root
RUN addgroup --system --gid 1001 medusa
RUN adduser --system --uid 1001 medusa

# Ensure the medusa user owns the backend folder
RUN mkdir -p /app/apps/backend/static
RUN chown -R medusa:medusa /app/apps/backend

USER medusa
COPY --from=installer /app .

WORKDIR /app/apps/backend

# Don't run migrations in Dockerfile - run them as a separate Railway deployment
# RUN pnpm db:migrate

CMD ["pnpm", "start"]
