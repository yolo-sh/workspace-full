# Workspace / Full

This repository contains the source code of the Docker image named `ghcr.io/yolo-sh/workspace-full`. 

This image contains the runtimes for the environments created via the [CLI](https://github.com/yolo-sh/cli).

## Table of contents
- [Requirements](#requirements)
- [Build](#build)
- [Runtimes](#runtimes)
- [Image](#image)
- [License](#license)

## Requirements

- `Docker`

## Build

In this repository root directory, run:

```bash
docker build -t yolo-full-workspace-image .
```
## Runtimes

The following runtimes are installed in this image:

- `docker (latest)`

- `docker compose (latest)`

- `php (latest)`

- `java 17.0 / maven 3.8`

- `node 18.7 (via nvm)`

- `python 3.10 (via pyenv)`

- `ruby 3.1 (via rvm)`

- `rust (latest)`

- `go (latest)`

## Image

The Dockerfile has been extensively commented to be self-explanatory. You can see it below:

```Dockerfile
# All environments need to inherit from "workspace-base"
FROM ghcr.io/yolo-sh/workspace-base:0.0.2

LABEL org.opencontainers.image.source=https://github.com/yolo-sh/workspace-full
LABEL org.opencontainers.image.description="The Docker image that contains the runtimes for the environments created via the Yolo CLI"

ARG DEBIAN_FRONTEND=noninteractive

# RUN will use bash
SHELL ["/bin/bash", "-c"]

# Run the following commands as root
USER root

# Force LibSSL to 1.1.1 to avoid conflicts 
# with old Ruby and Python versions
RUN set -euo pipefail \
  && apt-add-repository --yes ppa:rael-gc/rvm \
  && apt-get --assume-yes --quiet --quiet update \
  && apt-get --assume-yes --quiet --quiet remove libssl-dev \
  && touch /etc/apt/preferences.d/rael-gc-rvm-precise-pin-900 \
  && { echo 'Package: *'; \
    echo 'Pin: release o=LP-PPA-rael-gc-rvm'; \
    echo 'Pin-Priority: 900'; } >> /etc/apt/preferences.d/rael-gc-rvm-precise-pin-900 \
  && apt-get --assume-yes --quiet --quiet install libssl-dev \
  && apt-get clean && rm --recursive --force /var/lib/apt/lists/* /tmp/*

# Install Docker
RUN set -euo pipefail \
  && curl --fail --silent --show-error --location https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor --output /usr/share/keyrings/docker-archive-keyring.gpg \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release --codename --short) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null \
  && apt-get --assume-yes --quiet --quiet update \
  && apt-get --assume-yes --quiet --quiet install docker-ce docker-ce-cli containerd.io \
  && apt-get clean && rm --recursive --force /var/lib/apt/lists/* /tmp/*

# Install Docker compose
RUN set -euo pipefail \
  && LATEST_COMPOSE_VERSION=$(curl --fail --silent --show-error --location "https://api.github.com/repos/docker/compose/releases/latest" | grep --only-matching --perl-regexp '(?<="tag_name": ").+(?=")') \
  && curl --fail --silent --show-error --location "https://github.com/docker/compose/releases/download/${LATEST_COMPOSE_VERSION}/docker-compose-$(uname --kernel-name)-$(uname --machine)" --output /usr/libexec/docker/cli-plugins/docker-compose \
  && chmod +x /usr/libexec/docker/cli-plugins/docker-compose

# Add default user to docker group to avoid 
# having to run all docker commands with sudo
RUN set -euo pipefail \
  && groupadd --force docker \
  && usermod --append --groups docker "${USER}"

# Install PHP
RUN set -euo pipefail \
  && apt-get --assume-yes --quiet --quiet update \
  && apt-get --assume-yes --quiet --quiet install \
    composer \
    php-all-dev \
    php-apcu \
    php-cli \
    php-ctype \
    php-curl \
    php-date \
    php-dom \
    php-fileinfo \
    php-fpm \
    php-gd \
    php-iconv \
    php-imagick \
    php-intl \
    php-json \
    php-mbstring \
    php-mysql \
    php-mysqli \
    php-net-ftp \
    php-opcache \
    php-pdo \
    php-pgsql \
    php-phar \
    php-php-gettext \
    php-posix \
    php-simplexml \
    php-sqlite3 \
    php-tokenizer \
    php-xml \
    php-xmlwriter \
    php-zip \
  && apt-get clean && rm --recursive --force /var/lib/apt/lists/* /tmp/*

# Install Clang compiler (C/C++)
RUN set -euo pipefail \
  && curl --silent --show-error --location --fail https://apt.llvm.org/llvm-snapshot.gpg.key | apt-key add - \
  && apt-add-repository --yes "deb http://apt.llvm.org/jammy/ llvm-toolchain-jammy main" \
  && apt-get install --assume-yes --quiet --quiet \
    clang-format \
    clang-tools \
    cmake \
    clangd-14 \
  && update-alternatives --install /usr/bin/clangd clangd /usr/bin/clangd-14 100 \
  && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/*

# Install Java & Maven
RUN set -euo pipefail \
  && add-apt-repository --yes ppa:linuxuprising/java \
  && apt-get --assume-yes --quiet --quiet update \
  && echo oracle-java17-installer shared/accepted-oracle-license-v1-3 select true | debconf-set-selections \
  && apt-get install --assume-yes --quiet --quiet \
      gradle \
      oracle-java17-installer \
  && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/*

ARG MAVEN_VERSION=3.8.6
ENV MAVEN_HOME=/usr/share/maven
ENV PATH=$MAVEN_HOME/bin:$PATH
RUN set -euo pipefail \
  && mkdir --parents $MAVEN_HOME \
  && curl --silent --show-error --location --fail https://apache.osuosl.org/maven/maven-3/$MAVEN_VERSION/binaries/apache-maven-$MAVEN_VERSION-bin.tar.gz \
    | tar --extract --gzip --verbose --directory $MAVEN_HOME --strip-components=1

# Run the following commands 
# using default user
USER $USER
WORKDIR $HOME

# Install Node.js.
# Nvm uses the "NODE_VERSION" env var to 
# choose which version needs to be installed.
ARG NODE_VERSION=18.7.0
ENV PATH=$PATH:$HOME/.nvm/versions/node/v$NODE_VERSION/bin
RUN set -euo pipefail \
  && curl --silent --show-error --location --fail https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash \
  && bash -c ". .nvm/nvm.sh \
    && npm config set python /usr/bin/python --global \
    && npm config set python /usr/bin/python \
    && npm install -g typescript \
    && npm install -g yarn"

# Install Python
ENV PYTHON_VERSION=3.10.6
ENV PATH=$PATH:$HOME/.pyenv/bin:$HOME/.pyenv/shims
RUN set -euo pipefail \
  && sudo apt-get --assume-yes --quiet --quiet update \
  && sudo apt-get --assume-yes --quiet --quiet install \
    libbz2-dev \
    libffi-dev \
    liblzma-dev \
    libncursesw5-dev \
    libreadline-dev \
    libsqlite3-dev \
    libxml2-dev \
    libxmlsec1-dev \
    llvm \
    make \
    tk-dev \
    xz-utils \
    zlib1g-dev \
  && sudo apt-get clean && sudo rm --recursive --force /var/lib/apt/lists/* /tmp/*
RUN set -euo pipefail \
  && curl --silent --show-error --location --fail https://github.com/pyenv/pyenv-installer/raw/master/bin/pyenv-installer | bash \
  && { echo; \
    echo 'eval "$(pyenv init -)"'; \
    echo 'eval "$(pyenv virtualenv-init -)"'; } >> .zshrc \
  && pyenv install ${PYTHON_VERSION} \
  && pyenv global ${PYTHON_VERSION} \
  && pip install virtualenv pipenv python-language-server[all]==0.19.0 \
  && rm -rf /tmp/*

# Install Ruby
ENV RUBY_VERSION=3.1.2
RUN set -euo pipefail \
  && curl --silent --show-error --location --fail https://rvm.io/mpapis.asc | gpg --import - \
  && curl --silent --show-error --location --fail https://rvm.io/pkuczynski.asc | gpg --import - \
  && curl --silent --show-error --location --fail https://get.rvm.io | bash -s stable \
  && bash -lc " \
    rvm requirements \
    && rvm install ${RUBY_VERSION} \
    && rvm use ${RUBY_VERSION} --default \
    && rvm rubygems current \
    && gem install bundler --no-document" \
  && { echo; \
    echo '[[ -s "$HOME/.rvm/scripts/rvm" ]] && . "$HOME/.rvm/scripts/rvm"'; } >> .zshrc

# Install Rust
RUN set -euo pipefail \
  && curl --proto '=https' --tlsv1.2 --silent --show-error --location --fail https://sh.rustup.rs | sh -s -- -y \
  && .cargo/bin/rustup update \
  && .cargo/bin/rustup component add rls-preview rust-analysis rust-src \
  && .cargo/bin/rustup completions zsh > ~/.zfunc/_rustup

# Install Go
ENV PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
RUN set -euo pipefail \
  && cd /tmp \
  && LATEST_GO_VERSION=$(curl --fail --silent --show-error --location "https://go.dev/VERSION?m=text") \
  && ARCH=$(arch | sed s/aarch64/arm64/ | sed s/x86_64/amd64/) \
  && curl --fail --silent --show-error --location "https://go.dev/dl/${LATEST_GO_VERSION}.linux-${ARCH}.tar.gz" --output go.tar.gz \
  && sudo tar --directory /usr/local --extract --file go.tar.gz \
  && rm go.tar.gz \
  && go install github.com/ramya-rao-a/go-outline@latest \
  && go install github.com/cweill/gotests/gotests@latest \
  && go install github.com/fatih/gomodifytags@latest \
  && go install github.com/josharian/impl@latest \
  && go install github.com/haya14busa/goplay/cmd/goplay@latest \
  && go install github.com/go-delve/delve/cmd/dlv@latest \
  && go install honnef.co/go/tools/cmd/staticcheck@latest \
  && go install golang.org/x/tools/gopls@latest

WORKDIR $YOLO_WORKSPACE
```

## License

Yolo is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
