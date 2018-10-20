FROM ubuntu:14.04

MAINTAINER Alexander Sashnov "sashnov@ngs.ru"

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
