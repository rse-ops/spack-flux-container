ARG tag=22.04
FROM ubuntu:${tag}
ARG spack_cpu_arch=x86_64

# see http://hpctoolkit.org/software-instructions.html
RUN apt-get update && apt-get install -y \
    git \
    file \
    build-essential \
    python3 \
    python3-dev \
    python3-cffi \
    python3-six \
    python3-yaml \
    python3-jsonschema \
    python3-sphinx \
    openssh-server \
    openssh-client \
    gfortran \
    cmake \
    bzip2 \
    g++ gcc \
    diffutils \
    python3-dev \
    curl \
    wget \
    unzip

ENV LANG=C.UTF-8

#  Set default /usr/bin/python to python3
RUN printf "LANG=C.UTF-8" > /etc/locale.conf && \
    alternatives --set python /usr/bin/python3

# What we want to install and how we want to install it
# is specified in a manifest file (spack.yaml)
RUN mkdir /opt/spack-environment \
    &&  (echo "spack:" \
    &&   echo "  specs:" \
    &&   echo "  - mpich target=${spack_cpu_arch}" \
    &&   echo "  - flux-sched target=${spack_cpu_arch}" \
    &&   echo "  - flux-pmix target=${spack_cpu_arch}" \
    &&   echo "  concretizer:" \
    &&   echo "    unify: true" \
    &&   echo "  config:" \
    &&   echo "    install_tree: /opt/software" \
    &&   echo "  view: /opt/view") > /opt/spack-environment/spack.yaml

WORKDIR /opt/spack-environment
RUN git clone https://github.com/spack/spack.git

# This has openmpi replaced with virtual package mpi
COPY ./package.py ./var/spack/repos/builtin/packages/flux-pmix/package.py 

# Install the software, remove unnecessary deps
RUN cd /opt/spack-environment \
    && cd spack \
    # When flux-core 0.49.0 added, but flux-pmix 0.3.0 not added yet
    && git checkout b32edd3a72e8c39d69dea4ab0dd2066d460913d5 \
    && cd ../ \
    && . spack/share/spack/setup-env.sh \
    && spack env activate . \
    && spack external find openssh \
    && spack external find cmake \
    && spack external find python \
    && spack install --reuse --fail-fast \
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
RUN sudo adduser --password flux --uid 1000 flux && \
    chown -R flux /opt && \
    sudo chmod -R +r /opt

COPY ./etc.sudoers /etc/sudoers
COPY ./entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/bin/bash", "/entrypoint.sh"]
