# Originally forked from https://github.com/seanmorris/php-wasm
# ubuntu:lunar supports amd64 and arm64 (Apple Silicon) while
# emscripten/emsdk:3.1.24 supports amd64 only.
FROM ubuntu:lunar as emscripten

SHELL ["/bin/bash", "-c"]

ENV PKG_CONFIG_PATH /root/lib/lib/pkgconfig
ENV TIMER "(which pv > /dev/null && pv --name '${@}' || cat)"

WORKDIR /root
RUN mkdir lib

RUN set -euxo pipefail;\
    apt-get update; \
    apt-get --no-install-recommends -y install \
    build-essential \
    automake \
    autoconf \
    libxml2-dev \
    libtool \
    pkgconf \
    flex \
    make \
    re2c \
    gdb \
    git \
    pv \
    ca-certificates \
    curl \
    wget \
    unzip \
    cmake \
    python3

# Install Emscripten from the repository. We'd use the official
# Docker image, but there is no arm64 image available which makes
# the build take forever on Apple Silicon.
RUN ln -s /usr/bin/python3 /usr/bin/python
RUN git clone https://github.com/emscripten-core/emsdk.git && \
    ./emsdk/emsdk install 3.1.43 && \
    /root/emsdk/emsdk activate 3.1.43

RUN mkdir -p /root/lib/lib /root/lib/include /root/lib/share /root/lib/bin

# Create a script to capture libraries build in other steps
RUN echo $'#!/bin/bash\n\
for dir in $(ls $1); do \n\
    mkdir -p /root/lib/$dir/ && \n\
    cp -r /root/$1/$dir/* /root/lib/$dir/ 2>/dev/null || :; \n\
    done ' > /root/copy-lib.sh
RUN chmod a+x /root/copy-lib.sh
# Script to replace strings in files and exit with a non-zero status if the string is not found.
COPY ./replace.sh /root/replace.sh
RUN chmod a+x /root/replace.sh

# Patch emcc to allow skipping flags and passing additional flags using environment variables.
#
# We're compiling libraries statically using emscripten's -sSIDE_MODULE. It differs from the usual unix
# process as we're *not* supposed to link intermediate libraries against their dependencies. Instead,
# We'll provide all the static libraries in the final linking step.
#
# Unfortunately, most Makefiles stubbornly add -lz, -lxml2, etc. to the intermediate
# emcc calls, which causes the build to fail. This patch:
# * Enables force-removing linker options when running emcc
# * Enables adding -sSIDE_MODULE when running emcc
#
# Usage:
#  EMCC_SKIP="-lz -lxml2" EMCC_FLAGS="-sSIDE_MODULE" emmake make
#               ↑                            ↑
#    ┌──────────┴─────────────┐   ┌──────────┴─────────────┐
#    │  skip those flags when │   │  add these flags when  │
#    │      calling emcc      │   │    calling emcc        │
#    └────────────────────────┘   └────────────────────────┘
RUN cp /root/emsdk/upstream/emscripten/emcc /root/emsdk/upstream/emscripten/emcc2 && \
    cp /root/emsdk/upstream/emscripten/emcc.py /root/emsdk/upstream/emscripten/emcc2.py && \
    echo $'#!/bin/bash\n\
for arg do shift\n\
    [[ " ${EMCC_SKIP[*]} " =~ " ${arg} " ]] && continue \n\
    set -- "$@" "$arg" \n\
done\n\
# Passing extra flags breaks the version check \n\
if [[ "$@" == "-v" ]]; then\n\
    export EMCC_FLAGS=""\n\
fi\n\
/root/emsdk/upstream/emscripten/emcc2 "$@" $EMCC_FLAGS \n' > /root/emsdk/upstream/emscripten/emcc && \
    chmod a+x /root/emsdk/upstream/emscripten/emcc

