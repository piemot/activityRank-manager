# syntax=docker/dockerfile:1

ARG NODE_VERSION=20.17
ARG APP_DIRNAME=manager
ARG APP_PKG_NAME=@activityrank/manager

FROM node:${NODE_VERSION}-slim AS pruner
ARG APP_PKG_NAME
WORKDIR /app
COPY . .
RUN npx turbo prune --scope=$APP_PKG_NAME --docker

FROM node:${NODE_VERSION}-slim AS base
# RUN apt-get update && apt-get install -y openssl
# Always run with `tini`
ENV TINI_VERSION=v0.19.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /tini
RUN chmod +x /tini
ENTRYPOINT ["/tini", "--"]

RUN corepack enable
WORKDIR /app
COPY .gitignore .gitignore
COPY --from=pruner /app/out/json/ .
COPY --from=pruner /app/out/pnpm-lock.yaml ./pnpm-lock.yaml
COPY --from=pruner /app/out/pnpm-workspace.yaml ./pnpm-workspace.yaml

FROM base AS dev-deps
WORKDIR /app
RUN --mount=type=cache,id=pnpm,target=/pnpm/store pnpm install --frozen-lockfile

FROM base AS production-deps
WORKDIR /app
ENV NODE_ENV=production
RUN --mount=type=cache,id=pnpm,target=/pnpm/store pnpm install --prod --frozen-lockfile

FROM base AS builder
ARG APP_PKG_NAME
WORKDIR /app
RUN corepack enable
ENV NODE_ENV=production
COPY --from=pruner /app/out/full/ .
COPY --from=dev-deps /app/ .
COPY turbo.json turbo.json
RUN pnpm run --filter=$APP_PKG_NAME... build 
# Delete node_modules; they should be copied from  production-deps
RUN find . -name "node_modules" -type d -prune -exec rm -r '{}' +

FROM base AS runner
ARG APP_DIRNAME

ENV NODE_ENV=production

RUN addgroup --system --gid 1001 arusr
RUN adduser --system --uid 1001 arusr
# Switch to a non-root user for better security.
USER arusr

COPY --from=builder --chown=arusr:arusr /app .
COPY --from=production-deps --chown=arusr:arusr /app .

ENV PORT=3000
EXPOSE 3000

CMD ["node", "./apps/manager/dist/app.js"]
