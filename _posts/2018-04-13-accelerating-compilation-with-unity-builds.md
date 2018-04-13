---
title:  "Accelerating Compilation with Unity Builds"
layout: post
---

Unity builds, not to be confused with the Unity Engine, combine multiple C++ files into a single translation unit. This can significantly reduce compilation time and library size. Starting with version 4.7.0, You.i Engine provides a CMake module to implement unity builds.

# Advantages

- Significantly reduced compilation time
- Reduced library size
- Allows the compiler to make more optimizations (similar to link-time optimization)
- Slightly reduced linking time

# Disadvantages

- Symbols across separate C++ files must be uniquely named
- Possibly increased incremental compilation time
- In some cases, some symbols cannot be stripped out if they are unused in the final application

# Implementation

The unity build module in SDK works by taking as input a list of files. The module then generates a file from the files list which `#include`'s each of the input files. That generated file is then added to the output variable. Once all unity files have been generated, the user then adds the files from the output variable to the list of files to be compiled.

A CMake option controls the the unity module should do. By default, the `YI_UNITY_BUILD` variable is set to `OPTIMIZE_FOR_TIME`. If the variable is set to `OPTIMIZE_FOR_SIZE`, an optional second 'round' of unity build can be done to combine all unity builds with each other. This increases compilation time, but provides a minimally-sized library. If the variable is set to `DISABLED`, no unity file is generated and the list of input files is written out to the output variable.

Note that when the unity module generates an unity file, the list of input files is still written out to the output variable. The input files, however, are marked as 'headers' in CMake. This allows IDEs to locate the input files, even if they're being compiled through unity files.

# Results

When building SDK, unity builds have resulted in build times that are up to 7 times faster (e.g. from 6 minutes and 30 seconds down to 50 seconds), and library size that is up to 5 times smaller (e.g. from 500mb to 100mb for `libyouiengine.a`)

# Usage

To build a project in CMake, a list of C++ files is provided by the user (either by specifying each file, or by using globbing), and those files are added to an executable/library to be compiled. To (optionally) make use of the unity CMake module, an extra step is inserted between the 'generate the source files list' part and the 'add to executable/library' part.

Consider the following code from VideoPlayer's `CMakeList.txt`:

```cmake
include("${CMAKE_CURRENT_SOURCE_DIR}/SourceList.cmake")

[...]

if(NOT YI_SCRIPT_ONLY)
    set(YI_PROJECT_CODE_FILES
        ${YI_PROJECT_SOURCE}
        ${YI_PROJECT_HEADERS}
        ${YI_PROJECT_CODE_FILES}
    )
endif()
```

In this block, a sources list is provided through the `SourceList.cmake` file. That file defines a `YI_PROJECT_SOURCE` variable, which lists the project's .cpp files, and a `YI_PROJECT_HEADERS` variable, which lists the project's .h files. The content of those variables is then included in the `YI_PROJECT_CODE_FILES` variable, and the content of that variable is eventually added to the executable.

To make use of unity in this build, we only need to generate a new files list from `YI_APP_SOURCE_FILES`, and inserting that list into `YI_PROJECT_CODE_FILES`.

```cmake
include("${CMAKE_CURRENT_SOURCE_DIR}/SourceList.cmake")

[...]

include(Modules/YiConfigureUnityBuildFiles)
yi_configure_unity_build_files(OUTPUT YI_PROJECT_UNITY_SOURCE NAME "VideoPlayerUnityGenerated" FILES ${YI_PROJECT_SOURCE})

if(NOT YI_SCRIPT_ONLY)
    set(YI_PROJECT_CODE_FILES
        ${YI_PROJECT_UNITY_SOURCE} # <-- modified
        ${YI_PROJECT_HEADERS}
        ${YI_PROJECT_CODE_FILES}
    )
endif()
```

In the case of VideoPlayer sample, the SourceList.cmake file also has to be modified to make paths absolute. This is done by prefixing each source file with `${_SRC_DIR}/`

To support `OPTIMIZE_FOR_SIZE` unity builds, an extra section would be added like so:

```cmake
include("${CMAKE_CURRENT_SOURCE_DIR}/SourceList.cmake")

[...]

include(Modules/YiConfigureUnityBuildFiles)
yi_configure_unity_build_files(OUTPUT YI_PROJECT_UNITY_SOURCE NAME "VideoPlayerUnityGenerated" FILES ${YI_PROJECT_SOURCE})

if(YI_UNITY_BUILD STREQUAL OPTIMIZE_FOR_SIZE)
    yi_configure_unity_build_files(OUTPUT YI_PROJECT_UNITY_SOURCE NAME "YiUnityEverything" FILES
        ${CMAKE_CURRENT_BINARY_DIR}/generated/VideoPlayerUnityGenerated.cpp
        ${CMAKE_CURRENT_BINARY_DIR}/generated/OtherFilesUnityGenerated.cpp
        ${CMAKE_CURRENT_BINARY_DIR}/generated/DifferentThingsUnityGenerated.cpp
    )
endif()

if(NOT YI_SCRIPT_ONLY)
    set(YI_PROJECT_CODE_FILES
        ${YI_PROJECT_UNITY_SOURCE} # <-- modified
        ${YI_PROJECT_HEADERS}
        ${YI_PROJECT_CODE_FILES}
    )
endif()
```

The extra section generates an unity build file from other unity build files, further (and optionally) combining the `.cpp` files into a single translation unit.

After the CMake files have been modified, the `.cpp` files must then be modified to ensure that all symbols have unique names. In the VideoPlayer sample case, there are two `.cpp` files with the static function `LogMissingField`. Those functions need to be renamed or they need to be refactored into a single function.

Doing an unity build for the VideoPlayer sample wouldn't give much benefit since it has so few `.cpp` files (see the Considerations section for details).

# Considerations

## Use at least 10 unity build files
Computers generally have 4-8 cores, and the compiler cannot compile a single translation unit on multiple cores. As a result, it is recommended, when setting up a project as an unity build, that at least 10 unity build files be generated. Otherwise, compilation time may actually be reduced as other CPU cores would go unused.

## No more than 20 files per unity build file
If an unity file includes too many `.cpp` files (or if those the included `.cpp` files are very large), the speed of incremental builds can be noticeably affected when a single file is modified before recompiling. As a result, it is recommended that unity build files include no more than 20 files. The limit may be lower when the `.cpp` files are very large, and may be larger when the `.cpp` files are very small.

## Related file in the same unity build file
Unity files should contain 'related' files. This reduces the amount of unity files that may have to be recompiled when a header is modified. For example, one unity file could contain all screen files, while another could contain all view files. The folders used to separate header/source files are often a good indication of what should be included in the unity build files themselves.

## Unique symbols across all .cpp files
Symbols across `.cpp` files must be unique. This includes static function and variable names. Generally this only requires that those symbols be renamed to something file-specific. One common source of problems, however, is the `LOG_TAG` static variables. These variables are often defined as static const CYIString `LOG_TAG` in each file. To facilitate this, the unity build system (and the SDK logger) can use a #define `LOG_TAG` "MyClass" to define log tags. These defines are automatically undefined by the unity build module to avoid clashes. Alternatively, a unique `LOG_TAG` name can be used in each file, but this increases the amount of code that needs to be typed and increases the likelihood of errors.

## Build all unity build types
As the unity build is used, it becomes possible that a user would accidentally 'break' one of the other unity modes. For example, a user that always uses `OPTIMIZE_FOR_SIZE` may make use of a symbol from another `.cpp` file without realizing. If unity builds are disabled, the build would then fail. Similarly, non-unique symbol names across unity build files may not be caught if the user never builds with `OPTIMIZE_FOR_SIZE`. To reduce the likelihood of these problems occurring, the PRB can be setup to build each of the unity build types. For example, builds for Linux can be setup to build with unity builds DISABLED, and OSX can be setup to build with unity builds set to `OPTIMIZE_FOR_SIZE`.

When generating a project from the command line, the `--define YI_UNITY_BUILD=OPTIMIZE_FOR_SIZE` parameter can be used to switch to `OPTIMIZE_FOR_SIZE` mode. Similarly, the `--define YI_UNITY_BUILD=DISABLED` parameter can be used to disable unity builds.

## OPTIMIZE_FOR_SIZE for released libraries
The `OPTIMIZE_FOR_SIZE` unity build type is useful mostly for libraries, as the size of those libraries is significantly reduced. However, `OPTIMIZE_FOR_SIZE` increases the compilation time so it is recommended that it only be used when the libraries are being built for release/packaging/archiving. `OPTIMIZE_FOR_SIZE` can be used for executables too, but has little effect on the size of the generated executable. However, `OPTIMIZE_FOR_SIZE` allows for more compiler optimizations to be made. For example, a function defined in a different `.cpp` file can be inlined by the compiler when using `OPTIMIZE_FOR_SIZE`.

# Some files cannot be combined
Some files, for various reasons, cannot be combined with other files in an unity build. Those files must remain 'by themselves'. A few reasons why files would need to be excluded include: incompatible header includes, incompatible `#defines`, very large `.cpp` or `.c` files.


