# Copyright (c) 2022 Habana Labs, Ltd.
#
# SPDX-License-Identifier: Apache-2.0
#
# HabanaLabs Dockerfile base installer layer for RedHat 8.6
# Reference: https://github.com/HabanaAI/Setup_and_Install/blob/1.14.0/dockerfiles/base/Dockerfile.rhel8.6 
ARG BASE_IMAGE
FROM ${BASE_IMAGE}

ARG ARTIFACTORY_URL="vault.habana.ai"
ARG VERSION="1.14.0"
ARG REVISION="493"
ARG PT_VERSION="2.1.1"
ARG TF_VERSION="2.15.0"

USER root

## From: https://github.com/HabanaAI/Setup_and_Install/blob/1.14.0/dockerfiles/base/Dockerfile.rhel8.6
LABEL vendor="Habanalabs Ltd."
LABEL release="${VERSION}-${REVISION}"

COPY LICENSE /licenses/

RUN dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm && \
    dnf clean all && rm -rf /var/cache/yum

RUN echo "[appstream]" > /etc/yum.repos.d/CentOS-Linux-AppStream.repo && \
    echo "name=CentOS Linux 8 - AppStream" >> /etc/yum.repos.d/CentOS-Linux-AppStream.repo && \
    echo "mirrorlist=http://mirrorlist.centos.org/?release=\$releasever-stream&arch=\$basearch&repo=AppStream&infra=\$infra" >> /etc/yum.repos.d/CentOS-Linux-AppStream.repo && \
    echo "gpgcheck=0" >> /etc/yum.repos.d/CentOS-Linux-AppStream.repo


RUN echo "[BaseOS]" > /etc/yum.repos.d/CentOS-Linux-BaseOS.repo && \
    echo "name=CentOS Linux 8 - BaseOS" >> /etc/yum.repos.d/CentOS-Linux-BaseOS.repo && \
    echo "mirrorlist=http://mirrorlist.centos.org/?release=\$releasever-stream&arch=\$basearch&repo=BaseOS&infra=\$infra" >> /etc/yum.repos.d/CentOS-Linux-BaseOS.repo && \
    echo "gpgcheck=0" >> /etc/yum.repos.d/CentOS-Linux-BaseOS.repo

RUN dnf install -y \
    clang \
    cmake3 \
    cpp \
    gcc \
    gcc-c++ \
    glibc \
    glibc-headers \
    glibc-devel \
    jemalloc \
    libarchive \
    libksba \
    unzip \
    llvm \
    lsof \
    python38-devel \
    openssh-clients \
    libjpeg-devel \
    openssh-server \
    redhat-lsb-core \
    wget \
    git \
    mesa-libGL \
    iproute \
    python3-dnf-plugin-versionlock && \
    # update pkgs (except OS version) for resolving potentials CVEs
    dnf versionlock add redhat-release* && \
    dnf update -y && \
    dnf clean all && rm -rf /var/cache/yum

COPY install_efa.sh .
RUN ./install_efa.sh && rm install_efa.sh && rm -rf /etc/ld.so.conf.d/efa.conf /etc/profile.d/efa.sh

ENV LIBFABRIC_VERSION="1.20.0"
ENV LIBFABRIC_ROOT="/opt/habanalabs/libfabric-${LIBFABRIC_VERSION}"
ENV MPI_ROOT=/opt/amazon/openmpi
ENV LD_LIBRARY_PATH=$LIBFABRIC_ROOT/lib:${MPI_ROOT}/lib:/usr/lib/habanalabs:$LD_LIBRARY_PATH
ENV PATH=${LIBFABRIC_ROOT}/bin:${MPI_ROOT}/bin:$PATH
ENV OPAL_PREFIX=${MPI_ROOT}
ENV MPICC=${MPI_ROOT}/bin/mpicc
ENV RDMAV_FORK_SAFE=1
ENV FI_EFA_USE_DEVICE_RDMA=1

RUN echo "[habanalabs]" > /etc/yum.repos.d/habanalabs.repo && \
    echo "name=Habana RH8 Linux repo" >> /etc/yum.repos.d/habanalabs.repo && \
    echo "baseurl=https://${ARTIFACTORY_URL}/artifactory/rhel/8/8.6" >> /etc/yum.repos.d/habanalabs.repo && \
    echo "gpgkey=https://${ARTIFACTORY_URL}/artifactory/rhel/8/8.6/repodata/repomd.xml.key" >> /etc/yum.repos.d/habanalabs.repo

RUN echo "[powertools]" > /etc/yum.repos.d/powertools.repo && \
    echo "name=powertools" >> /etc/yum.repos.d/powertools.repo && \
    echo "baseurl=http://mirror.centos.org/centos/8-stream/PowerTools/x86_64/os/"  >> /etc/yum.repos.d/powertools.repo && \
    echo "gpgcheck=0" >> /etc/yum.repos.d/powertools.repo

RUN dnf install -y habanalabs-rdma-core-"$VERSION"-"$REVISION".el8 \
        habanalabs-thunk-"$VERSION"-"$REVISION".el8 \
        habanalabs-firmware-tools-"$VERSION"-"$REVISION".el8 \
        habanalabs-graph-"$VERSION"-"$REVISION".el8 && \
    rm -f /etc/yum.repos.d/habanalabs.repo && rm -rf /tmp/* && \
    dnf clean all && rm -rf /var/cache/yum

RUN rpm -V habanalabs-rdma-core && rpm -V habanalabs-thunk && rpm -V habanalabs-firmware-tools && rpm -V habanalabs-graph

# There is no need to store pip installation files inside docker image
ENV PIP_NO_CACHE_DIR=on
ENV PIP_DISABLE_PIP_VERSION_CHECK=1
ENV RDMA_CORE_ROOT=/opt/habanalabs/rdma-core/src
ENV RDMA_CORE_LIB=${RDMA_CORE_ROOT}/build/lib

RUN wget -nv -O /tmp/libfabric-${LIBFABRIC_VERSION}.tar.bz2 https://github.com/ofiwg/libfabric/releases/download/v${LIBFABRIC_VERSION}/libfabric-${LIBFABRIC_VERSION}.tar.bz2 && \
    cd /tmp/ && tar xf /tmp/libfabric-${LIBFABRIC_VERSION}.tar.bz2 && \
    cd /tmp/libfabric-${LIBFABRIC_VERSION} && \
    ./configure --prefix=$LIBFABRIC_ROOT --enable-psm3-verbs --enable-verbs=yes --with-synapseai=/usr && \
    make && make install

ENV LIBOFI_VERSION="0.0.1"
RUN wget -nv -O /tmp/v${LIBOFI_VERSION}.tar.gz https://github.com/HabanaAI/hccl_ofi_wrapper/archive/refs/tags/v${LIBOFI_VERSION}.tar.gz && \
    cd /tmp/ && tar xf /tmp/v${LIBOFI_VERSION}.tar.gz && \
    cd /tmp/hccl_ofi_wrapper-${LIBOFI_VERSION} && \
    make && cp -f libhccl_ofi_wrapper.so /usr/lib/habanalabs/libhccl_ofi_wrapper.so && \
    cd / && \
    rm -rf /tmp/v${LIBOFI_VERSION}.tar.gz /tmp/hccl_ofi_wrapper-${LIBOFI_VERSION}

RUN python3.8 -m pip install pip==23.3.1 setuptools==67.3.3 wheel==0.38.4

RUN alternatives --install /usr/bin/python3 python3 /usr/bin/python3.8 2 && \
    alternatives --install /usr/bin/python3 python3 /usr/bin/python3.6 1 && \
    alternatives --set python3 /usr/bin/python3.8

RUN python3.8 -m pip install habana_media_loader=="${VERSION}"."${REVISION}"

# SSH configuration necessary to support mpi-operator v2
RUN mkdir -p /var/run/sshd && \
    sed -i 's/[ #]\(.*StrictHostKeyChecking \).*/ \1no/g' /etc/ssh/ssh_config && \
    sed -i 's/#\(ForwardAgent \).*/\1yes/g' /etc/ssh/ssh_config && \
    echo "    UserKnownHostsFile /dev/null" >> /etc/ssh/ssh_config && \
    sed -i 's/#\(StrictModes \).*/\1no/g' /etc/ssh/sshd_config && \
    ssh-keygen -A && \
    mkdir -p /var/run/sshd && echo "/usr/sbin/sshd -p 3022" | tee -a ~/.bashrc

RUN echo "export LANG=en_US.UTF-8" >> /root/.bashrc
RUN export LANG=en_US.UTF-8
ENV GC_KERNEL_PATH=/usr/lib/habanalabs/libtpc_kernels.so
ENV HABANA_LOGS=/var/log/habana_logs/
ENV HABANA_SCAL_BIN_PATH=/opt/habanalabs/engines_fw
ENV HABANA_PLUGINS_LIB_PATH=/opt/habanalabs/habana_plugins





####### From: https://github.com/HabanaAI/Setup_and_Install/blob/1.14.0/dockerfiles/pytorch/Dockerfile.rhel8.6
LABEL name="PyTorch Installer"
LABEL summary="Habanalabs PyTorch installer layer for RHEL8.6"
LABEL description="Image with pre installed Habanalabs packages for PyTorch"

ENV LANG=en_US.UTF-8
ENV PYTHONPATH=/root:/usr/lib/habanalabs/

RUN dnf install -y \
    curl \
    cairo-devel \
    numactl-devel \
    iproute \
    which \
    zlib-devel \
    lapack-devel \
    openblas-devel \
    numactl \
    gperftools-devel && \
    dnf clean all && rm -rf /var/cache/yum

COPY install_packages.sh .

RUN export BASE_NAME=rhel86 && ./install_packages.sh && rm -f install_packages.sh && \
    /sbin/ldconfig && echo "source /etc/profile.d/habanalabs.sh" >> ~/.bashrc

ENV LD_PRELOAD=/lib64/libtcmalloc.so.4
ENV TCMALLOC_LARGE_ALLOC_REPORT_THRESHOLD=7516192768

RUN rm -rf /tmp/*

####### From: https://github.com/HabanaAI/Setup_and_Install/blob/1.14.0/dockerfiles/tensorflow/Dockerfile.rhel8.6
LABEL name="Tensorflow Installer (${TF_VERSION})"
LABEL summary="Habanalabs Tensorflow (${TF_VERSION}) installer layer for RHEL 8.6"
LABEL description="Image with pre installed Habanalabs packages for Tensorflow-${TF_VERSION}"

ENV TF_MODULES_RELEASE_BUILD=/usr/lib/habanalabs/
ENV PYTHONPATH=/root:/usr/lib/habanalabs/

# Install unzip to extract pre-trained weights for BERT demo
RUN dnf install -y \
        bc \
        protobuf-devel \
        libffi-devel \
        bzip2-devel && \
    dnf clean all && rm -rf /var/cache/dnf

COPY install-python310.sh install-python310.sh
RUN export BASE_NAME=rhel86 && ./install-python310.sh $BASE_NAME
ENV LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH

RUN python3.10 -m pip install habana_media_loader=="${VERSION}"."${REVISION}" && \
    python3.10 -m pip install tensorflow-cpu==${TF_VERSION} && \
    python3.10 -m pip install habana-tensorflow=="${VERSION}"."${REVISION}" && \
    python3.10 -m pip install habana-horovod=="${VERSION}"."${REVISION}"

# For AML/CentOS/RHEL OS'es TFIO_DATAPATH have to be specified to import tensorflow_io lib correctly
ENV TFIO_DATAPATH=/usr/local/lib64/python3.10/site-packages/

# For AML/CentOS/RHEL ca-cert file is expected exactly under /etc/ssl/certs/ca-certificates.crt
# otherwise curl will fail during access to S3 AWS storage
RUN ln -s /etc/ssl/certs/ca-bundle.crt /etc/ssl/certs/ca-certificates.crt

RUN echo "source /etc/profile.d/habanalabs.sh" >> ~/.bashrc && \
    dnf clean all && rm -rf /var/cache/dnf && rm -rf /tmp/*
