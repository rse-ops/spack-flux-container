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
    &&   echo "  - mpich target=${spack_cpu_arch}" \
    &&   echo "  - flux-sched target=${spack_cpu_arch}" \
    &&   echo "  - flux-pmix target=${spack_cpu_arch}" \
    &&   echo "  concretizer:" \
    &&   echo "    unify: true" \
    &&   echo "  config:" \
    &&   echo "    install_tree: /opt/software" \
    &&   echo "  view: /opt/view") > /opt/spack-environment/spack.yaml

# Install the software, remove unnecessary deps
RUN cd /opt/spack-environment \
    && git clone -b develop https://github.com/spack/spack.git \
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
