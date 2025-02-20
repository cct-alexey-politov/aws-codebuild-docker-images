# Copyright 2020-2022 Amazon.com, Inc. or its affiliates. All Rights Reserved.
#
# Licensed under the Amazon Software License (the "License"). You may not use this file except in compliance with the License.
# A copy of the License is located at
#
#    http://aws.amazon.com/asl/
#
# or in the "license" file accompanying this file.
# This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied.
# See the License for the specific language governing permissions and limitations under the License.
FROM public.ecr.aws/ubuntu/ubuntu:20.04 AS core

ARG DEBIAN_FRONTEND="noninteractive"

# Install git, SSH, and other utilities
RUN set -ex \
    && echo 'Acquire::CompressionTypes::Order:: "gz";' > /etc/apt/apt.conf.d/99use-gzip-compression \
    && apt-get update \
    && apt install -y -qq apt-transport-https gnupg ca-certificates \
    && apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF \
    && apt-get install software-properties-common -y -qq --no-install-recommends \
    && apt-add-repository -y ppa:git-core/ppa \
    && apt-get update \
    && apt-get install git=1:2.* -y -qq --no-install-recommends \
    && git version \
    && apt-get install -y -qq --no-install-recommends openssh-client \
    && mkdir ~/.ssh \
    && mkdir -p /codebuild/image/config \
    && touch ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa -H github.com >> ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa -H bitbucket.org >> ~/.ssh/known_hosts \
    && chmod 600 ~/.ssh/known_hosts \
    && apt-get install -y -qq --no-install-recommends \
    apt-utils asciidoc autoconf automake build-essential bzip2 \
    bzr curl dirmngr docbook-xml docbook-xsl dpkg-dev \
    e2fsprogs expect fakeroot file g++ gcc gettext gettext-base \
    groff gzip iptables jq less libapr1 libaprutil1 \
    libargon2-0-dev libbz2-dev libc6-dev libcurl4-openssl-dev \
    libdb-dev libdbd-sqlite3-perl libdbi-perl libdpkg-perl \
    libedit-dev liberror-perl libevent-dev libffi-dev libgeoip-dev \
    libglib2.0-dev libhttp-date-perl libio-pty-perl libjpeg-dev \
    libkrb5-dev liblzma-dev libmagickcore-dev libmagickwand-dev \
    libmysqlclient-dev libncurses5-dev libncursesw5-dev libonig-dev \
    libpq-dev libreadline-dev libserf-1-1 libsodium-dev libsqlite3-dev libssl-dev \
    libsvn1 libsvn-perl libtcl8.6 libtidy-dev libtimedate-perl \
    libtool libwebp-dev libxml2-dev libxml2-utils libxslt1-dev \
    libyaml-dev libyaml-perl llvm locales make mlocate \
    netbase openssl patch pkg-config procps python3-configobj \
    python-openssl rsync sgml-base sgml-data \
    tar tcl tcl8.6 tk tk-dev unzip wget xfsprogs xml-core xmlto xsltproc \
    libzip5 libzip-dev vim xvfb xz-utils zip zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

ENV LC_CTYPE="C.UTF-8"

RUN useradd codebuild-user

#=======================End of layer: core  =================


FROM core AS tools

# Install Firefox
RUN set -ex \
    && apt-add-repository -y "deb http://archive.canonical.com/ubuntu $(lsb_release -sc) partner" \
    && apt-get install -y -qq firefox \
    && firefox --version

# Install GeckoDriver
RUN set -ex \
    && apt-get install -y -qq firefox-geckodriver \
    && geckodriver --version

# Install Chrome
RUN set -ex \
    && wget -q -O - https://dl.google.com/linux/linux_signing_key.pub | apt-key add - \
    && echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" | tee /etc/apt/sources.list.d/google.list \
    && apt-get update \
    && apt-get install -y -qq google-chrome-stable \
    && google-chrome --version

# Install ChromeDriver
RUN set -ex \
    && CHROME_VERSION=`google-chrome --version | awk -F '[ .]' '{print $3"."$4"."$5}'` \
    && CHROME_DRIVER_VERSION=`wget -qO- chromedriver.storage.googleapis.com/LATEST_RELEASE_$CHROME_VERSION` \
    && wget -qO /tmp/chromedriver_linux64.zip https://chromedriver.storage.googleapis.com/$CHROME_DRIVER_VERSION/chromedriver_linux64.zip \
    && unzip -q /tmp/chromedriver_linux64.zip -d /opt \
    && rm /tmp/chromedriver_linux64.zip \
    && mv /opt/chromedriver /opt/chromedriver-$CHROME_DRIVER_VERSION \
    && chmod 755 /opt/chromedriver-$CHROME_DRIVER_VERSION \
    && ln -s /opt/chromedriver-$CHROME_DRIVER_VERSION /usr/bin/chromedriver \
    && chromedriver --version

# Install stunnel
RUN set -ex \
    && STUNNEL_VERSION=5.64 \
    && STUNNEL_TAR=stunnel-$STUNNEL_VERSION.tar.gz \
    && STUNNEL_SHA256="eebe53ed116ba43b2e786762b0c2b91511e7b74857ad4765824e7199e6faf883" \
    && curl -o $STUNNEL_TAR https://www.usenix.org.uk/mirrors/stunnel/archive/5.x/$STUNNEL_TAR && echo "$STUNNEL_SHA256 $STUNNEL_TAR" | sha256sum --check && tar xfz $STUNNEL_TAR \
    && cd stunnel-$STUNNEL_VERSION \
    && ./configure \
    && make -j4 \
    && make install \
    && openssl genrsa -out key.pem 2048 \
    && openssl req -new -x509 -key key.pem -out cert.pem -days 1095 -subj "/C=US/ST=Washington/L=Seattle/O=Amazon/OU=Codebuild/CN=codebuild.amazon.com" \
    && cat key.pem cert.pem >> /usr/local/etc/stunnel/stunnel.pem \
    && cd .. && rm -rf stunnel-${STUNNEL_VERSION}*

# AWS Tools
# https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html
RUN curl -sS -o /usr/local/bin/aws-iam-authenticator https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/aws-iam-authenticator \
    && curl -sS -o /usr/local/bin/kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl \
    && curl -sS -o /usr/local/bin/ecs-cli https://s3.amazonaws.com/amazon-ecs-cli/ecs-cli-linux-amd64-latest \
    && curl -sS -L https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz | tar xz -C /usr/local/bin \
    && chmod +x /usr/local/bin/kubectl /usr/local/bin/aws-iam-authenticator /usr/local/bin/ecs-cli /usr/local/bin/eksctl

# Configure SSM
RUN set -ex \
    && mkdir /tmp/ssm \
    && cd /tmp/ssm \
    && wget -q https://s3.amazonaws.com/amazon-ssm-us-east-1/3.1.1374.0/debian_amd64/amazon-ssm-agent.deb \
    && dpkg -i amazon-ssm-agent.deb

# Install AWS CLI v2
# https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html
RUN curl https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o /tmp/awscliv2.zip \
    && unzip -q /tmp/awscliv2.zip -d /opt \
    && /opt/aws/install --update -i /usr/local/aws-cli -b /usr/local/bin \
    && rm /tmp/awscliv2.zip \
    && rm -rf /opt/aws \
    && aws --version

# Install env tools for runtimes

#nodejs
ARG SRC_DIR="/usr/src"
ARG N_SRC_DIR="$SRC_DIR/n"
RUN git clone https://github.com/tj/n $N_SRC_DIR \
    && cd $N_SRC_DIR && make install 

#=======================End of layer: tools  =================
FROM tools AS runtimes

#****************      NODEJS     ****************************************************

ENV NODE_16_VERSION="16.15.1"

RUN  n $NODE_16_VERSION && npm install --save-dev -g -f grunt && npm install --save-dev -g -f grunt-cli && npm install --save-dev -g -f webpack \
    && curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
    && echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
    && apt-get update && apt-get install -y -qq --no-install-recommends yarn \
    && yarn --version \
    && cd / && rm -rf $N_SRC_DIR && rm -rf /tmp/*

#****************      END NODEJS     ****************************************************

#=======================End of layer: runtimes  =================


FROM runtimes AS lastools

#****************        LASTools    *********************************************
WORKDIR /tmp

RUN apt update && \
    apt install -y \
    build-essential \
    pkg-config \
    cmake \
    gcc \
    gdb \
    wget \
    vim \
    git

#RUN mkdir lastools && \
#    cd lastools && \
#    wget https://downloads.rapidlasso.de/lastools.tar.gz && \
#    tar -xvf lastools.tar.gz

ENV TIFF_VERSION 4.0.9
RUN wget http://download.osgeo.org/libtiff/tiff-${TIFF_VERSION}.tar.gz && \
    tar -xvf tiff-${TIFF_VERSION}.tar.gz && \
    cd tiff-${TIFF_VERSION} && \
    cmake . && \
    make -j $(nproc) && \
    make install -j $(nproc)

RUN apt update && \
    apt install -y \
    zlib1g \
    zlib1g-dev

ENV LIBPNG_VERSION 1.4.22
RUN git clone --branch v${LIBPNG_VERSION} --single-branch https://github.com/glennrp/libpng && \
    cd libpng && \
    mkdir build && \
    cd build && \
    cmake .. && \
    make -j$(nproc) && \
    make install -j$(nproc)

RUN apt-get update && \
    apt-get install -y libjpeg62-dev \
    proj-bin \
    mlocate \
    autotools-dev \
    automake \
    libtool \
    zip \
    proj-bin \
    libproj-dev \
    ccache

ENV GEOTIFF_VERSION 1.4.3
RUN git clone --branch ${GEOTIFF_VERSION} --single-branch https://github.com/OSGeo/libgeotiff && \
    cd libgeotiff && \
    cd libgeotiff && \
    mkdir build && \
    cd build && \
    export CFLAGS="-I/usr/include -DACCEPT_USE_OF_DEPRECATED_PROJ_API_H=1" && \
    cmake -DBUILD_SHARED_LIBS=ON .. && \
    make -j$(nproc) && \
    make install -j$(nproc) && \
    cd .. && \    
    ./autogen.sh && \
    ./configure && \
    make dist && \
    tar xvzf libgeotiff-${GEOTIFF_VERSION}.tar.gz && \
    cd libgeotiff-${GEOTIFF_VERSION} && \
    mkdir build_autoconf && \
    cd build_autoconf && \
    CFLAGS="-Wall -Wextra -DACCEPT_USE_OF_DEPRECATED_PROJ_API_H=1 -D BUILD_SHARED_LIBS=ON"  ../configure && \
    make -j3 && \
    make check && \
    cd .. && \
    mkdir build_cmake && \
    cd build_cmake && \
    cmake -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_C_FLAGS="-Wall -Wextra -DACCEPT_USE_OF_DEPRECATED_PROJ_API_H=1" -DBUILD_SHARED_LIBS=ON .. && \
    make -j3 && \
    make install

RUN apt-get update && \
    apt-get install -y \
    gawk \
    bison

ENV LAS_VERSION 2.0.1
RUN git clone --branch v${LAS_VERSION} --single-branch https://github.com/LAStools/LAStools && \
    cd LAStools && \
    cd bin && \
    wget https://downloads.rapidlasso.de/lastools.tar.gz && \
    tar -xvf lastools.tar.gz && \
    cd .. && \
    cmake -D BUILD_SHARED_LIBS=ON . && \
    make -j $(nproc) && \
    make install -j $(nproc)

RUN wget https://sourceforge.net/projects/libjpeg/files/libjpeg/6b/jpegsrc.v6b.tar.gz && \
    tar -xvf jpegsrc.v6b.tar.gz

RUN apt update && \
    apt install -y \
    gnupg \
    software-properties-common

RUN mkdir -m755 -p /etc/apt/keyrings && \
    wget -O /etc/apt/keyrings/qgis-archive-keyring.gpg https://download.qgis.org/downloads/qgis-archive-keyring.gpg && \
    apt update && \
    apt install -y qgis \
    qgis-plugin-grass

RUN echo  "export PATH=/tmp/LAStools/bin:$PATH" >> ~/.bashrc
RUN echo "export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/tmp/LAStools/LASlib/lib:/usr/local/lib" >> ~/.bashrc
#*********************** END  LASTools  ****************************

#************************* libLAS **********************************
WORKDIR /tmp

RUN apt update && \
    apt install -y \
    libboost-program-options-dev \
    libboost-iostreams-dev \
    libboost-filesystem-dev \
    libboost-thread-dev

RUN wget http://download.osgeo.org/liblas/libLAS-1.8.1.tar.bz2 && \
    tar xvf libLAS-1.8.1.tar.bz2 && \
    cd libLAS-1.8.1 && \
    mkdir build && \
    cd build && \
    cmake -G "Unix Makefiles" ../ && \
    make && \
    make install

COPY PointCloudThinner /usr/local/bin    
RUN chmod +x /usr/local/bin/PointCloudThinner
#*********************** END  libLAS  ****************************

#=======================End of layer: lastools  =================

FROM lastools AS runtimes_n_corretto

#****************        DOCKER    *********************************************
ARG DOCKER_BUCKET="download.docker.com"
ARG DOCKER_CHANNEL="stable"
ARG DIND_COMMIT="3b5fac462d21ca164b3778647420016315289034"
ARG DOCKER_COMPOSE_VERSION="1.27.4"
ARG SRC_DIR="/usr/src"

ARG DOCKER_SHA256="9ccfc39305ae1d8882d18c9c431544fca82913d6df717409ac2244ac58c4f070"
ARG DOCKER_VERSION="20.10.15"

# Install Docker
RUN set -ex \
    && curl -fSL "https://${DOCKER_BUCKET}/linux/static/${DOCKER_CHANNEL}/x86_64/docker-${DOCKER_VERSION}.tgz" -o docker.tgz \
    && echo "${DOCKER_SHA256} *docker.tgz" | sha256sum -c - \
    && tar --extract --file docker.tgz --strip-components 1  --directory /usr/local/bin/ \
    && rm docker.tgz \
    && docker -v \
    # set up subuid/subgid so that "--userns-remap=default" works out-of-the-box
    && addgroup dockremap \
    && useradd -g dockremap dockremap \
    && echo 'dockremap:165536:65536' >> /etc/subuid \
    && echo 'dockremap:165536:65536' >> /etc/subgid \
    && wget -q "https://raw.githubusercontent.com/docker/docker/${DIND_COMMIT}/hack/dind" -O /usr/local/bin/dind \
    && curl -L https://github.com/docker/compose/releases/download/${DOCKER_COMPOSE_VERSION}/docker-compose-Linux-x86_64 > /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/dind /usr/local/bin/docker-compose \
    # Ensure docker-compose works
    && docker-compose version

VOLUME /var/lib/docker
#*********************** END  DOCKER  ****************************

#=======================End of layer: corretto  =================
FROM runtimes_n_corretto AS std_v5

# Activate runtime versions specific to image version.
RUN n $NODE_16_VERSION

# Configure SSH
COPY ssh_config /root/.ssh/config
COPY runtimes.yml /codebuild/image/config/runtimes.yml
COPY dockerd-entrypoint.sh /usr/local/bin/dockerd-entrypoint.sh
COPY legal/bill_of_material.txt     /usr/share/doc/bill_of_material.txt
COPY amazon-ssm-agent.json          /etc/amazon/ssm/amazon-ssm-agent.json

ENTRYPOINT ["/usr/local/bin/dockerd-entrypoint.sh"]

#=======================END of STD:5.0  =================
