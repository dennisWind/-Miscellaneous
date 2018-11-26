Complex Cross compiling example

Here are step by step instructions for cross-building the external projects needed for lws with lwsws + mbedtls as an example.

In the example, my toolchain lives in <tool_chain_file>/compile_tools/toolchain-aarch64_armv8-a_gcc-4.8-linaro_glibc-2.19 and is named arm-openwrt-linux-muslgnueabi. So you will need to adapt those to where your toolchain lives and its name where you see them here.

Likewise I do all this in /tmp but it has no special meaning, you can adapt that to somewhere else.

All "foreign" cross-built binaries are sent into /tmp/cross so they cannot be confused for 'native' x86_64 stuff on your host machine in /usr/[local/]....

Prepare the cmake toolchain file

1) cd /tmp

2) wget -O mytoolchainfile https://raw.githubusercontent.com/warmcat/libwebsockets/master/contrib/cross-arm-linux-gnueabihf.cmake

3) Edit /tmp/mytoolchainfile adapting CROSS_PATH, CMAKE_C_COMPILER and CMAKE_CXX_COMPILER to reflect your toolchain install dir and path to your toolchain C and C++ compilers respectively. For my case:

set(CROSS_PATH <tool_chain_file>/compile_tools/toolchain-aarch64_armv8-a_gcc-4.8-linaro_glibc-2.19/)

set(CMAKE_C_COMPILER "${CROSS_PATH}/bin/arm-openwrt-linux-muslgnueabi-gcc")

set(CMAKE_CXX_COMPILER "${CROSS_PATH}/bin/arm-openwrt-linux-muslgnueabi-g++")

libsockets:::

cmake .. -DCMAKE_INSTALL_PREFIX:PATH=<tool_chain_file>/compile_tools/toolchain-aarch64_armv8-a_gcc-4.8-linaro_glibc-2.19 \

    -DCMAKE_TOOLCHAIN_FILE=../cross-arm-linux-gnueabihf.cmake \

    -DLWS_WITHOUT_EXTENSIONS=1 -DLWS_WITH_SSL=0 \

    -DLWS_WITH_ZIP_FOPS=0 -DLWS_WITH_ZLIB=0

lib/libmbedtls:::

cmake .. -DCMAKE_TOOLCHAIN_FILE=../cross-arm-linux-gnueabihf.cmake -DCMAKE_INSTALL_PREFIX:PATH=<tool_chain_file>/compile_tools/toolchain-aarch64_armv8-a_gcc-4.8-linaro_glibc-2.19 -DCMAKE_BUILD_TYPE=RELEASE -DUSE_SHARED_MBEDTLS_LIBRARY=1

libsockets:::

cmake .. -DCMAKE_TOOLCHAIN_FILE=<tool_chain_file>/compile_tools/toolchain-aarch64_armv8-a_gcc-4.8-linaro_glibc-2.19 \

-DCMAKE_INSTALL_PREFIX:PATH=<tool_chain_file>/compile_tools/toolchain-aarch64_armv8-a_gcc-4.8-linaro_glibc-2.19 \

-DLWS_WITH_LWSWS=1 \

-DLWS_WITH_MBEDTLS=1 \

-DLWS_MBEDTLS_LIBRARIES="../../mbedtls-2.11.0/build/library/libmbedtls.so" \

-DLWS_MBEDTLS_INCLUDE_DIRS=../../mbedtls-2.11.0/include/mbedtls \

1/4: Building libuv cross:

1) export PATH=<tool_chain_file>/compile_tools/toolchain-aarch64_armv8-a_gcc-4.8-linaro_glibc-2.19/bin:$PATH

2) cd /tmp 

###mkdir cross we will put the cross-built libs in /tmp/cross

3) git clone https://github.com/libuv/libuv.git ##get libuv

4) cd libuv

5) ./autogen.sh

6) ./configure --host=aarch64-openwrt-linux-gnu --prefix=/tmp/cross

7) make && make install this will install to /tmp/cross/...

8) file /tmp/cross/lib/libuv.so.1.0.0 Check it's really built for ARM

/tmp/cross/lib/libuv.so.1.0.0: ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV), dynamically linked, not stripped

2/4: Building zlib cross

1) cd /tmp

2) git clone https://github.com/madler/zlib.git

3) cd /tmp/zlib

4) CC=aarch64-openwrt-linux-gnu-gcc ./configure --prefix=/tmp/cross

5) make && make install

6) file /tmp/cross/lib/libz.so.1.2.11 This is just to confirm we built an ARM lib as expected

/tmp/cross/lib/libz.so.1.2.11: ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV), dynamically linked, not stripped

3/4: Building mbedtls cross

1) cd /tmp

2) git clone https://github.com/ARMmbed/mbedtls.git

3) cd mbedtls ; mkdir build ; cd build

3) cmake .. -DCMAKE_TOOLCHAIN_FILE=../cross-arm-linux-gnueabihf.cmake -DCMAKE_INSTALL_PREFIX:PATH=/tmp/cross -DCMAKE_BUILD_TYPE=RELEASE -DUSE_SHARED_MBEDTLS_LIBRARY=1

## mbedtls also uses cmake, so you can simply reuse the toolchain file you used for libwebsockets. That is why you shouldn't put project-specific options in the toolchain file, it should just describe the toolchain.

4) make && make install

5) file /tmp/cross/lib/libmbedcrypto.so.2.6.0

/tmp/cross/lib/libmbedcrypto.so.2.13.1: ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV), dynamically linked, not stripped

4/4: Building libwebsockets with everything

1) cd /tmp

2) git clone ssh://git@github.com/warmcat/libwebsockets

3) cd libwebsockets ; mkdir build ; cd build

4) (this is all one line on the commandline)

cmake .. -DCMAKE_TOOLCHAIN_FILE=../cross-arm-linux-gnueabihf.cmake \

-DCMAKE_INSTALL_PREFIX:PATH=/tmp/cross \

-DLWS_WITH_LWSWS=1 \

-DLWS_WITH_MBEDTLS=1 \

-DLWS_MBEDTLS_LIBRARIES="/tmp/cross/lib/libmbedcrypto.so;/tmp/cross/lib/libmbedtls.so;/tmp/cross/lib/libmbedx509.so" \

-DLWS_MBEDTLS_INCLUDE_DIRS=/tmp/cross/include \

-DLWS_LIBUV_LIBRARIES=/tmp/cross/lib/libuv.so \

-DLWS_LIBUV_INCLUDE_DIRS=/tmp/cross/include \

-DLWS_ZLIB_LIBRARIES=/tmp/cross/lib/libz.so \

-DLWS_ZLIB_INCLUDE_DIRS=/tmp/cross/include

3) make && make install

4) file /tmp/cross/lib/libwebsockets.so

/tmp/cross/lib/libwebsockets.so: ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV), dynamically linked, not stripped

5) aarch64-openwrt-linux-objdump -p /tmp/cross/lib/libwebsockets.so | grep NEEDED

## Confirm that the lws library was linked against everything we expect (libm / libc are provided by your toolchain)

  NEEDED              libmbedtls.so

  NEEDED              libuv.so.1

  NEEDED              libm.so.6

  NEEDED              libgcc_s.so.1

  NEEDED              libc.so.6

You will also find the lws test apps in /tmp/cross/bin... to run lws on the target you will need to copy the related things from /tmp/cross... all the .so from /tmp/cross/lib and anything from /tmp/cross/bin you want.