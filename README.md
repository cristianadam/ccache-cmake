# ccache-cmake

This project includes a CMake build port for [ccache](https://github.com/ccache/ccache), which is 
is part of the project as a git subtree.

[gperf](https://www.gnu.org/software/gperf/) 3.1 is also included under the gperf folder.

# Building

Should be done as any other CMake project:

```
$ mkdir build
$ cd build 
$ cmake -GNinja -DCMAKE_BUILD_TYPE=Release ../ccache-cmake
```
