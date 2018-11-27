# -Miscellaneous

1,交叉编译libwebsockets库
2，NDK生成独立工具链（standalone toolchain)
3,NDK下编译libwebsockts库
4，编译protobuf/protobuf-c库
5,交叉编译flac库：
1) ./autogen.sh
2) ./configure --host=aarch64-poky-linux-gnu --prefix=<where you want to install the .so lib path> --enable-shared --disable-asm-optimizations
3)make
4)make install
