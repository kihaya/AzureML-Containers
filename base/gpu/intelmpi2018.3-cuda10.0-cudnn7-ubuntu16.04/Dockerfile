# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT License.

FROM mcr.microsoft.com/azureml/o16n-base/python-assets@sha256:f609724dddd9ceacf418667053b5167641d8f2ea41d81686463a3fbd9e9bfd7e AS inferencing-assets

# Tag: cuda:10.0-cudnn7-devel-ubuntu16.04
# Env: CUDA_VERSION=10.0.130
# Env: CUDA_PKG_VERSION=10-0=10.0.130-1
# Env: NCCL_VERSION=2.4.8
# Env: CUDNN_VERSION=7.6.3.30
# Env: NVIDIA_VISIBLE_DEVICES=all
# Env: NVIDIA_DRIVER_CAPABILITIES=compute,utility
# Env: NVIDIA_REQUIRE_CUDA=cuda>=10.0 brand=tesla,driver>=384,driver<385 brand=tesla,driver>=410,driver<411
# Label: com.nvidia.cuda.version=10.0.130
# Label: com.nvidia.cudnn.version=7.6.3.30
# Label: com.nvidia.volumes.needed=nvidia_driver
# Ubuntu 16.04
FROM nvidia/cuda@sha256:27f07b36cf5dbf9c148ade5795e53932c421c9526143afc83e4f9dc55fb618fd

USER root:root
ENV com.nvidia.cuda.version $CUDA_VERSION
ENV com.nvidia.volumes.needed nvidia_driver

ENV LANG=C.UTF-8 LC_ALL=C.UTF-8
ENV DEBIAN_FRONTEND noninteractive
ENV LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/lib64:/usr/local/cuda/extras/CUPTI/lib64

# Install Common Dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    # SSH and RDMA
    libmlx4-1 \
    libmlx5-1 \
    librdmacm1 \
    libibverbs1 \
    libmthca1 \
    libdapl2 \
    dapl2-utils \
    openssh-client \
    openssh-server \
    iproute2 && \
    # Others
    apt-get install -y \
    build-essential \
    bzip2 \
    libbz2-1.0 \
    systemd \
    git \
    wget \
    cpio \
    libsm6 \
    libxext6 \
    libxrender-dev \
    fuse && \
    apt-get clean -y && \
    rm -rf /var/lib/apt/lists/*

# Inference
# Copy logging utilities, nginx and rsyslog configuration files, IOT server binary, etc.
COPY --from=inferencing-assets /artifacts /var/
RUN /var/requirements/install_system_requirements.sh && \
    cp /var/configuration/rsyslog.conf /etc/rsyslog.conf && \
    cp /var/configuration/nginx.conf /etc/nginx/sites-available/app && \
    ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/app && \
    rm -f /etc/nginx/sites-enabled/default
ENV SVDIR=/var/runit
ENV WORKER_TIMEOUT=300
EXPOSE 5001 8883 8888

# Conda Environment
ENV MINICONDA_VERSION 4.7.12.1
ENV PATH /opt/miniconda/bin:$PATH
RUN wget -qO /tmp/miniconda.sh https://repo.continuum.io/miniconda/Miniconda3-${MINICONDA_VERSION}-Linux-x86_64.sh && \
    bash /tmp/miniconda.sh -bf -p /opt/miniconda && \
    conda clean -ay && \
    rm -rf /opt/miniconda/pkgs && \
    rm /tmp/miniconda.sh && \
    find / -type d -name __pycache__ | xargs rm -rf

# Intel MPI installation
ENV INTEL_MPI_VERSION 2018.3.222
ENV PATH $PATH:/opt/intel/compilers_and_libraries/linux/mpi/bin64
RUN cd /tmp && \
    wget -q "http://registrationcenter-download.intel.com/akdlm/irc_nas/tec/13063/l_mpi_${INTEL_MPI_VERSION}.tgz" && \
    tar zxvf l_mpi_${INTEL_MPI_VERSION}.tgz && \
    sed -i -e 's/^ACCEPT_EULA=decline/ACCEPT_EULA=accept/g' /tmp/l_mpi_${INTEL_MPI_VERSION}/silent.cfg && \
    cd /tmp/l_mpi_${INTEL_MPI_VERSION} && \
    ./install.sh -s silent.cfg --arch=intel64 && \
    cd / && \
    rm -rf /tmp/l_mpi_${INTEL_MPI_VERSION}* && \
    rm -rf /opt/intel/compilers_and_libraries_${INTEL_MPI_VERSION}/linux/mpi/intel64/lib/debug* && \
    echo "source /opt/intel/compilers_and_libraries_${INTEL_MPI_VERSION}/linux/mpi/intel64/bin/mpivars.sh" >> ~/.bashrc

# Msodbcsql17 installation
RUN apt-get update && \
    apt-get install -y curl && \
    curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - && \
    curl https://packages.microsoft.com/config/ubuntu/16.04/prod.list > /etc/apt/sources.list.d/mssql-release.list && \
    apt-get update && \
    ACCEPT_EULA=Y apt-get install -y msodbcsql17
