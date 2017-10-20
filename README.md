# Tensorflow_iMX6Q
Tensorflow C++ source on Linux i.MX6 porting guide

## 1. Goals:
  * Cross compile tensorflow static lib, test `benchmark` on i.MX6Q
  * Compile an example(tensorflow/contribs/pi-example/label_image) using libtensorflow.a
## 2. Enviroment:
  * Host: Ubuntu 16.04
  * Target: Linux imx6qsabresd 3.14.52
  * Cross Compiler: gcc-linaro-arm-linux-gnueabihf-4.8-2014.04_linux
    * Do not use gcc-4.9, it'll cause compile error `__atomic_compare_exchange`; and I also try gcc-4.6.2, result in `--std=c++11 not found` error.  
## 3. Build `libtensorflow-core.a` and `benchmark`:
  ```
  cd tensorflow-master
  tensorflow/contrib/makefile/download_dependencies.sh
  ```
  The dependencies locate on `tensorflow/contrib/makefile/downloads/`:
    only `nsync` and `protoc` need to be cross-compiled, other dependencies only need to use the head files.
  * Compile `protoc`, reference to [this guide](https://github.com/eurotech/edc-examples/wiki/Cross-compiling-protobuf-for-ARM-architecture):
    * The first step is building a native (x86) version of the protobuf libraries and compiler (protoc) since the native compiler is necessary to build the test applications even if you're compiling for a different architecture:
      * Run the ./configure script without argument from tensorflow/contrib/makefile/downloads/protobuf/ than run make from tensorflow/contrib/makefile/downloads/protobuf/src. If everything works you should find the protoc executable in tensorflow/contrib/makefile/downloads/protobuf/src/.libs
      * Copy the protoc executable somewhere (for example in /usr/bin)
    * build the protobuf libraries for ARM:
      * `CC=arm-linux-gnueabihf-gcc CXX=arm-linux-gnueabihf-g++ ./configure --host=arm-linux --with-protoc=/usr/bin/protoc`
      * `rm -r src/.libs`
      * Run `make` in `src/`
    * Copy useful libs to tensorflow root directory:
      * `cd tensorflow-master/ && mkdir build`
      * `cp -r tensorflow/contrib/makefile/downloads/protobuf/src/.libs build/libs`
  * Build `nsync`:
    * Build a native(x86) version: 
      ```
      export HOST_NSYNC_LIB=`tensorflow/contrib/makefile/compile_nsync.sh`
      ```
      The object files locate on `tensorflow/contrib/makefile/downloads/nsync/builds/default.li
nux.c++11/`
    * Build a ARM version:
      ```
      export TARGET_NSYNC_LIB=`tensorflow/contrib/makefile/compile_nsync.sh -t linux -a armv7`
      ```
      this command will create a `armv7.linux.c++11/` directory in `tensorflow/contrib/makefile/downloads/nsync/builds/`,
      and generate native(x86) object files, we need to recompile an arm version:
      ```
      cd tensorflow/contrib/makefile/downloads/nsync/builds/armv7.linux.c++11/
      make clean
      vim Makefile
      modify first line to "CC=arm-linux-gnueabihf-g++"
      make
      ```
   * Build tensorflow:
      ```
      cd tensorflow-master/
      make -f tensorflow/contrib/makefile/Makefile HOST_OS=PI TARGET=PI OPTFLAGS="-Os" CXX="arm-linux-gnueabihf-g++ -march=armv7-a -mfloat-abi=hard -mfpu=neon -mtune=cortex-a9 --sysroot=/opt/fsl-imx-fb/3.14.52-1.1.0/sysroots/cortexa9hf-vfp-neon-poky-linux-gnueabi -L`pwd`/build/libs"
      ```
      Don't forget to add your --sysroot!
   * Test `benchmark`:
      download [data model](https://storage.googleapis.com/download.tensorflow.org/models/inception5h.zip)
      cp `tensorflow/contrib/makefile/gen/bin/benchmark` and build/libs/* to arm board
      ```
      benchmark \
      --graph=./tensorflow_inception_graph.pb \
      --input_layer="input:0" \
      --input_layer_shape="1,224,224,3" \
      --input_layer_type="float" \
      --output_layer="output:0"
      ```
## 4. Build `label_image`:
  * Replace `tensorflow/contrib/pi_examples/label_image/Makefile` with this repository.
  * `make -f tensorflow/contrib/pi_examples/label_image/Makefile  CXX="arm-linux-gnueabihf-g++ -march=armv7-a -mfloat-abi=hard -mfpu=neon -mtune=cortex-a9 --sysroot=/opt/fsl-imx-fb/3.14.52-1.1.0/sysroots/cortexa9hf-vfp-neon-poky-linux-gnueabi -L`pwd`/build/libs"`
  * Run `label_image` with [this guide](https://github.com/tensorflow/tensorflow/tree/r1.4/tensorflow/contrib/pi_examples)
  
## Reference:
> https://github.com/tensorflow/tensorflow/tree/r1.4/tensorflow/contrib/makefile  
> https://github.com/eurotech/edc-examples/wiki/Cross-compiling-protobuf-for-ARM-architecture  
> https://github.com/tensorflow/tensorflow/issues/9259  
> https://releases.linaro.org/archive/14.04/components/toolchain/binaries/  
> https://github.com/tensorflow/tensorflow/tree/r1.4/tensorflow/contrib/pi_examples
