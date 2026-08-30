# Multi-stage: build the React app, then serve it with nginx.
# Self-hosters never need Node locally — `docker compose up` builds everything.
#
# --platform=$BUILDPLATFORM pins the build stage to the host's native arch even when
# cross-building for other targets (e.g. amd64 host building an arm64 image). The build
# output (static JS/CSS/HTML) is arch-independent, so there's no reason to run it under
# QEMU — and QEMU-emulated npm installs are known to corrupt esbuild/rollup's platform-
# specific native binaries, which is what breaks `vite build` with unrelated-looking
# module-resolution errors.
FROM --platform=$BUILDPLATFORM node:22-alpine AS build
WORKDIR /app
COPY frontend/package.json frontend/package-lock.json* ./
RUN npm ci 2>/dev/null || npm install
COPY frontend/ ./
RUN npm run build

# Exercise images + GIFs (~140 MB, CC — github.com/hasaneyldrm/exercises-dataset).
# docker compose hands these over from a sidecar through a shared volume; Railway has no
# shared volume between services, so on this branch they're baked into the web image.
FROM alpine/git AS media
RUN git clone --depth 1 https://github.com/hasaneyldrm/exercises-dataset /ds \
 && mkdir -p /media/img /media/gif \
 && cp /ds/images/*.jpg /media/img/ \
 && cp /ds/videos/*.gif /media/gif/

FROM nginx:alpine
# nginx:alpine runs envsubst over /etc/nginx/templates/*.template at container start, and
# sources executable *.envsh files from /docker-entrypoint.d first (05-resolver.envsh).
COPY web/nginx.conf.template /etc/nginx/templates/default.conf.template
COPY web/docker-entrypoint.d/05-resolver.envsh /docker-entrypoint.d/05-resolver.envsh
ENV NGINX_ENVSUBST_FILTER="^(PORT|API_UPSTREAM|NGINX_RESOLVER)$" \
    PORT=8080 \
    API_UPSTREAM=api.railway.internal:3000
COPY --from=build /app/dist /usr/share/nginx/html
COPY --from=media /media/img /usr/share/nginx/html/img
COPY --from=media /media/gif /usr/share/nginx/html/gif
