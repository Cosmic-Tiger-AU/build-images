FROM golang:1.20-alpine3.19 as ssm-builder
ARG VERSION=3.2.2086.0
RUN set -ex && apk add --no-cache make git gcc libc-dev curl bash && \
    curl -sLO https://github.com/aws/amazon-ssm-agent/archive/${VERSION}.tar.gz && \
    mkdir -p /go/src/github.com && \
    tar xzf ${VERSION}.tar.gz && \
    mv amazon-ssm-agent-${VERSION} /go/src/github.com/amazon-ssm-agent && \
    cd /go/src/github.com/amazon-ssm-agent && \
    echo ${VERSION} > VERSION && \
    gofmt -w agent && make checkstyle || ./Tools/bin/goimports -w agent && \
    make build-linux

FROM alpine:3.19 as base
RUN apk update && apk upgrade 
RUN apk add --no-cache bash git openssh openssh-client-default curl wget tar gzip ca-certificates openssl \
    bzip2 e2fsprogs glib gzip zip unzip argon2-dev build-base aws-cli sqlite-dev \
    curl-dev libzip-dev rsync wget which xfsprogs sudo gcompat libc6-compat \
    && update-ca-certificates \
    && rm -rf /var/cache/apk/*

RUN addgroup -g 1000 codebuild-user && adduser -G root -D -u 1000 codebuild-user \
    && echo "codebuild-user ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers.d/codebuild-user \
    && chmod 0440 /etc/sudoers.d/codebuild-user
RUN adduser -D ssm-user && echo "ssm-user ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/ssm-agent-users && \
    mkdir -p /etc/amazon/ssm

FROM base as tools

# Install stunnel
RUN set -ex \
    && STUNNEL_VERSION=5.69 \
    && STUNNEL_TAR=stunnel-$STUNNEL_VERSION.tar.gz \
    && STUNNEL_SHA256="1ff7d9f30884c75b98c8a0a4e1534fa79adcada2322635e6787337b4e38fdb81" \
    && curl -o $STUNNEL_TAR https://www.stunnel.org/archive/5.x/$STUNNEL_TAR \
    && echo "$STUNNEL_SHA256 $STUNNEL_TAR" | sha256sum -c - \
    && tar xvfz $STUNNEL_TAR \
    && cd stunnel-$STUNNEL_VERSION \
    && ./configure \
    && make -j4 \
    && make install \
    && openssl genrsa -out key.pem 2048 \
    && openssl req -new -x509 -key key.pem -out cert.pem -days 1095 -subj "/C=US/ST=Washington/L=Seattle/O=Amazon/OU=Codebuild/CN=codebuild.amazon.com" \
    && cat key.pem cert.pem >> /usr/local/etc/stunnel/stunnel.pem \
    && cd .. ; rm -rf stunnel-${STUNNEL_VERSION}*

# AWS Tools
RUN curl -sS -o /usr/local/bin/aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.25.6/2023-01-30/bin/linux/arm64/aws-iam-authenticator \
    && curl -sS -o /usr/local/bin/kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.25.6/2023-01-30/bin/linux/arm64/kubectl \
    && curl -sS -o /usr/local/bin/ecs-cli https://amazon-ecs-cli.s3.amazonaws.com/ecs-cli-linux-arm64-latest \
    && chmod +x /usr/local/bin/kubectl /usr/local/bin/aws-iam-authenticator /usr/local/bin/ecs-cli

# SSM
COPY --from=ssm-builder /go/src/github.com/amazon-ssm-agent/bin/linux_amd64/ /usr/bin
COPY --from=ssm-builder /go/src/github.com/amazon-ssm-agent/bin/amazon-ssm-agent.json.template /etc/amazon/ssm/amazon-ssm-agent.json
COPY --from=ssm-builder /go/src/github.com/amazon-ssm-agent/bin/seelog_unix.xml /etc/amazon/ssm/seelog.xml

# Node
ENV N_SRC_DIR="$SRC_DIR/n"
RUN git clone https://github.com/tj/n $N_SRC_DIR \
    && cd $N_SRC_DIR && make install

# Python
RUN curl https://pyenv.run | bash
ENV PATH="/root/.pyenv/shims:/root/.pyenv/bin:$PATH"

FROM tools as runtimes
#****************      NODEJS     ****************************************************

ENV NODE_18_VERSION="20.10.0"

RUN  n $NODE_18_VERSION && npm install --save-dev -g -f grunt && npm install -g -f pnpm && npm install --save-dev -g -f grunt-cli && npm install --save-dev -g -f webpack \
    && cd / && rm -rf $N_SRC_DIR; rm -rf /tmp/*


#**************** PYTHON *****************************************************
ENV PYTHON_311_VERSION="3.11.2"
ENV PYTHON_PIP_VERSION=23.0.1
ENV PYYAML_VERSION=6.0.1

COPY tools/runtime_configs/python/$PYTHON_311_VERSION /root/.pyenv/plugins/python-build/share/python-build/$PYTHON_311_VERSION
RUN   env PYTHON_CONFIGURE_OPTS="--enable-shared" pyenv install $PYTHON_311_VERSION; rm -rf /tmp/*
RUN   pyenv global  $PYTHON_311_VERSION
RUN set -ex \
    && pip3 install --no-cache-dir --upgrade --force-reinstall "pip==$PYTHON_PIP_VERSION" \
    && pip3 install --no-cache-dir --upgrade "PyYAML==$PYYAML_VERSION" \
    && pip3 install --no-cache-dir --upgrade 'setuptools==57.4.0' wheel aws-sam-cli boto3 pipenv virtualenv


FROM runtimes as docker_rt
#Docker 2.17
ENV DOCKER_BUCKET="download.docker.com" \
    DOCKER_CHANNEL="stable" \
    DIND_COMMIT="3b5fac462d21ca164b3778647420016315289034"

ENV DOCKER_VERSION="23.0.1"
ENV DOCKER_COMPOSE_VERSION="2.17.2"

VOLUME /var/lib/docker

RUN set -ex \
    && curl -fSL "https://${DOCKER_BUCKET}/linux/static/${DOCKER_CHANNEL}/aarch64/docker-${DOCKER_VERSION}.tgz" -o docker.tgz \
    && tar --extract --file docker.tgz --strip-components 1  --directory /usr/local/bin \
    && rm docker.tgz \
    && docker -v \
    # set up subuid/subgid so that "--userns-remap=default" works out-of-the-box
    && groupadd dockremap \
    && useradd -g dockremap dockremap \
    && echo 'dockremap:165536:65536' >> /etc/subuid \
    && echo 'dockremap:165536:65536' >> /etc/subgid \
    && wget -q "https://raw.githubusercontent.com/docker/docker/${DIND_COMMIT}/hack/dind" -O /usr/local/bin/dind \
    && curl -L https://github.com/docker/compose/releases/download/v${DOCKER_COMPOSE_VERSION}/docker-compose-Linux-aarch64 > /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/dind /usr/local/bin/docker-compose \
    # Ensure docker-compose works
    && docker-compose version