FROM ubuntu:16.04

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      wget \
      make \
      libc6-dev \
      clang-3.8 \
      curl \
      libedit-dev \
      libpython2.7 \
      python2.7-minimal \
      libicu-dev \
      libssl-dev \
      libxml2 \
      tzdata \
      git \
      libcurl4-openssl-dev \
      pkg-config \
 && update-alternatives --quiet --install /usr/bin/clang clang /usr/bin/clang-3.8 100 \
 && update-alternatives --quiet --install /usr/bin/clang++ clang++ /usr/bin/clang++-3.8 100 \
 && rm -r /var/lib/apt/lists/*
RUN wget -q --no-check-certificate -O - https://swift.org/builds/swift-4.1.2-release/ubuntu1604/swift-4.1.2-RELEASE/swift-4.1.2-RELEASE-ubuntu16.04.tar.gz | tar zxvf - --strip-components=1
RUN chmod -R o+r /usr/lib/swift

RUN swift --version
