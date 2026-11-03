ARG RUBY_VERSION="4.0.9"
ARG DEBIAN_BASE="trixie"
# Add SBOM scan context for intermediate steps
ARG BUILDKIT_SBOM_SCAN_CONTEXT=true
ARG BUILDKIT_SBOM_SCAN_STAGE=true
FROM ruby:${RUBY_VERSION}-slim-trixie AS runtime-base
LABEL maintainer="operations@openproject.com"

ARG NODE_VERSION="24.22.1"
# npm bundled with node lags behind on patches for its vendored deps (e.g. tar)
ARG NPM_VERSION="11.19.0"
ARG BIM_SUPPORT=true
ENV USE_JEMALLOC=false
ENV DEBIAN_FRONTEND=noninteractive
ENV BUNDLE_JOBS=8
ENV BUNDLE_RETRY=3
ENV BUNDLE_WITHOUT="development:test"

# SYSTEM
ENV DOCKER=1
ENV APP_USER=app
ENV APP_PATH=/app
ENV APP_DATA_PATH=/var/openproject/assets
ENV BUNDLE_PATH="$APP_PATH/vendor/bundle"
ENV BUNDLE_APP_CONFIG="$APP_PATH/.bundle"
ENV PGVERSION="17"
ENV PGVERSION_CHOICES="13 15 17"
ENV PGBIN="/usr/lib/postgresql/$PGVERSION/bin"
ENV PATH="$PGBIN:$PATH"
ENV BUNDLE_WITHOUT="development:test"

# RAILS
# Set a default key base, ensure to provide a secure value in production environments!
ENV SECRET_KEY_BASE=OVERWRITE_ME
ENV RAILS_ENV=production
ENV RAILS_LOG_TO_STDOUT=1
ENV RAILS_SERVE_STATIC_FILES=1

# OPENPROJECT
# Valid values are: standard,bim
ENV OPENPROJECT_EDITION=standard
ENV OPENPROJECT_INSTALLATION__TYPE=docker
ENV OPENPROJECT_ATTACHMENTS__STORAGE__PATH=$APP_DATA_PATH/files
ENV OPENPROJECT_RAILS__CACHE__STORE=file_store
ENV OPENPROJECT_ANGULAR_UGLIFY=true

# set the following to an empty string if you don't want to automatically use the bundled hocuspocus
# (or set an external hocuspocus directly - if empty you can set it in the UI)
ENV OPENPROJECT_COLLABORATIVE__EDITING__HOCUSPOCUS__URL=auto
ENV OPENPROJECT_COLLABORATIVE__EDITING__HOCUSPOCUS__SECRET=

RUN useradd -d /home/$APP_USER -m $APP_USER && \
  mkdir -p $APP_PATH && chown $APP_USER:$APP_USER $APP_PATH && \
  mkdir -p $APP_DATA_PATH && chown $APP_USER:$APP_USER $APP_DATA_PATH && chmod g+rwx $APP_DATA_PATH

WORKDIR $APP_PATH

# upgrade bundler
RUN gem install bundler --no-document

# runtime dependencies
COPY ./docker/prod/setup/preinstall-common.sh ./docker/prod/setup/preinstall-common.sh
RUN ./docker/prod/setup/preinstall-common.sh

FROM runtime-base AS build-base
ARG NODE_VERSION="24.22.1"
ARG NPM_VERSION="11.19.0"

# build-only dependencies
COPY ./docker/prod/setup/preinstall-build.sh ./docker/prod/setup/preinstall-build.sh
RUN ./docker/prod/setup/preinstall-build.sh

FROM build-base AS app-build

# stuff required for gems
COPY Gemfile Gemfile.* .ruby-version ./
COPY modules ./modules
# Add vendor for saas-openproject plugins
COPY vendor ./vendor
# Add lib in case a plugin tries to load VERSION file under lib
COPY lib ./lib

# The restored vendor/bundle cache already arrives with `COPY vendor` above. Naming
# it again here copied its *contents* into vendor/, leaving a stale second gem tree.
COPY ./docker/prod/setup/bundle-install.sh ./vendor/
RUN bash vendor/bundle-install.sh && rm vendor/bundle-install.sh

COPY . .

# Copy lock file again as the updated version was overriden by COPY just now.
# `COPY . .` also restores the CI vendor/bundle cache over the installed bundle, so
# the superseded gem versions bundle-install.sh pruned are back and need pruning again.
RUN cp Gemfile.lock.bak Gemfile.lock && rm Gemfile.lock.bak && \
  bundle clean --force && \
  ./docker/prod/setup/precompile-assets.sh && \
  ./docker/prod/setup/dump-schema.sh

FROM app-build AS app-build-slim
COPY ./docker/prod/setup/prune-slim-runtime.sh ./docker/prod/setup/prune-slim-runtime.sh
RUN ./docker/prod/setup/prune-slim-runtime.sh

FROM runtime-base AS app-runtime

COPY --chown=$APP_USER:$APP_USER --from=app-build /app /app

RUN ./docker/prod/setup/postinstall-common.sh && \
  cp ./config/database.production.yml config/database.yml && \
  ln -s $APP_PATH/docker/prod/setup/.irbrc /home/$APP_USER/

FROM runtime-base AS app-runtime-slim

COPY --chown=$APP_USER:$APP_USER --from=app-build-slim /app /app

RUN ./docker/prod/setup/postinstall-common.sh && \
  cp ./config/database.production.yml config/database.yml && \
  ln -s $APP_PATH/docker/prod/setup/.irbrc /home/$APP_USER/

# -------------------------------------
# slim (public)
# -------------------------------------
FROM app-runtime-slim AS slim

USER $APP_USER
EXPOSE 8080
CMD ["./docker/prod/web"]
ENTRYPOINT ["./docker/prod/entrypoint-slim.sh"]
VOLUME ["$APP_DATA_PATH"]

# -------------------------------------
# slim-bim (public)
# same as slim but with BIM support enabled
# -------------------------------------
FROM app-runtime-slim AS slim-bim

USER $APP_USER
EXPOSE 8080
CMD ["./docker/prod/web"]
ENTRYPOINT ["./docker/prod/entrypoint-slim.sh"]
VOLUME ["$APP_DATA_PATH"]
ENV OPENPROJECT_EDITION=bim
# -------------------------------------
# all-in-one (public)
# -------------------------------------
# Wefabricate wf02: the all-in-one software is installed on runtime-base before the app
# arrives, and the app arrives in three layers (gems, compiled assets, code). A change to
# the code then rebuilds only the code layer, so the WF02 host pulls that layer instead of
# the whole image. Only the Postgres 17 server is installed, as the data volume is 17.
FROM runtime-base AS all-in-one-base

ENV PGVERSION_CHOICES="17"
ENV OPENPROJECT_RAILS__CACHE__STORE=memcache
ENV DATABASE_URL=postgres://openproject:openproject@127.0.0.1/openproject
ENV PGDATA=/var/openproject/pgdata
ENV NODE_OPTIONS=--use-openssl-ca

COPY --from=openproject/hocuspocus:17.9.6 --chown=$APP_USER:$APP_USER /app /opt/hocuspocus
# Keep node/npm in all-in-one for bundled hocuspocus even when BIM support is disabled.
COPY --from=build-base /usr/local/bin/node /usr/local/bin/node
COPY --from=build-base /usr/local/lib/node_modules /usr/local/lib/node_modules

COPY ./docker/prod/setup/postinstall-onprem.sh ./docker/prod/setup/postinstall-onprem.sh
RUN ./docker/prod/setup/postinstall-onprem.sh && \
  ln -sf ../lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm && \
  ln -sf ../lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx && \
  ln -sf ../lib/node_modules/corepack/dist/corepack.js /usr/local/bin/corepack && \
  ln -s /app/docker/prod/setup/.irbrc /root/

# Build-only: the app (pruned as for slim: no frontend sources, specs or source maps)
# split into the parts that get a layer of their own below. Sprockets names its manifest
# randomly on every build, so it travels with the code and leaves the assets layer stable.
FROM app-build-slim AS app-build-assets
RUN rm public/assets/.sprockets-manifest-*.json

FROM app-build-slim AS app-build-code
RUN mkdir /tmp/manifest && mv public/assets/.sprockets-manifest-*.json /tmp/manifest/ && \
  rm -rf vendor/bundle public/assets && \
  mkdir public/assets && mv /tmp/manifest/.sprockets-manifest-*.json public/assets/

FROM all-in-one-base AS all-in-one

# Gems and assets change far less often than the code; a COPY whose content is unchanged
# hits the cache and keeps its layer digest, even when app-build itself was rebuilt.
COPY --chown=$APP_USER:$APP_USER --from=app-build-slim /app/vendor/bundle /app/vendor/bundle
COPY --chown=$APP_USER:$APP_USER --from=app-build-assets /app/public/assets /app/public/assets
COPY --chown=$APP_USER:$APP_USER --from=app-build-code /app /app

RUN ./docker/prod/setup/postinstall-common.sh && \
  cp ./config/database.production.yml config/database.yml && \
  ln -s $APP_PATH/docker/prod/setup/.irbrc /home/$APP_USER/

# Expose ports for apache and postgres
EXPOSE 80

# Expose the postgres data directory and OpenProject data directory as volumes
VOLUME ["$PGDATA", "$APP_DATA_PATH"]

# Set a custom entrypoint to allow for privilege dropping and one-off commands
ENTRYPOINT ["./docker/prod/entrypoint.sh"]

# Set default command to launch the all-in-one configuration supervised by supervisord
CMD ["./docker/prod/supervisord"]
