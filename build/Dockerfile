FROM debian:stretch

ARG user=worker
ARG debian_mirror=deb.debian.org

### openmvg
# setup system and get build dependencies
RUN useradd --create-home --shell /bin/bash $user \
 && sed -r -i -e s/deb.debian.org/$debian_mirror/ /etc/apt/sources.list \
 && apt-get update && apt-get install -y \
      build-essential \
      cmake \
      graphviz \
      git \
      libpng-dev \
      libjpeg-dev \
      libtiff-dev \
      libxxf86vm1 \
      libxxf86vm-dev \
      libxi-dev \
      libxrandr-dev \
      python-dev \  
      python-pip \
      wget

USER $user
ENV PATH $PATH:/home/$user/bin/

# openMVG build options
ARG branch=develop
ARG BUILD_TESTS=OFF
ARG BUILD_EXAMPLES=ON

# clone build and install
WORKDIR /home/$user
RUN mkdir src \
 && git clone --single-branch -b $branch --recursive https://github.com/openmvg/openmvg src/openMVG \
 && cd src/openMVG \
 && git submodule update --init --recursive \
 && wget https://github.com/openMVG/openMVG/commit/b51afc89b391b255fdda02bff67ff62c68212a09.patch -O /tmp/b51afc89b391b255fdda02bff67ff62c68212a09.patch \
 && ( patch -p1 -f < /tmp/b51afc89b391b255fdda02bff67ff62c68212a09.patch || true ) \
 && mkdir build \
 && cd build \
 && cmake \
      -DCMAKE_BUILD_TYPE=RELEASE \
      -DCMAKE_INSTALL_PREFIX="/home/$user" \
      -DOpenMVG_BUILD_TESTS=$BUILD_TESTS \
      -DOpenMVG_BUILD_EXAMPLES=$BUILD_EXAMPLES \
      . \
      ../src/ \
 && make -j $(nproc) \
 && if [ $BUILD_TESTS = ON ] ; then  make test ; fi  \
 && make install

### cmvs-pmvs
# clone build and install
RUN git clone \
      --single-branch \
      -b master \
      --recursive \
       https://github.com/pmoulon/cmvs-pmvs \
       src/cmvs-pmvs \
 && cd src/cmvs-pmvs \
 && mkdir build \
 && cd build \
 && cmake \
      -DCMAKE_BUILD_TYPE=RELEASE \
      -DCMAKE_INSTALL_PREFIX="/home/$user" \
      ../program/ \
 && make -j $(nproc) \
 && make install

### MVE & co
USER root
RUN apt-get install -y \
      build-essential \
      cmake \
      git \
      libpng-dev \
      libtiff5-dev \
      libjpeg62-turbo-dev \
      mesa-common-dev \
      libtbb-dev 

# clone build and install
USER $user
WORKDIR /home/$user
RUN git clone \
      --single-branch \
      -b master \
      --recursive \
      https://github.com/simonfuhrmann/mve \
      src/mve \
 && git clone \
      --single-branch \
      -b master \
      --recursive \
      https://github.com/flanggut/smvs \
      src/smvs \
 && git clone \
      --single-branch \
      -b master \
      --recursive \
      https://github.com/mkazhdan/PoissonRecon \
      src/PoissonRecon \
 && git clone \
      --single-branch \
      -b master \
      --recursive \
      https://github.com/nmoehrle/mvs-texturing \
      src/mvs-texturing

RUN make -j $(nproc) -C src/mve
RUN make -j $(nproc) -C src/smvs
RUN make -j $(nproc) -C src/PoissonRecon
RUN cd src/mvs-texturing \
 && mkdir build \
 && cd build \
 && cmake -DCMAKE_BUILD_TYPE=Release .. \
 && make -j $(nproc)  \
 && cd \
 && find \
      src/mve/apps \
      src/smvs/app \
      src/PoissonRecon/Bin \
      src/mvs-texturing/build/apps \
      -type f -executable -exec mv {} bin \;

### potreeconverter
USER root
RUN apt-get install libboost-all-dev -y
USER $user
# clone build and install
WORKDIR /home/$user/src
RUN git clone \
   --single-branch \
   -b master \
   https://github.com/m-schuetz/LAStools.git \
 && cd LAStools \
 && git checkout 8065ce39d50d09907691b5feda0267279428e586 \
 && cd LASzip \
 && mkdir build \
 && cd build \
 && cmake -DCMAKE_BUILD_TYPE=Release .. \
 && make -j $(nproc) \
 && cd /home/$user/src \
 && git clone \
   --single-branch \
   -b develop \
   --recursive \
   https://github.com/potree/PotreeConverter.git \
 && cd PotreeConverter \
 && git checkout b8c8df7914aa082ebd399798b8171ba96014bd17 \
 && mkdir build \
 && cd build \
 && cmake \
   -DCMAKE_BUILD_TYPE=Release \
   -DLASZIP_INCLUDE_DIRS=/home/$user/src/LAStools/LASzip/dll \
   -DLASZIP_LIBRARY=/home/$user/src/LAStools/LASzip/build/src/liblaszip.so \
   .. \
 && make -j $(nproc)

# copy our page template
COPY resources /home/$user/src/PotreeConverter/build/PotreeConverter/resources

COPY bin /home/$user/bin

# install nodejs

WORKDIR /home/$user
ARG nvm_version=v0.35.0
ARG node_version=v8.16.2
RUN wget -q -O /tmp/install.sh https://raw.githubusercontent.com/creationix/nvm/$nvm_version/install.sh \
 &&  . /tmp/install.sh \
 && export NVM_DIR="/home/$user/.nvm" \
 && [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh" \
 && nvm install $node_version \
 && echo progress=false >> .npmrc

USER root
RUN apt-get install -y \
      jq \
      parallel \
      imagemagick \
      exiftool \
      moreutils \
      libssl-dev \
      bc \
      vim

USER $user
RUN git clone https://github.com/doxel/jpeg_metadata_size src/jpeg_metadata_size \
 && cd src/jpeg_metadata_size \
 && make \
 && PREFIX=/home/$user make install


CMD ["/bin/bash", "-l", "-c", "export DEBUG=1 ; while true ; do doxel-workerflow 2>&1 ; sleep 10 ;  done"]
