ARG ARCH=amd64
FROM ilmco/${ARCH}-ubuntu:16.04 as builder

ARG ARCH
ADD qemu-aarch64-static /usr/bin/

ARG CLANG_VERSION=3.8
ENV CLANG_VERSION=${CLANG_VERSION}

RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      autoconf \
      ca-certificates \
      clang-${CLANG_VERSION} \
      cmake \
      git \
      icu-devtools \
      libblocksruntime-dev \
      libbsd-dev \
      libcurl4-openssl-dev \
      libcurl4-openssl-dev \
      libedit-dev \
      libicu-dev \
      libncurses5-dev \
      libpython-dev \
      libsqlite3-dev \
      libtool \
      libxml2-dev \
      libz-dev \
      ninja-build \
      pkg-config \
      python \
      rsync \
      swig \
      systemtap-sdt-dev \
      uuid-dev \
      wget \
 && rm -r /var/lib/apt/lists/* \
 && rm -r /var/cache/apt/*

RUN update-alternatives --quiet --install /usr/bin/clang clang /usr/bin/clang-${CLANG_VERSION} 100
RUN update-alternatives --quiet --install /usr/bin/clang++ clang++ /usr/bin/clang++-${CLANG_VERSION} 100

WORKDIR /root/build

ENV RELEASE=4.1.2-RELEASE
ENV BRANCH=swift-${RELEASE}

RUN wget -qO - https://github.com/apple/swift/archive/${BRANCH}.tar.gz | (mkdir swift; cd swift; tar zxf - --strip-components=1) \
 && wget -qO - https://github.com/ninja-build/ninja/archive/v1.8.2.tar.gz | (mkdir ninja; cd ninja; tar zxf - --strip-components=1) \
 && wget -qO - https://github.com/apple/swift-package-manager/archive/${BRANCH}.tar.gz | (mkdir swiftpm; cd swiftpm; tar zxf - --strip-components=1) \
 && for PKG in clang cmark compiler-rt llbuild lldb llvm; do \
      echo Downloading ${PKG}; \
      wget -qO - https://github.com/apple/swift-${PKG}/archive/${BRANCH}.tar.gz | (mkdir ${PKG}; cd ${PKG}; tar zxf - --strip-components=1); \
    done \
 && for PKG in swift-corelibs-foundation swift-corelibs-libdispatch swift-corelibs-xctest swift-integration-tests swift-xcode-playground-support; do \
      echo Downloading ${PKG}; \
      wget -qO - https://github.com/apple/${PKG}/archive/${BRANCH}.tar.gz | (mkdir ${PKG}; cd ${PKG}; tar zxf - --strip-components=1); \
    done

COPY ${ARCH} platform
RUN [ ! -d platform/patches ] || for PATCH_DIR in platform/patches/*.diffs; do \
      TARGET_DIR=`echo $PATCH_DIR | sed -E 's/platform\/patches\/(.*).diffs/\1/'`; \
      for f in $PATCH_DIR/*.diff; do \
        (cd $TARGET_DIR; patch -p1 < ../$f); \
      done; \
    done;

ENV INSTALL_DIR=install
ENV PACKAGE=swift-${RELEASE}.tgz
RUN ./swift/utils/build-script --build-subdir buildbot_linux \
      -R \
      --lldb \
      --llbuild \
      --xctest \
      --swiftpm \
      --foundation \
      --libdispatch \
      -- \
      --install-libdispatch \
      --install-foundation \
      --install-swift \
      --install-lldb \
      --install-llbuild \
      --install-xctest \
      --install-swiftpm \
      --install-prefix=/usr \
      --swift-install-components='autolink-driver;compiler;clang-builtin-headers;stdlib;swift-remote-mirror;sdk-overlay;dev' \
      --build-swift-static-stdlib \
      --build-swift-static-sdk-overlay \
      --install-destdir=$(pwd)/${INSTALL_DIR} \
      --installable-package=$(pwd)/${PACKAGE}


FROM ilmco/${ARCH}-ubuntu:16.04
COPY --from=builder /root/build/*.tgz /
