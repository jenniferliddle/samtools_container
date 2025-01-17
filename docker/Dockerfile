FROM ubuntu:18.04 AS samtools_build

ARG HTSLIB_VERSION=1.19.1
ARG SAMTOOLS_VERSION=1.19.2
ARG BCFTOOLS_VERSION=1.19
ARG IRODS_VERSION="4.2.11"
ARG DEFLATE_VERSION=1.19

COPY irods_clients_dev /opt/docker/irods_clients_dev

RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get upgrade -y && \
    apt-get install -qq -y --no-install-recommends \
      git \
      ca-certificates \
      autotools-dev \
      build-essential \
      autoconf \
      automake \
      libbz2-dev \
      libcurl4-openssl-dev \
      libgd-dev \
      liblzma-dev \
      libssl-dev \
      libncurses5-dev \
      curl \
      dirmngr \
      gpg \
      gpg-agent \
      cmake \
      lsb-release \
      locales && \
      locale-gen en_GB en_GB.UTF-8 && \
      localedef -i en_GB -c -f UTF-8 -A /usr/share/locale/locale.alias en_GB.UTF-8

ENV LANG=en_GB.UTF-8 \
    LANGUAGE=en_GB \
    LC_ALL=en_GB.UTF-8


RUN curl -sSL https://packages.irods.org/irods-signing-key.asc | apt-key add - && \
    echo "deb [arch=amd64] https://packages.irods.org/apt/ $(lsb_release -sc) main" |\
    tee /etc/apt/sources.list.d/renci-irods.list && \
    apt-get update && \
    apt-get install -q -y --no-install-recommends \
    irods-dev="${IRODS_VERSION}-1~$(lsb_release -sc)" \
    irods-runtime="${IRODS_VERSION}-1~$(lsb_release -sc)"

RUN apt-key add - < /opt/docker/irods_clients_dev/keys/git-core-ppa.asc && \
    echo "deb https://ppa.launchpadcontent.net/git-core/ppa/ubuntu $(lsb_release -sc) main" |\
    tee /etc/apt/sources.list.d/git-core.list && \
    apt-get update && \
    apt-get install -q -y --no-install-recommends \
    git

RUN cd /tmp && \
    curl -sSL -O https://github.com/ebiggers/libdeflate/releases/download/v${DEFLATE_VERSION}/libdeflate-${DEFLATE_VERSION}.tar.gz && \
    tar xzf libdeflate-${DEFLATE_VERSION}.tar.gz && \
    cd libdeflate-${DEFLATE_VERSION} && \
    cmake . && \
    cmake --build .

RUN cd /tmp && \
    curl -sSL -O "https://github.com/samtools/htslib/releases/download/${HTSLIB_VERSION}/htslib-${HTSLIB_VERSION}.tar.bz2" && \
    tar xfj htslib-${HTSLIB_VERSION}.tar.bz2 && \
    cd htslib-${HTSLIB_VERSION} && \
    CPPFLAGS=-I../libdeflate-${DEFLATE_VERSION} LDFLAGS=-L../libdeflate-${DEFLATE_VERSION} ./configure --enable-plugins --with-libdeflate && \
    make && \
    make install && \
    ldconfig

RUN cd /tmp && \
    curl -sSL -O "https://github.com/samtools/htslib-plugins/archive/refs/tags/201712.tar.gz" && \
    tar xfz 201712.tar.gz && \
    cd htslib-plugins-201712 && \
    export IRODS_HOME=/usr/include/irods && \
    make && \
    make install

RUN cd /tmp && \
    curl -sSL -O "https://github.com/samtools/samtools/releases/download/${SAMTOOLS_VERSION}/samtools-${SAMTOOLS_VERSION}.tar.bz2" && \
    tar xfj samtools-${SAMTOOLS_VERSION}.tar.bz2 && \
    cd samtools-${SAMTOOLS_VERSION} && \
    ./configure --with-htslib=/tmp/htslib-${HTSLIB_VERSION} && \
    make install

RUN cd /tmp && \
    curl -sSL -O "https://github.com/samtools/bcftools/releases/download/${BCFTOOLS_VERSION}/bcftools-${BCFTOOLS_VERSION}.tar.bz2" && \
    tar xfj bcftools-${BCFTOOLS_VERSION}.tar.bz2 && \
    cd bcftools-${BCFTOOLS_VERSION} && \
    ./configure --with-htslib=/tmp/htslib-${HTSLIB_VERSION} && \
    make install

#
# Next stage: make everything smaller
#
FROM ubuntu:18.04

RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get upgrade -y && \
    apt-get install -qq -y --no-install-recommends \
      libssl-dev \
      gpg \
      gpg-agent \
      locales && \
      locale-gen en_GB en_GB.UTF-8 && \
      localedef -i en_GB -c -f UTF-8 -A /usr/share/locale/locale.alias en_GB.UTF-8


COPY --from=samtools_build /usr/local/bin/samtools /usr/local/bin/samtools
COPY --from=samtools_build /usr/local/bin/bcftools /usr/local/bin/bcftools
COPY --from=samtools_build /usr/local/bin/htsfile /usr/local/bin/htsfile
COPY --from=samtools_build /usr/local/bin/tabix /usr/local/bin/tabix
COPY --from=samtools_build /usr/local/lib/lib* /usr/local/lib/
COPY --from=samtools_build /usr/lib/lib* /usr/lib/
COPY --from=samtools_build /usr/local/libexec/htslib /usr/local/libexec/htslib/
COPY --from=samtools_build /opt/irods-externals /opt/irods-externals
COPY --from=samtools_build /tmp/libdeflate-1.19/libdeflate.so* /usr/local/lib/

# Copy the singularity-wrapper scripts
ARG DOCKER_IMAGE
ARG DOCKER_TAG
ENV DOCKER_IMAGE="$DOCKER_IMAGE"
ENV DOCKER_TAG="$DOCKER_TAG"

RUN ldconfig

COPY --from=singularity ./scripts/* /usr/local/bin/
COPY ./manifest.txt /opt/wtsi-npg/etc/manifest.txt

CMD ["/bin/bash"]

