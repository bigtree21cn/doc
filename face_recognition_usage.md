This doc is about how to install and test the face_recogination code source.
This doc is refer to [doc](https://github.com/ageitgey/face_recognition/blob/master/README_Simplified_Chinese.md)

# Install
## Install on baremental
- **install python3**
- **install dependency**

    ```bash
    apt-get install -y --fix-missing \
        build-essential \
        cmake \
        gfortran \
        git \
        wget \
        curl \
        graphicsmagick \
        libgraphicsmagick1-dev \
        libatlas-dev \
        libavcodec-dev \
        libavformat-dev \
        libgtk2.0-dev \
        libjpeg-dev \
        liblapack-dev \
        libswscale-dev \
        pkg-config \
        python3-dev \'
        python3-numpy \
        software-properties-common \
        zip \
        && apt-get clean && rm -rf /tmp/* /var/tmp/*
    ```
- **build dlib**  (optional in case you only use python)
    ```bash
    git clone https://github.com/davisking/dlib.git dlib
    cd dlib
    mkdir build; cd build; cmake ..; cmake --build .
    ```
- **install python dlib and extension**
    ```bash
    python3 setup.py install
    ```
- **install face_recongnition module**
    ```bash
    pip3 install face_recognition
    ```
## Install with docker
- Refer to [dockerfile](https://github.com/ageitgey/face_recognition/blob/master/Dockerfile)
  
# Testing
