# The official nginx images do not ship the brotli module, so build it
# against the exact nginx version used at runtime and install it as a
# dynamic module.
ARG NGINX_VERSION=1.31.8
ARG NGX_BROTLI_COMMIT=53e0fb55f250f48fe80c0ddbf6c0f7103cdbaae1
ARG HUGO_VERSION=0.168.0

FROM nginx:${NGINX_VERSION} AS brotli

ARG NGINX_VERSION
ARG NGX_BROTLI_COMMIT

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      ca-certificates curl git build-essential cmake \
      libpcre2-dev zlib1g-dev libssl-dev \
 && rm -rf /var/lib/apt/lists/*

WORKDIR /src
RUN curl -fsSL "https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" | tar xz \
 && git clone https://github.com/google/ngx_brotli.git \
 && cd ngx_brotli \
 && git checkout "${NGX_BROTLI_COMMIT}" \
 && git submodule update --init --recursive

# ngx_brotli links the brotli libraries statically into the modules.
RUN cd ngx_brotli/deps/brotli \
 && cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -B out . \
 && cmake --build out -j"$(nproc)"

WORKDIR /src/nginx-${NGINX_VERSION}
RUN ./configure --with-compat --add-dynamic-module=/src/ngx_brotli \
 && make modules

# Site build tools: hugo + the scripts in site/tools. The source tree is
# mounted at /src at runtime, so content changes do not require a rebuild.
FROM debian:trixie-slim AS builder

ARG HUGO_VERSION

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      ca-certificates curl bash make findutils imagemagick \
 && rm -rf /var/lib/apt/lists/* \
 && case "$(uname -m)" in \
      x86_64)  hugo_arch=amd64 ;; \
      aarch64) hugo_arch=arm64 ;; \
      *) echo "unsupported architecture: $(uname -m)" >&2; exit 1 ;; \
    esac \
 && curl -fsSL "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-${hugo_arch}.tar.gz" \
      | tar -xz -C /usr/local/bin hugo \
 && hugo version \
 && magick -version | head -n 1

ENV HOME=/tmp \
    HUGO_CACHEDIR=/tmp/hugo-cache

WORKDIR /src

FROM nginx:${NGINX_VERSION}

ARG NGINX_VERSION

COPY --from=brotli /src/nginx-${NGINX_VERSION}/objs/ngx_http_brotli_filter_module.so \
     /src/nginx-${NGINX_VERSION}/objs/ngx_http_brotli_static_module.so \
     /usr/lib/nginx/modules/

# The stock nginx.conf has no include for extra modules, so prepend the
# load_module directives to the upstream config.
RUN { \
      echo 'load_module /usr/lib/nginx/modules/ngx_http_brotli_filter_module.so;'; \
      echo 'load_module /usr/lib/nginx/modules/ngx_http_brotli_static_module.so;'; \
      cat /etc/nginx/nginx.conf; \
    } > /tmp/nginx.conf \
 && mv /tmp/nginx.conf /etc/nginx/nginx.conf
