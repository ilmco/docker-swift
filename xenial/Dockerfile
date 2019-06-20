ARG ARCH
ARG CLANG_VERSION=6.0
ARG SWIFT_VERSION=5.0.1
ARG SWIFT_BRANCH=RELEASE
ARG BUILD_NUMBER

FROM ${ARCH}/ubuntu:xenial as builder
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
      libncurses5-dev \
      libpython3-dev \
      libsqlite3-dev \
      libtool \
      libxml2-dev \
      libz-dev \
      ninja-build \
      pkg-config \
      python3 \
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
RUN git clone https://github.com/apple/swift.git
RUN ./swift/utils/update-checkout --clone
RUN ./swift/utils/update-checkout --scheme swift-${SWIFT_VERSION}-branch --tag ${BRANCH}

RUN for PKG in $SWIFT_CORE_PACKAGES $SWIFT_EXTRA_PACKAGES; do \
      [ ! -d platform/patches ] || for PATCH_DIR in platform/patches/*.diffs; do \
        echo Patching ${PKG}; \
        TARGET_DIR=`echo $PATCH_DIR | sed -E 's/platform\/patches\/(.*).diffs/\1/'`; \
        for f in $PATCH_DIR/*.diff; do \
          [ ! -d $TARGET_DIR ] || (cd $TARGET_DIR; patch -p1 < ../$f); \
        done; \
      done; \
    done

ENV WORKSPACE=/output/root
ENV PACKAGE=/output/${BRANCH}.tgz
RUN mkdir -p ${WORKSPACE}
RUN ./swift/utils/build-script \
    -R \
    --build-subdir buildbot_linux --install-prefix=/usr \
    --install-destdir="${WORKSPACE}/install" --installable-package="${PACKAGE}" \
    --lldb --llbuild --xctest --swiftpm --foundation --libdispatch \
    -- --install-libdispatch --install-foundation --install-swift --install-lldb --install-llbuild --install-xctest --install-swiftpm \
    '--swift-install-components=autolink-driver;compiler;clang-builtin-headers;stdlib;swift-remote-mirror;sdk-overlay;dev' \
    --build-swift-static-stdlib --build-swift-static-sdk-overlay


#
# Stage -- Make debian packages
#
FROM ilmco/${ARCH}-ubuntu-fpm:16.04 as stage
COPY --from=builder /output/*.tgz /input/

ARG SWIFT_VERSION
ARG BUILD_NUMBER=0
ARG CLANG_VERSION

ENV SWIFT_VERSION=${SWIFT_VERSION} BUILD_NUMBER=${BUILD_NUMBER}
ENV PACKAGE_VERSION=${SWIFT_VERSION}-${BUILD_NUMBER}
ENV CLANG_VERSION=${CLANG_VERSION}

RUN mkdir -p /output

# Swift4 Runtime
WORKDIR /tmp/root-runtime
RUN tar xvf /input/swift-*-RELEASE.tgz --wildcards 'usr/lib/swift/linux/*.so'
RUN find . -name '*.so' -exec chmod +x {} \;
RUN fpm -s dir -t deb -n swift5-runtime -v ${PACKAGE_VERSION} -p /output -C `pwd` \
      -d libatomic1 \
      -d libbsd0 \
      -d libc6 \
      -d libcurl3 \
      -d libgcc1 \
      -d libstdc++6 \
      -d libxml2 \
      -d zlib1g
RUN find . -type f | cut -c 3- > /tmp/runtime-file.list

# Swift4
WORKDIR /tmp/root
RUN tar xvf /input/swift-*-RELEASE.tgz --exclude-from=/tmp/runtime-file.list
RUN find . -name '*.so' -exec chmod +x {} \;
RUN fpm -s dir -t deb -n swift5 -v ${PACKAGE_VERSION} -p /output -C `pwd` \
      -d libc6 \
      -d libgcc1 \
      -d libstdc++6 \
      -d swift5-runtime \
      -d clang-${CLANG_VERSION} \
      -d python3-minimal \
      -d libpython3.5

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
