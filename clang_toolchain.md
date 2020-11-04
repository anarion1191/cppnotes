# Building a Clang Toolchain

## Compiling libcxx Without libcxxabi
```
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/home/kian/libc++ -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_CXX_FLAGS="-std=c++17" ../libcxx
```

## Compiling libcxxabi With libc++
```
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/home/kian/libc++abi -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_CXX_FLAGS="-std=c++17 -I/home/kian/libc++/include" -DLIBCXXABI_LIBCXX_INCLUDES=/home/kian/libc++/include/c++/v1 ../libcxxabi/
```

## Compiling libcxx With libcxxabi
```
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/home/kian/cpp-toolchain/libc++ -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_CXX_FLAGS="-std=c++17" -DLIBCXX_CXX_ABI=libcxxabi -DLIBCXX_CXX_ABI_LIBRARY_PATH=/home/kian/libc++abi/lib -DLIBCXX_CXX_ABI_INCLUDE_PATHS=../libcxxabi/include ../libcxx
```

## Compiling source code using the above toolchain
If both libc++ and libc++abi are installed with a prefix of /usr, then compiling code can be as simple as the following:

```
clang++ -o helloworld -std=c++17 -stdlib=libc++ -lc++ -lc++abi -lc -lgcc_s main.cpp
```

The following is a simple shell script that can be used to compile source code using the above toolchain, if it is not installed in a place that is already in the lookup path:

```
#!/bin/bash
LIBCPP_INCLUDE_DIR="/home/kian/cpp-toolchain/libc++/include/c++/v1"
LIBCPP_LIB_DIR="/home/kian/cpp-toolchain/libc++/lib"
LIBCPPABI_LIB_DIR="/home/kian/cpp-toolchain/libc++abi/lib"
CXX_FLAGS="-std=c++17 -stdlib=libc++ -lc++abi -lc -lgcc_s"
SOURCES="src/main.cpp"
clang++ -o ./build/a.out -I "${LIBCPP_INCLUDE_DIR}" -L "${LIBCPP_LIB_DIR}" -L "${LIBCPPABI_LIB_DIR}" ${CXX_FLAGS} ${SOURCES}
```

Running `ldd` on the produced executable will show the dynamic libraries it needs to run:
```
    linux-vdso.so.1 =>  (0x00007fffa4997000)
    libc++abi.so.1 => not found
    libc++.so.1 => not found
    libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f70061ea000)
    libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f7005fd4000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f7005c0a000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f70064f3000)
```

So, it turns out, the shared libraries for libc++ and libc++abi are not in a path that they should be, like the other shared libraries. I think the reason is due to the fact that I have installed the shared libraries in my home folder. Therefore, in order to get this to work properly, I have to set the environment variable `LD_LIBRARY_PATH` to the directory where these shared libraries are in.

Aside from using the `LD_LIBRARY_PATH` variable, during compilation, the flag `-Wl,-rpath,/abs/path/to/libdir` can be used to provide an absolute path to the location of the shared libraries that may be needed.As a slightly better alternative, the symbol `$ORIGIN` can be used to build the path relative to the executable's location. As an example, to inform `ld` to lookup the lib directory that is alongside the executable, the following flag can be used: `-Wl,-rpath,'$ORIGIN/lib'`.
It is important to use single quotes so that `$ORIGIN` does not get resolved when executing the command. The proper value for origin is resolved at runtime.

In order for the linker to be able to see where libraries are located, the configuration in `/etc/ld.so.conf` is used. In Debian-based distros, there is a `/etc/ld.so.conf.d/` directory that has a bunch of different configurations. Long story short, adding shared libraries to `/usr/local/lib` will make sure the linker sees these libraries because this directory is in the default configurations of /etc/ld.so.conf.d.