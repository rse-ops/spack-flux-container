ARG tag="8"
FROM rockylinux:${tag} as builder

ENV DEBIAN_FRONTEND=noninteractive
ENV SPACK_ROOT=/opt/spack-environment/spack
ARG spack_cpu_arch=x86_64

#  Enable PowerTools for development packages
RUN yum -y update \
 && dnf -y install 'dnf-command(config-manager)' \
 && yum config-manager --set-enabled powertools \
 && yum -y update \
#  Enable EPEL
 && yum -y install epel-release \
#  Utilities
 && yum -y install \
	wget \
	man-db \
	less \
	xz \
	lbzip2 \
        fftw-devel \
        fftw \
	git \
	curl \
	bind-utils \
	sudo \
	munge \
	ccache \
	lua \
	valgrind \
	jq \
	which \
	file \
	unzip \
	vim \
	patch \
	diffutils \
#  Compilers, autotools
	pkgconfig \
	libtool \
	autoconf \
	automake \
	gcc-gfortran \
	gcc \
	gcc-c++ \
	make \
	cmake \
	bison \	
	flex \
#  Python
	python36 \
	python3-devel \
	python3-cffi \
	python3-six \
	python3-yaml \
	python3-jsonschema \
	python3-sphinx \
#  Development dependencies
	libsodium-devel \
        libyaml-devel \
        libgomp \
	zeromq-devel \
	czmq-devel \
	jansson-devel \
	munge-devel \
	ncurses-devel \
	lz4-devel \
	sqlite-devel \
        boost-devel \
	libuuid-devel \
	hwloc-devel \
	lua-devel \
        libedit-devel \
	valgrind-devel \
	libs3-devel \
	systemd-devel \
	libarchive-devel \
	pam-devel \
#  Other deps
        openssh-server \
        openssh-clients \
	perl-Time-HiRes \
	lua-posix \
	libfaketime \
	cppcheck \
	enchant \
	aspell \
	aspell-en \
	sudo \
	glibc-langpack-en \
	hwloc \
 && yum clean all

#  Set default /usr/bin/python to python3
RUN alternatives --set python /usr/bin/python3

ENV LANG=C.UTF-8
RUN printf "LANG=C.UTF-8" > /etc/locale.conf

# What we want to install and how we want to install it
# is specified in a manifest file (spack.yaml)
RUN mkdir /opt/spack-environment \
    &&  (echo "spack:" \
    &&   echo "  specs:" \
    &&   echo "  - intel-mpi@2021.8.0 target=${spack_cpu_arch}" \
    &&   echo "  - flux-sched target=${spack_cpu_arch}" \
    &&   echo "  - flux-core+mpi target=${spack_cpu_arch}" \
    &&   echo "  concretizer:" \
    &&   echo "    unify: true" \
    &&   echo "  packages:" \
    &&   echo "    intel-mpi:" \
    &&   echo "      externals:" \
    &&   echo "       - spec: intel-mpi@2021.8.0" \
    &&   echo "         prefix: /opt/intel/mpi/2021.8.0" \
    &&   echo "         buildable: false" \
    &&   echo "    all:" \
    &&   echo "      providers:"\
    &&   echo "        mpi:" \
    &&   echo "         - intel-mpi" \
    &&   echo "  config:" \
    &&   echo "    install_tree: /opt/software" \
    &&   echo "  view: /opt/view") > /opt/spack-environment/spack.yaml

WORKDIR /opt/spack-environment
RUN git clone https://github.com/spack/spack.git && \
    git clone https://github.com/GoogleCloudPlatform/hpc-tools && \
    cd hpc-tools/rocky_linux_8 && \
    chmod +x google_install_intelmpi && \
    # installs to /opt/intel
    ./google_install_intelmpi --impi_2021
    
# Install the software, remove unnecessary deps
RUN cd /opt/spack-environment && \
    cd spack && \
    # When flux-core 0.49.0 added, but flux-pmix 0.3.0 not added yet
    git checkout b32edd3a72e8c39d69dea4ab0dd2066d460913d5 && \
    mkdir -p /opt/spack-environment/spack/var/spack/repos/builtin/packages/flux-security 

# This has openmpi replaced with virtual package mpi
# Enable Intel(R) MPI 2021 Libraries with:
# source /opt/intel/mpi/latest/env/vars.sh 
COPY ./package.py /opt/spack-environment/spack/var/spack/repos/builtin/packages/flux-core/package.py 
COPY ./flux-security/package.py /opt/spack-environment/spack/var/spack/repos/builtin/packages/flux-security/package.py 
RUN . /opt/intel/mpi/latest/env/vars.sh && \
    cd /opt/spack-environment \
    && . spack/share/spack/setup-env.sh \
    && spack env activate . \
    && spack external find intel-mpi \
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
    && echo ". /opt/intel/mpi/latest/env/vars.sh" >> /etc/profile.d/z10_spack_environment.sh \
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
