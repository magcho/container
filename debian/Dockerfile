FROM node:16.13.0-slim@sha256:9ec1ff69c844f2de3a6a2180cd49ca75797d9f2a0fc52bb33c8a672fd0fe7e18 AS base

FROM base AS builder
# Build arguments to change source url, branch or tag
ARG CODIMD_REPOSITORY
ARG HEDGEDOC_REPOSITORY=https://github.com/hedgedoc/hedgedoc.git
ARG VERSION=master
RUN if [ -n "${CODIMD_REPOSITORY}" ]; then echo "CODIMD_REPOSITORY is deprecated. Please use HEDGEDOC_REPOSITORY instead" && exit 1; fi

# Clone the source and remove git repository but keep the HEAD file
RUN apt-get update && apt-get install --no-install-recommends -y git jq ca-certificates
RUN git clone --depth 1 --branch "$VERSION" "$HEDGEDOC_REPOSITORY" /hedgedoc
RUN git -C /hedgedoc log --pretty=format:'%ad %h %d' --abbrev-commit --date=short -1
RUN git -C /hedgedoc rev-parse HEAD > /tmp/gitref
RUN rm -rf /hedgedoc/.git/*
RUN mv /tmp/gitref /hedgedoc/.git/HEAD
RUN jq ".repository.url = \"${HEDGEDOC_REPOSITORY}\"" /hedgedoc/package.json > /hedgedoc/package.new.json
RUN mv /hedgedoc/package.new.json /hedgedoc/package.json

# Install app dependencies and build
WORKDIR /hedgedoc
RUN yarn install --production=false --pure-lockfile
RUN yarn run build
RUN yarn install --production=true --pure-lockfile


FROM base
ARG UID=10000
ENV NODE_ENV=production

RUN apt-get update && \
    apt-get install --no-install-recommends -y gosu && \
    rm -r /var/lib/apt/lists/*

COPY --chown=$UID --from=builder /hedgedoc /hedgedoc

# Add configuraton files
COPY ["resources/config.json", "/files/"]

# Healthcheck
COPY --chown=$UID /resources/healthcheck.mjs /hedgedoc/healthcheck.mjs
HEALTHCHECK --interval=5s CMD node healthcheck.mjs

# For backwards compatibility
RUN ln -s /hedgedoc /codimd

# Symlink configuration files
RUN rm -f /hedgedoc/config.json
RUN ln -s /files/config.json /hedgedoc/config.json

# Create hedgedoc user
RUN adduser --uid $UID --home /hedgedoc/ --disabled-password --system hedgedoc

WORKDIR /hedgedoc
EXPOSE 3000

COPY ["resources/docker-entrypoint.sh", "/usr/local/bin/docker-entrypoint.sh"]

ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

CMD ["node", "app.js"]
