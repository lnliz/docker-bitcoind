# This Dockerfile builds Bitcoin Inquisition and packages it into a minimal `final` image

# VERSION of Bitcoin Inquitision to be build
#   NOTE: Unlike our other images this one is NOT prefixed with `v`,
#           as many things (like download URLs) use this form instead.
ARG VERSION=29.1-inq

# CPU architecture to build binaries for
ARG ARCH

# Define default versions so that they don't have to be repeated throughout the file
ARG VER_ALPINE=3.22

# $USER name, and data $DIR to be used in the `final` image
ARG USER=bitcoind
ARG DIR=/data


FROM alpine:${VER_ALPINE} AS preparer

ARG VERSION

# Download source code from same website as github is probably deterministicly built
ADD https://github.com/bitcoin-inquisition/bitcoin/releases/download/v$VERSION/bitcoin-$VERSION.tar.gz ./bitcoin-$VERSION.tar.gz

# Extract
RUN tar -xzf "bitcoin-$VERSION.tar.gz" && \
    rm -f "bitcoin-$VERSION.tar.gz"


#
## `builder` builds Bitcoin Inquisition regardless on how the source, and BDB code were obtained.
#
# NOTE: this stage is emulated using QEMU
# NOTE: `${ARCH:+${ARCH}/}` - if ARCH is set, append `/` to it, leave it empty otherwise
FROM ${ARCH:+${ARCH}/}alpine:${VER_ALPINE} AS builder

ARG VERSION

RUN apk add --no-cache \
    cmake \
    boost-dev \
    sqlite-dev \
    build-base \
    chrpath \
    file \
    libevent-dev \
    libressl \
    libtool \
    linux-headers \
    zeromq-dev

# Change to the extracted directory
WORKDIR /bitcoin-$VERSION/

# Copy bitcoin source (downloaded & verified in previous stages)
COPY  --from=preparer /bitcoin-$VERSION/  ./

ENV BITCOIN_PREFIX="/opt/bitcoin-$VERSION"

#
# BUILD_TESTS is slow but needed for bitcoin-wallet tool
#
RUN cmake -S . -B build \
    -DCMAKE_INSTALL_PREFIX="$BITCOIN_PREFIX" \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_EXE_LINKER_FLAGS="-L/opt/db4/lib/" \
    -DCMAKE_CXX_FLAGS="-I/opt/db4/include/" \
    -DBUILD_CLI=ON \
    -DBUILD_UTIL=ON \
    -DBUILD_TX=ON \
    -DBUILD_SHARED_LIBS=OFF \
    -DENABLE_WALLET=ON \
    -DREDUCE_EXPORTS=ON \
    -DBUILD_TESTS=ON \
    -DBUILD_BENCH=OFF \
    -DBUILD_GUI=OFF \
    -DBUILD_GUI_TESTS=OFF \
    -DWITH_SQLITE=ON \
    -DWITH_ZMQ=ON \
    -DWITH_CCACHE=ON

RUN cmake --build build -- -j$(( $(nproc) + 1 ))
RUN cmake --install build

# List installed binaries pre-strip & strip them
RUN ls -lh "$BITCOIN_PREFIX/bin/"
RUN strip -v "$BITCOIN_PREFIX/bin/bitcoin"*

# List installed binaries post-strip & print their checksums
RUN ls -lh "$BITCOIN_PREFIX/bin/"
RUN sha256sum "$BITCOIN_PREFIX/bin/bitcoin"*



#
## `final` aggregates build results from previous stages into a necessary minimum
#       ready to be used, and published to Docker Hub.
#
# NOTE: this stage is emulated using QEMU
# NOTE: `${ARCH:+${ARCH}/}` - if ARCH is set, append `/` to it, leave it empty otherwise
FROM ${ARCH:+${ARCH}/}alpine:${VER_ALPINE} AS final

ARG VERSION
ARG USER
ARG DIR

LABEL maintainer="Liz (liz.ln@proton.me)"

RUN apk add --no-cache \
    libevent \
    libsodium \
    libstdc++ \
    libzmq \
    sqlite-libs

COPY  --from=builder /opt/bitcoin-$VERSION/bin/bitcoin*  /usr/local/bin/

# NOTE: Default GID == UID == 1000
RUN adduser --disabled-password \
    --home "$DIR/" \
    --gecos "" \
    "$USER"

USER $USER

# Prevents `VOLUME $DIR/.bitcoind/` being created as owned by `root`
RUN mkdir -p "$DIR/.bitcoin/"

# Expose volume containing all `bitcoind` data
VOLUME $DIR/.bitcoin/

# REST interface
EXPOSE 8080

# P2P network (mainnet, testnet & regnet, signet respectively)
EXPOSE 8333 18333 18444 38333

# RPC interface (mainnet, testnet & regnet, signet respectively)
EXPOSE 8332 18332 18443 38332

# ZMQ ports (for transactions & blocks respectively)
EXPOSE 28332 28333

ENTRYPOINT ["bitcoind"]

CMD ["-zmqpubrawblock=tcp://0.0.0.0:28332", "-zmqpubrawtx=tcp://0.0.0.0:28333"]
