# Azahar RetroArch Core Builder (Android)

This repository provides instructions and a workflow for building the **Azahar** libretro core (`.so`) for Android using Docker. This allows you to run Azahar (a Citra fork) within RetroArch on Android devices.

The below is tested on this commit: [Azahar-c5516](https://github.com/azahar-emu/azahar/tree/c55165e19b8f015bf754bfd32b71b2794cfe9436). Made it possible after this merge: [libretro core - 1215](https://github.com/azahar-emu/azahar/pull/1215)

# TLDR: 
**This will build .so file to use for RetroArch on Android**
``` sh
git clone --recursive https://github.com/azahar-emu/azahar.git
#edit error.cpp as per this guide, otherwise you will get: /source/src/common/error.cpp:44:9: error: cannot initialize a variable of type 'int' with an rvalue of type 'char * _Nonnull'
docker run -it -v [path-to-azahar-source]:/source cimg/android:2026.02.1-ndk sleep infinity
```
``` sh
apt-get update
apt-get install -y    cmake    ninja-build    build-essential
export ANDROID_NDK_HOME=/home/circleci/android-sdk/ndk/29.0.14206865/
cmake .. -G Ninja     -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake     -DANDROID_ABI=arm64-v8a     -DANDROID_PLATFORM=android-24     -DENABLE_LIBRETRO=ON     -DENABLE_QT=OFF     -DENABLE_SDL2=OFF     -DENABLE_ROOM=OFF     -DENABLE_TESTS=OFF
ninja azahar_libretro
#The .so file with a name 'azahar_libretro_android.so' will appear in builddir/bin/Release folder, approximate size is 357MB
#Copy the file to Android and load the core from RetroArch menu
```

## Prerequisites

Before starting, ensure you have the following installed on your host machine:

1.  **Docker Desktop** (or Docker Engine) installed and running.
2.  **Git** (to clone the Azahar source code).
3.  **Azahar Source Code**: You must have the Azahar repository cloned locally.

## Build Instructions

### 1. Clone Azahar Source
If you haven't already, clone the Azahar repository to your local machine.

```bash
git clone --recursive https://github.com/Azahar-Emu/Azahar.git
```

Edit `src/common/error.cpp`:

```cpp
// Copyright Citra Emulator Project / Azahar Emulator Project
// Licensed under GPLv2 or any later version
// Refer to the license.txt file included.

// Copyright 2013 Dolphin Emulator Project
// Licensed under GPLv2 or any later version
// Refer to the license.txt file included.

#include <cstddef>
#ifdef _WIN32
#include <windows.h>
#else
#include <cerrno>
#include <cstring>
#endif

#include "common/error.h"

namespace Common {

std::string NativeErrorToString(int e) {
#ifdef _WIN32
    LPSTR err_str;

    DWORD res = FormatMessageA(FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_ALLOCATE_BUFFER |
                                   FORMAT_MESSAGE_IGNORE_INSERTS,
                               nullptr, e, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT),
                               reinterpret_cast<LPSTR>(&err_str), 1, nullptr);
    if (!res) {
        return "(FormatMessageA failed to format error)";
    }
    std::string ret(err_str);
    LocalFree(err_str);
    return ret;
#else
    char err_str[255];
    // Modern Android NDK and GLIBC often use the GNU-specific strerror_r
#if (defined(__GLIBC__) && (_GNU_SOURCE || (_POSIX_C_SOURCE < 200112L && _XOPEN_SOURCE < 600))) || \
    defined(__ANDROID__)
    // Thread safe (GNU-specific) - returns char*
    return std::string(strerror_r(e, err_str, sizeof(err_str)));
#else
    // Thread safe (XSI-compliant) - returns int
    if (strerror_r(e, err_str, sizeof(err_str)) != 0) {
        return "(strerror_r failed to format error)";
    }
    return std::string(err_str);
#endif // GNU vs XSI
#endif // _WIN32
}

std::string GetLastErrorMsg() {
#ifdef _WIN32
    return NativeErrorToString(GetLastError());
#else
    return NativeErrorToString(errno);
#endif
}

} // namespace Common
```

### 2. Start the Docker Container
Run the Docker container using the Android NDK image. You need to mount your local Azahar source directory into the container.

**⚠️ Important:** Replace `C:\Path\To\Your\azahar` with the **absolute path** to your local Azahar source folder.

**Windows (PowerShell/CMD):**
```powershell
docker run -it -v C:\Path\To\Your\azahar:/source cimg/android:2026.02.1-ndk sleep infinity
```

**Linux/macOS:**
```bash
docker run -it -v /path/to/your/azahar:/source cimg/android:2026.02.1-ndk sleep infinity
```

*Note: The container will start, enter to its shell session. Run the following steps **inside** the Docker container.*
```bash
docker ps
docker exec -it [container-name] sh
```

### 3. Install Build Dependencies
Inside the container, update packages and install necessary build tools:

```bash
apt-get update
apt-get install -y cmake ninja-build build-essential
```

### 4. Configure Environment
Set the Android NDK home environment variable. This path is specific to the `cimg/android:2026.02.1-ndk` image.

```bash
export ANDROID_NDK_HOME=/home/circleci/android-sdk/ndk/29.0.14206865/
```

### 5. Configure CMake
Create a build directory and configure the build for Android ARM64.

```bash
cd /source
mkdir builddir
cd builddir

cmake .. -G Ninja \
    -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake \
    -DANDROID_ABI=arm64-v8a \
    -DANDROID_PLATFORM=android-24 \
    -DENABLE_LIBRETRO=ON \
    -DENABLE_QT=OFF \
    -DENABLE_SDL2=OFF \
    -DENABLE_ROOM=OFF \
    -DENABLE_TESTS=OFF
```

### 6. Compile
Run Ninja to build the libretro core.

```bash
ninja azahar_libretro
```

### 7. Retrieve the Artifact
Once the build completes successfully, the shared object file will be located in the following directory inside the container:

*   **Path:** `/source/builddir/bin/Release/`
*   **Filename:** `azahar_libretro_android.so`
*   **Approximate Size:** 357MB

You can copy this file out of the container to your host machine using `docker cp` (run this from your **host** terminal, not inside the container):

```bash
# Get the Container ID first via 'docker ps'
docker cp <container_id>:/source/builddir/bin/Release/azahar_libretro_android.so ./azahar_libretro_android.so
```

## Usage

1.  Transfer `azahar_libretro_android.so` to your Android device.
2.  Place the file in your RetroArch `cores` directory (usually `/Android/data/com.retroarch/files/cores/`).
3.  Open RetroArch, go to **Load Core**, and select **Azahar**.

## Notes & Troubleshooting

*   **NDK Path:** If the Docker image version changes, the `ANDROID_NDK_HOME` path (`/home/circleci/android-sdk/ndk/...`) may change. Check the container's filesystem if the build fails to locate the toolchain.
*   **Architecture:** This build is configured for `arm64-v8a`. Ensure your Android device supports 64-bit architecture (most modern devices do).
*   **Build Time:** Compilation may take a significant amount of time depending on your host CPU resources allocated to Docker.
*   **Storage:** Ensure you have enough disk space; the build artifacts and intermediate files can be large.

## License

Please refer to the original [Azahar Repository](https://github.com/Azahar-Emu/Azahar) for licensing information regarding the emulator source code. Ensure you comply with all legal requirements regarding BIOS files and game ROMs.
