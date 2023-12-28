FROM alpine:3.19 as base

RUN apk update && apk upgrade

RUN apk update && \
    apk add --update --no-cache \
    git \
    wget \
    libressl-dev \
    bash \
    curl \
    vim \
    zip \
    unzip \
    openssl \
    build-base \
    && rm -rf /var/cache/apk/*


FROM base as deps_node

RUN apk add --update --no-cache \
    nodejs \
    npm \
    && rm -rf /var/cache/apk/*

RUN npm install -g pnpm@latest \
    && npm cache clean --force \
    && echo "store-dir = /pnpm-store" >> /etc/npmrc

VOLUME ["/pnpm-store"]

FROM deps_node as deps_rust

RUN curl https://sh.rustup.rs -sSf | bash -s -- -y
ENV PATH="/home/node/.cargo/bin:${PATH}"