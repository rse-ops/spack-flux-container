ARG tag="20.04"
FROM ubuntu:${tag} as builder

ENV DEBIAN_FRONTEND=noninteractive
ENV SPACK_ROOT=/opt/spack-environment/spack
ARG spack_cpu_arch=zen3
ARG build_jobs=6

RUN apt-get update && \
    apt-get -y install \
    apt-utils \
    autotools-dev \
    autoconf \
    automake \
    build-essential \
    ca-certificates \
    cmake \
    curl \
    dnsutils \
    fftw3-dev fftw3 \
    gfortran \
    git \
    libyaml-cpp-dev \
    libedit-dev \
    libnuma-dev \
    libgomp1 \
    libboost-graph-dev \
    libboost-system-dev \
    libboost-filesystem-dev \
    libboost-regex-dev \
    munge \
    pkg-config \
    openssh-server \
    openssh-client \
    python-yaml \
    python3-jsonschema \
    python3-pip \
    python3-cffi \
    python3 \
    pdsh \
    sudo \
    unzip \
    wget \
    && apt-get clean \
    && apt-get autoremove \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# What we want to install and how we want to install it
# is specified in a manifest file (spack.yaml)
#RUN mkdir /opt/spack-environment \
#    &&  (echo "spack:" \
#    &&   echo "  specs:" \
#    &&   echo "  - openmpi@4.1.2 fabrics=ofi +legacylaunchers target=${spack_cpu_arch}" \
#    &&   echo "  - libfabric@1.14.0 fabrics=efa,tcp,udp,sockets,verbs,shm,mrail,rxd,rxm target=${spack_cpu_arch}" \
#    &&   echo "  - flux-sched target=${spack_cpu_arch}" \
#    &&   echo "  - flux-pmix target=${spack_cpu_arch}" \
#    &&   echo "  concretizer:" \
#    &&   echo "    unify: when_possible" \
#    &&   echo "  config:" \
#    &&   echo "    install_tree: /opt/software" \
#    &&   echo "  view: /opt/view") > /opt/spack-environment/spack.yaml

# What we want to install and how we want to install it
# is specified in a manifest file (spack.yaml)
#RUN mkdir /opt/spack-environment \
    &&  (echo "spack:" \
    &&   echo "  specs:" \
    &&   echo "  - openmpi@4.1.2 fabrics=ofi +legacylaunchers target=${spack_cpu_arch}" \
    &&   echo "  - libfabric@1.14.0 fabrics=efa,tcp,udp,sockets,verbs,shm,mrail,rxd,rxm target=${spack_cpu_arch}" \
    &&   echo "  concretization: together" \
    &&   echo "  config:" \
    &&   echo "    install_tree: /opt/software" \
    &&   echo "  view: /opt/view") > /opt/spack-environment/spack.yaml

# Install the software, remove unnecessary deps
RUN cd /opt/spack-environment \
    && git clone https://github.com/spack/spack.git \
    && cd spack \
    && git checkout 7893be7712ed709f6136ac83f49afc3d718d5ddc \
    && cd ../ \
    && . spack/share/spack/setup-env.sh \
    && pip3 install clingo \
    #&& spack install clingo target=${spack_cpu_arch} \
    && spack env activate . \
    && spack external find openssh \
    && spack external find cmake \
    && spack external find pkg-config \
    #&& spack external find python \
    && spack install --fail-fast --no-cache \
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

# Important! The versions of flux with pmix won't work without this flag
# the flux-restful api added support via version 0.0.15 of the python client
ENV FLUX_OPTION_FLAGS="-ompi=openmpi@5"

# Make sure that flux owns the spack install (so we see it)
# We have to install python deps to system python because they are not
# seen in the spaaaaaaaaaaaaaaaaaaaaaaaaaack install
RUN sudo adduser --disabled-password --uid 1000 --gecos "" flux && \
    chown -R flux /opt && \
    sudo chmod -R +r /opt && \
    sudo apt-get install python3-distutils && \
    wget https://bootstrap.pypa.io/get-pip.py && \
    sudo python3 -m pip install pyyaml jsonschema cffi

COPY ./etc.sudoers /etc/sudoers
COPY ./entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/bin/bash", "/entrypoint.sh"]
