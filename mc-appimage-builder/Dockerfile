
# === First step ===
# is to backport from Ubuntu 16.04 and rebuild 'gnutls' library and it's dependencies.
# GNUTLS will be patched by this patch:
# https://github.com/darealshinji/vlc-AppImage/issues/1#issuecomment-321041496
# which allows to search root CA certificates in different locations
# to allow the resulting AppImage to be portable between Linux distributions.
# This tries to solve known Linux platfom issue:
# https://gitlab.com/probono/platformissues#certificates

FROM ubuntu:14.04 as gnutls

MAINTAINER Alexander Sashnov "sashnov@ngs.ru"

# "FROM ... as gnutls" - 'gnutls' is a name of intermediate image
# that will not be a part of the resulting image. We will just
# COPY --from=gnutls ...
# the needed .so libraries only, left behind all the extracted sources and patches.

ENV DEBIAN_FRONTEND noninteractive

ENV HOME    /home/ubuntu
WORKDIR     /home/ubuntu

RUN apt update && apt full-upgrade -y && apt install -y --no-install-recommends \
    build-essential \
    fakeroot \
    devscripts

# RUN apt-get build-dep libgnutls-openssl27
#
# That will not work. Repository version of libgnutls28 requires too new
# debhelper and other packages. Also, we are going to turn off some of the components
# so we do not need to install those dependencies
#
# Here are the difference between stock Ubuntu 16.04 gnutls28 and our build:
# - Do not use nettle-dev (shiped with gnutls copy of nettle-mini lib will be used)
# - do not require the latest debhelper
# - without guile
# - without libidn
# - without libopts25-dev (the comment in debian/control says it's optional)
# - without gtk-doc-tools, texinfo (we will not build the documentation)
RUN apt install -y --no-install-recommends \
    autogen \
    automake \
    autotools-dev \
    bison \
    chrpath \
    datefudge \
    debhelper \
    dh-autoreconf \
    dpkg-dev \
    libffi-dev \
    libopts25-dev \
    libp11-kit-dev \
    libtasn1-6-dev \
    nettle-dev \
    pkg-config \
    zlib1g-dev

# Getting libgnutls-3.4 sources from Ubuntu 16.04 repository:
RUN /bin/echo -e "\ndeb http://archive.ubuntu.com/ubuntu/ xenial-updates main\ndeb-src http://archive.ubuntu.com/ubuntu/ xenial-updates main\n" >> /etc/apt/sources.list

RUN apt-get update && apt-get source libp11-kit-dev nettle-dev libgnutls-openssl27

RUN cd nettle-3.2 \
    && dpkg-buildpackage -b -d

COPY p11-kit-strip-and-backport.patch /home/ubuntu

# Just downgrading build dependency P11_KIT_MINIMUM 0.23 => 0.20.0 doesn't work:
#    In file included from pkcs11.c:39:0:
#    pkcs11x.h:24:29: fatal error: p11-kit/pkcs11x.h: No such file or directory

RUN cd p11-kit-0.23.2 \
    && patch -p1 < ../p11-kit-strip-and-backport.patch \
    && dpkg-buildpackage -b -d

# Install nettle and p11-kit:
RUN dpkg -i *.deb

COPY gnutls-multiple-root-cert-locations.patch /home/ubuntu

COPY gnutls-strip-and-backport.patch /home/ubuntu

RUN cd gnutls28-3.4.10 \
    && patch -p1 < ../gnutls-multiple-root-cert-locations.patch \
    && patch -p1 < ../gnutls-strip-and-backport.patch \
    && dpkg-buildpackage -b -d

# remove conflicting packages
RUN apt-get remove -y libhogweed2 libgnutls-openssl27 iputils-ping ubuntu-minimal libnettle4

RUN dpkg -i libgnutls-openssl27_*.deb libgnutlsxx28_*.deb gnutls-bin_*.deb libgnutls30_*.deb



# === Second step ===
#
# Build an image with Official Qt installer.
# The code to run the official Qt installer is based on:
#   https://github.com/rabits/dockerfiles
# (automation the installer with custom Java scripts).
#
# Another interesting approach is a complete implementation of what
# the Qt installer does purely in bash scripts:
#  https://raw.githubusercontent.com/probonopd/linuxdeployqt/feature/upstream-qt/qtdownload
# (but we will not use it here).

FROM ubuntu:14.04

MAINTAINER Alexander Sashnov "sashnov@ngs.ru"

ENV DEBIAN_FRONTEND noninteractive

ENV QT_VERSION  5.9.6
ENV QT_PATH /opt/Qt
ENV QT_DESKTOP $QT_PATH/${QT_VERSION}/gcc_64

ENV PATH    $QT_DESKTOP/bin:$PATH
ENV HOME    /home/ubuntu

# WORKDIR directive will create the directory automatically
WORKDIR     /home/ubuntu

# Install updates & requirements:
#  * git, openssh-client, ca-certificates - clone & build
#  * locales, sudo - useful to set utf-8 locale & sudo usage
#  * curl - to download Qt bundle
#  * build-essential, pkg-config, libgl1-mesa-dev - basic Qt build requirements
#  * libsm6, libice6, libxext6, libxrender1, libfontconfig1, libdbus-1-3 - dependencies of the Qt bundle run-file
#  * libcairo2 needed for running appimagetool
#  * libusb-1.0-0-dev needed by Moolticute daemon
RUN apt update && apt full-upgrade -y && apt install -y --no-install-recommends \
    git \
    openssh-client \
    ca-certificates \
    locales \
    sudo \
    curl \
    build-essential \
    pkg-config \
    libcairo2 \
    libegl1-mesa \
    libfuse2 \
    libgl1-mesa-dev \
    libsm6 \
    libice6 \
    libxext6 \
    libxrender1 \
    libfontconfig1 \
    libdbus-1-3 \
    libusb-1.0-0-dev \
    libxi6


COPY extract-qt-installer.sh /tmp/qt/

# Download & unpack Qt toolchains & clean
# Note curl -4 option: this is workaround for 'unable to resolve host' in ubuntu:14.04 container
RUN curl -4 -Lo /tmp/qt/installer.run "https://download.qt.io/official_releases/online_installers/qt-unified-linux-x64-online.run" \
    && QT_CI_PACKAGES=qt.$(echo "${QT_VERSION}" | tr -d .).gcc_64 /tmp/qt/extract-qt-installer.sh /tmp/qt/installer.run "$QT_PATH" \
    && find "$QT_PATH" -mindepth 1 -maxdepth 1 ! -name "${QT_VERSION}" -exec echo 'Cleaning Qt SDK: {}' \; -exec rm -r '{}' \; \
    && rm -rf /tmp/qt

# Reconfigure locale
RUN locale-gen en_US.UTF-8 && dpkg-reconfigure locales

# Add group & user + sudo
RUN groupadd -r ubuntu && useradd --create-home --gid ubuntu ubuntu && echo 'ubuntu ALL=NOPASSWD: ALL' > /etc/sudoers.d/ubuntu

# dirty hack. Probably it's better try to switch on rhel7 template which is officially supported by Qt 5.9.6:
# https://wiki.qt.io/Qt_5.9_Tools_and_Versions
# For QtDBus.so libdbus 1.3 is needed.
RUN echo "deb http://archive.ubuntu.com/ubuntu/ xenial main" > /etc/apt/sources.list.d/xenial.list && \
    apt-get update && \
    apt-get install -y libdbus-1-3


# Inside Docker fuse mount will not work, so extract AppImages:
# https://github.com/AppImage/AppImageKit/issues/405

# get linuxdeploy
RUN curl -4 -LO "https://github.com/linuxdeploy/linuxdeploy/releases/download/continuous/linuxdeploy-x86_64.AppImage" && \
    chmod +x linuxdeploy-x86_64.AppImage && \
    ./linuxdeploy-x86_64.AppImage --appimage-extract && \
    rm -f linuxdeploy-x86_64.AppImage && \
    mv squashfs-root linuxdeploy-x86_64.AppDir

# get linuxdeploy Qt plugin:
RUN curl -4 -LO "https://github.com/linuxdeploy/linuxdeploy-plugin-qt/releases/download/continuous/linuxdeploy-plugin-qt-x86_64.AppImage" && \
    chmod +x linuxdeploy-plugin-qt-x86_64.AppImage && \
    ./linuxdeploy-plugin-qt-x86_64.AppImage --appimage-extract && \
    rm -f linuxdeploy-plugin-qt-x86_64.AppImage && \
    mv squashfs-root linuxdeploy-plugin-qt-x86_64.AppDir

COPY scripts /scripts

# Clean up APT when done.
RUN apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Copy patched GNU TLS library (openssl compatibility wrapper) from 'gnutls' layer:
ENV GNUTLS_MULTI_CA_DIR  /opt/gnutls-multiroot-ca

RUN mkdir -p $GNUTLS_MULTI_CA_DIR/lib/

COPY --from=gnutls /usr/lib/x86_64-linux-gnu/libgnutls-openssl.so.27 $GNUTLS_MULTI_CA_DIR/lib/
COPY --from=gnutls /usr/lib/x86_64-linux-gnu/libgnutls.so.30         $GNUTLS_MULTI_CA_DIR/lib/
COPY --from=gnutls /usr/lib/x86_64-linux-gnu/libp11-kit.so.0         $GNUTLS_MULTI_CA_DIR/lib/
COPY --from=gnutls /usr/lib/x86_64-linux-gnu/libnettle.so.6          $GNUTLS_MULTI_CA_DIR/lib/
COPY --from=gnutls /usr/lib/x86_64-linux-gnu/libhogweed.so.4         $GNUTLS_MULTI_CA_DIR/lib/
COPY --from=gnutls /usr/lib/x86_64-linux-gnu/libgmp.so.10            $GNUTLS_MULTI_CA_DIR/lib/
COPY --from=gnutls /usr/lib/x86_64-linux-gnu/libffi.so.6             $GNUTLS_MULTI_CA_DIR/lib/
