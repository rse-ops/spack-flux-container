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
    &&   echo "  - openmpi@4.1.2 fabrics=ofi +legacylaunchers target=${spack_cpu_arch}" \
    &&   echo "  - libfabric@1.14.0 fabrics=efa,tcp,udp,sockets,verbs,shm,mrail,rxd,rxm target=${spack_cpu_arch}" \
    &&   echo "  - flux-sched target=${spack_cpu_arch}" \
    &&   echo "  - flux-pmix target=${spack_cpu_arch}" \
    &&   echo "  concretizer:" \
    &&   echo "    unify: true" \
    &&   echo "  config:" \
    &&   echo "    install_tree: /opt/software" \
    &&   echo "  view: /opt/view") > /opt/spack-environment/spack.yaml

# Install the software, remove unnecessary deps
RUN cd /opt/spack-environment \
    && git clone --single-branch --branch v0.19.0 https://github.com/spack/spack.git \
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
