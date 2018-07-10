ARG ARCH
ARG CLANG_VERSION=6.0
ARG SWIFT_VERSION=4.1.2
ARG SWIFT_BRANCH=RELEASE
ARG BUILD_NUMBER=0

FROM ilmco/${ARCH}-ubuntu:16.04 as builder
ARG ARCH
ARG CLANG_VERSION
ARG SWIFT_VERSION
ARG SWIFT_BRANCH

ADD qemu-aarch64-static /usr/bin/

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
 && find /var/cache/apt/archives /var/lib/apt/lists -not -name lock -type f -delete \
 && update-alternatives --install /usr/bin/clang++ clang++ /usr/bin/clang++-$CLANG_VERSION 100 \
 && update-alternatives --install /usr/bin/clang clang /usr/bin/clang-$CLANG_VERSION 100

WORKDIR /root/build
COPY ${ARCH} platform

ENV SWIFT_RELEASE=${SWIFT_VERSION}-${SWIFT_BRANCH}
ENV BRANCH=swift-${SWIFT_RELEASE}
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
    done \
 && [ ! -d platform/patches ] || for PATCH_DIR in platform/patches/*.diffs; do \
      TARGET_DIR=`echo $PATCH_DIR | sed -E 's/platform\/patches\/(.*).diffs/\1/'`; \
      for f in $PATCH_DIR/*.diff; do \
        [ ! -d $TARGET_DIR ] || (cd $TARGET_DIR; patch -p1 < ../$f); \
      done; \
    done;

ENV INSTALL_DIR=/output/root
ENV PACKAGE=/output/${BRANCH}.tgz
RUN mkdir -p ${INSTALL_DIR}
RUN ./swift/utils/build-script \
      -R \
      --host-cc=/usr/bin/clang-${CLANG_VERSION} \
      --host-cxx=/usr/bin/clang++-${CLANG_VERSION} \
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
      --install-destdir=${INSTALL_DIR} \
      --installable-package=${PACKAGE}

#
# Stage -- Make debian packages
#
FROM ilmco/${ARCH}-ubuntu-fpm:16.04 as stage
COPY --from=builder /output/*.tgz /input/

ARG SWIFT_VERSION
ARG BUILD_NUMBER
ARG CLANG_VERSION

ENV SWIFT_VERSION=${SWIFT_VERSION} BUILD_NUMBER=${BUILD_NUMBER}
ENV PACKAGE_VERSION=${SWIFT_VERSION}-${BUILD_NUMBER}
ENV CLANG_VERSION=${CLANG_VERSION}

RUN mkdir -p /output

# Swift4 Runtime
WORKDIR /tmp/root-runtime
RUN tar xvf /input/swift-*-RELEASE.tgz --wildcards 'usr/lib/swift/linux/*.so'
RUN find . -name '*.so' -exec chmod +x {} \;
RUN fpm -s dir -t deb -n swift4-runtime -v ${PACKAGE_VERSION} -p /output -C `pwd` \
      -d libicu55 \
      -d libcurl3 \
      -d libxml2 \
      -d libstdc++6 \
      -d libatomic1
RUN find . -type f | cut -c 3- > /tmp/runtime-file.list

# Swift4
WORKDIR /tmp/root
RUN tar xvf /input/swift-*-RELEASE.tgz --exclude-from=/tmp/runtime-file.list
RUN fpm -s dir -t deb -n swift4 -v ${PACKAGE_VERSION} -p /output -C `pwd` \
      -d swift4-runtime \
      -d clang-${CLANG_VERSION} \
      -d python2.7-minimal

#
# Test -- Construct a new image, install packages from stage, test.
#
FROM ilmco/${ARCH}-ubuntu:16.04 as test
COPY qemu-aarch64-static /usr/bin

VOLUME /pkgs
COPY --from=stage /output/*.deb /pkgs/
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      /pkgs/*.deb \
 && rm -r /var/lib/apt/lists/* \
 && rm -r /var/cache/apt/*

RUN swift --version
# TODO: Run more tests
