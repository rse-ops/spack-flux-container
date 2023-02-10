FROM ubuntu:22.04 as builder

ARG spack_cpu_arch=zen3
ARG build_jobs=6

ARG DEBIAN_FRONTEND='noninteractive'
RUN apt-get update \
    && apt-get -y install \
    autotools-dev \
    autoconf \
    automake \
    cmake \
    git \
    python3 python3-pip python3-ipython\
    openssh-server \
    openssh-client \
    fftw3-dev fftw3 \
    libtool \
    apt-utils \
    software-properties-common \
    build-essential \
    gcc g++ gfortran \
    && apt-get clean \
    && apt-get autoremove 
 
# What we want to install and how we want to install it
# is specified in a manifest file (spack.yaml)
RUN mkdir /opt/spack-environment \
    &&  (echo "spack:" \
    &&   echo "  specs:" \
    &&   echo "  - openmpi@4.1.2 fabrics=ofi +legacylaunchers target=${spack_cpu_arch}" \
    &&   echo "  - libfabric@1.14.0 fabrics=efa,tcp,udp,sockets,verbs,shm,mrail,rxd,rxm target=${spack_cpu_arch}" \
    &&   echo "  - flux-sched target=${spack_cpu_arch}" \
    &&   echo "  - flux-pmix target=${spack_cpu_arch}" \
    &&   echo "  - pmix@4.1.2 target=${spack_cpu_arch}" \
    &&   echo "  - flux-core@master target=${spack_cpu_arch}"\
    &&   echo "  concretization: together" \
    &&   echo "  config:" \
    &&   echo "    install_tree: /opt/software" \
    &&   echo "  view: /opt/view") > /opt/spack-environment/spack.yaml

# I had done this manually to debug spack
# COPY ./spack /opt/spack-environment/spack

# Install the software, remove unnecessary deps
RUN cd /opt/spack-environment \
    # I commit changes here
    && git clone --branch manual-fixes-lammps-efa-jan-2023 https://github.com/flux-framework/spack.git

#COPY hwloc-package.py /opt/spack-environment/spack/var/spack/repos/builtin/packages/hwloc/package.py

RUN cd /opt/spack-environment \
    && . spack/share/spack/setup-env.sh \
    && spack env activate . \
    && spack compiler find \
    && spack external find openssh \
    && spack external find cmake

RUN DEBIAN_FRONTEND=noninteractive cd /opt/spack-environment \
    && . spack/share/spack/setup-env.sh \
    && spack env activate . \
    && spack spec flux-core@master \
    && spack install --reuse --fail-fast --no-checksum \
    && spack gc -y

# Strip all the binaries
RUN find -L /opt/view/* -type f -exec readlink -f '{}' \; | \
    xargs file -i | \
    grep 'charset=binary' | \
    grep 'x-executable\|x-archive\|x-sharedlib' | \
    awk -F: '{print $1}' | xargs strip -s

# Modifications to the environment that are necessary to run
RUN cd /opt/spack-environment \
    && . spack/share/spack/setup-env.sh \
    && spack env activate --sh -d . >> /etc/profile.d/z10_spack_environment.sh

ENV FLUX_OPTION_FLAGS="-ompi=openmpi@5"

# Make sure that flux owns the spack install (so we see it)
# We have to install python deps to system python because they are not
# seen in the spaaaaaaaaaaaaaaaaaaaaaaaaaack install
RUN adduser --disabled-password --uid 1000 --gecos "" flux && \
    chown -R flux /opt && \
    chmod -R +r /opt
RUN wget https://bootstrap.pypa.io/get-pip.py && \
    apt-get update && apt-get install -y python3-distutils && \
    python3 get-pip.py && \
    python3 -m pip install pyyaml jsonschema cffi


ENV FI_EFA_USE_DEVICE_RDMA=1
ENV RDMAV_FORK_SAFE=1
ENV DEBIAN_FRONTEND=noninteractive

# Install time for timed commands
RUN apt-get update && apt-get install -y time sudo \
    && apt-get clean && \
    apt-get autoremove && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

COPY ./etc.sudoers /etc/sudoers
COPY ./entrypoint.sh /entrypoint.sh

USER root
RUN git clone --depth 1 --branch stable_29Sep2021_update3 https://github.com/lammps/lammps.git /opt/lammps && \
    cd /opt/lammps && \
    mkdir build && \
    cd build && \
    . /etc/profile && \ 
    cmake ../cmake -D PKG_REAXFF=yes -D BUILD_MPI=yes -D PKG_OPT=yes -D FFT=FFTW3 && \
    make -j ${build_jobs} && \
    make install

# Create a home to put the examples in, and
# ensure lmp executable is on path for flux user
# And anyone can interact with lammps examples
RUN mv /opt/lammps/examples /home/flux/examples && \
    chown -R 1234 /home/flux/examples && \
    cp /root/.local/bin/lmp /usr/local/bin/lmp
    
# Target this example
ENV PATH=/root/.local/bin:$PATH
WORKDIR /home/flux/examples/reaxff/HNS
#ENTRYPOINT ["/bin/bash", "/entrypoint.sh"]
