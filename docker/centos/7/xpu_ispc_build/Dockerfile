#
#  Copyright (c) 2017-2023, Intel Corporation
#
#  SPDX-License-Identifier: BSD-3-Clause

FROM centos:7
LABEL maintainer="Dmitry Babokin <dmitry.y.babokin@intel.com>"
SHELL ["/bin/bash", "-c"]

ARG REPO=ispc/ispc
ARG SHA=main
ARG LLVM_VERSION=16.0
ARG L0L_VER=1.14.0
ARG VC_INTRINSICS_COMMIT_SHA="77f069b71fa9042bc4b2f68d06942a43c8ecec73"
ARG SPIRV_TRANSLATOR_COMMIT_SHA="8703d43bd14d000fc630c2d3918d918819d23741"
ARG TBB=default

# !!! Make sure that your docker config provides enough memory to the container,
# otherwise LLVM build may fail, as it will use all the cores available to container.

# Packages required to build ISPC and Clang.
RUN yum -y update; yum -y install centos-release-scl epel-release; yum clean all
RUN yum install -y vim wget yum-utils gcc gcc-c++ git python3 m4 bison flex patch make && \
    yum install -y glibc-devel.x86_64 glibc-devel.i686 xz devtoolset-8 && \
    yum install -y libtool autopoint gettext-devel texinfo help2man && \
    yum clean -y all

# TBB is parameter to build both packge usual and oneapi-tbb.
RUN if [ "$TBB" == "default" ]; then  yum -y install tbb tbb-devel && yum clean -y all; fi
COPY oneAPI.repo /etc/yum.repos.d/oneAPI.repo
RUN if [ "$TBB" == "oneapi" ]; then yum -y install intel-oneapi-tbb-devel intel-oneapi-tbb && yum clean -y all; fi

# These packages are required if you need to link ISPC with -static.
RUN yum install -y libstdc++-static && \
    yum clean -y all

# Download and install required version of CMake. CMake 3.20 is required starting from LLVM 16.0.
RUN wget -q --retry-connrefused --waitretry=5 --read-timeout=20 --timeout=15 -t 5 https://github.com/Kitware/CMake/releases/download/v3.20.5/cmake-3.20.5-linux-x86_64.sh && \
    sh cmake-3.20.5-linux-x86_64.sh --prefix=/usr/local --skip-license && rm cmake-3.20.5-linux-x86_64.sh

# If you are behind a proxy, you need to configure git.
RUN if [ -v http_proxy ]; then git config --global --add http.proxy "$http_proxy"; fi

WORKDIR /usr/local/src

# Fork ispc on github and clone *your* fork.
RUN mkdir /usr/local/src/llvm && \
    git clone https://github.com/$REPO.git ispc

# This is home for Clang builds
ENV LLVM_HOME=/usr/local/src/llvm
ENV ISPC_HOME=/usr/local/src/ispc
ENV XE_DEPS=/usr/local/deps

# If you are going to run test for future platforms, go to
# http://www.intel.com/software/sde and download the latest version,
# extract it, add to path and set SDE_HOME.

WORKDIR /usr/local/src/ispc

# Build Clang with all required patches.
# Pass required LLVM_VERSION with --build-arg LLVM_VERSION=<version>.
# Note self-build options, it's required to build clang and ispc with the same compiler,
# i.e. if clang was built by gcc, you may need to use gcc to build ispc (i.e. run "make gcc"),
# or better do clang selfbuild and use it for ispc build as well (i.e. just "make").
# "rm" are just to keep docker image small.
RUN git checkout $SHA && \
    source /opt/rh/devtoolset-8/enable && \
    ./alloy.py -b --version="$LLVM_VERSION" --selfbuild --llvm-disable-assertions --full-checkout && \
    rm -rf "$LLVM_HOME"/build-"$LLVM_VERSION" "$LLVM_HOME"/llvm-"$LLVM_VERSION" "$LLVM_HOME"/bin-"$LLVM_VERSION"_temp "$LLVM_HOME"/build-"$LLVM_VERSION"_temp

ENV PATH=$LLVM_HOME/bin-$LLVM_VERSION/bin:$PATH

# Build ncurses with -fPIC: https://github.com/ispc/ispc/pull/2502#issuecomment-1526931698
WORKDIR /usr/local/src
RUN git clone https://github.com/mirror/ncurses.git
WORKDIR /usr/local/src/ncurses
# Checkout version with fix for CVE-2023-29491
RUN git checkout 87c2c84 && CFLAGS="-fPIC" CXXFLAGS="-fPIC" ./configure --with-termlib && make -j"$(nproc)" && make install -j"$(nproc)"

# Install new flex (2.6.4)
WORKDIR /usr/local/src
RUN git clone https://github.com/westes/flex.git
WORKDIR /usr/local/src/flex
RUN git checkout v2.6.4 && ./autogen.sh && ./configure && make -j"$(nproc)" && make install

# vc-intrinsics
WORKDIR /usr/local/src
RUN git clone https://github.com/intel/vc-intrinsics.git src && \
    git --git-dir=src/.git --work-tree=src checkout $VC_INTRINSICS_COMMIT_SHA && \
    cmake src \
        -B build \
        -DCMAKE_CXX_COMPILER=clang++ \
        -DCMAKE_C_COMPILER=clang \
        -DLLVM_DIR=$LLVM_HOME/bin-$LLVM_VERSION/lib/cmake/llvm \
        -DCMAKE_INSTALL_PREFIX=$XE_DEPS && \
    cmake --build build -j"$(nproc)" && \
    cmake --install build && \
    rm -rf build src

# SPIRV Translator
RUN git clone https://github.com/KhronosGroup/SPIRV-LLVM-Translator.git src && \
    git --git-dir=src/.git --work-tree=src checkout $SPIRV_TRANSLATOR_COMMIT_SHA && \
    cmake src \
        -B build \
        -DCMAKE_CXX_COMPILER=clang++ \
        -DCMAKE_C_COMPILER=clang \
        -DLLVM_DIR=$LLVM_HOME/bin-$LLVM_VERSION/lib/cmake/llvm/ \
        -DCMAKE_INSTALL_PREFIX=$XE_DEPS && \
    cmake --build build -j"$(nproc)" && \
    cmake --install build && \
    rm -rf build src

# L0
RUN git clone https://github.com/oneapi-src/level-zero.git src && \
    git --git-dir=src/.git --work-tree=src checkout v$L0L_VER && \
    cmake src \
        -B build \
        -DCMAKE_CXX_COMPILER=clang++ \
        -DCMAKE_C_COMPILER=clang \
        -DCMAKE_INSTALL_PREFIX=/usr/local && \
    cmake --build build -j"$(nproc)" && \
    cmake --install build && \
    rm -rf build src

# Build ISPC
ENV LD_LIBRARY_PATH=$LLVM_HOME/bin-$LLVM_VERSION/lib:$LD_LIBRARY_PATH
WORKDIR /usr/local/src/ispc
RUN cmake . \
        -B build \
        -DCMAKE_INSTALL_PREFIX=/usr/local/src/ispc/bin-$LLVM_VERSION \
        -DISPC_PREPARE_PACKAGE=ON \
        -DISPC_CROSS=ON \
        -DXE_ENABLED=ON \
        -DXE_DEPS_DIR=$XE_DEPS && \
    cmake --build build --target package -j"$(nproc)" && \
    (cmake --build build --target check-all || true) && \
    cmake --install build && \
    mv build/*.tar.gz ./ && \
    rm -rf build

# Export path for ispc
ENV PATH=/usr/local/src/ispc/bin-$LLVM_VERSION/bin:$PATH
