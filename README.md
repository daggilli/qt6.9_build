# Building Qt 6.9 under Ubuntu 22.04

My system is currently running Ubuntu 22.04 and the apt version of Qt is outdated (6.2.4). The Qt online installer is not fit for purpose. I don't want to sign up to an account, give all my details, and get spammed until the end of time just to use Qt for my personal projects. So I decided to install it directly from source, the latest version being 6.9.1 as of time of writing. This turned out to be easier said than done. In the hope that this might save someone hours of frustration, I am going to share how I did it.

**I make mo representations as to the correctness or completeness of these instructions. I have tried to be thorough, but your system's configuration might vary in ways that I cannot anticipate.**

## First steps

### Get the source

```
mkdir -p qt6.9.1/build
cd qt6.9.1
wget https://download.qt.io/official_releases/qt/6.9/6.9.1/single/qt-everywhere-src-6.9.1.tar.xz
tar xf qt-everywhere-src-6.9.1.tar.xz
```

### Install the necessary packages

This is a ridiculously long list and some of these packages might not be necessary. But most of them are, and it is very tedious to start a build, find out twenty minutes later it has failed, and then go hunting fot the new dependency. I am using gcc-15 and cmake 4, but I think gcc-13 and cmake 3.28 _should_ be sufficient. I assume you have a reasonably well-configured generic build environment already with tools like autoconf and pkg-config. I am not going to discuss how to instal GCC or Cmake and so on.

```
sudo apt update
sudo apt install ninja-build perl bison flex gperf libglib2.0-dev \
  libfontconfig1-dev libfreetype6-dev libx11-dev libxext-dev libxfixes-dev \
  libxi-dev libxrender-dev libxcb1-dev libxcb-util-dev libxcb-icccm4-dev \
  libxcb-image0-dev libxcb-keysyms1-dev libxcb-randr0-dev \
  libxcb-render-util0-dev libxcb-render0-dev libxcb-shape0-dev libxcb-shm0-dev \
  libxcb-sync-dev libxcb-xfixes0-dev libxcb-xinerama0-dev libxcb-xkb-dev \
  libxkbcommon-dev libxkbcommon-x11-dev libxcomposite-dev libxrandr-dev \
  libxcursor-dev libxdamage-dev libxinerama-dev libdrm-dev libegl1-mesa-dev \
  libgl1-mesa-dev libglx-dev libxcb-glx0-dev libdbus-1-dev libudev-dev

```

### Install xcb-cursor

Ubuntu 22.04 does not split xcb-cursor out to a separate package, but Qt6 needs it. Therefore so it can find the necessary symbols, it must be installed by hand

```
git clone https://gitlab.freedesktop.org/xorg/lib/libxcb-cursor.git
cd libxcb-cursor
./autogen.sh --prefix=/usr/local
make -j$(nproc)
sudo make install
cd ../build
```

You must also ensure that pkg-config can find xcb-cursor by modifying your system's search path:

`set PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH`

## Configure

```
 cmake "$(realpath ../qt-everywhere-src-6.9.1)" -DCMAKE_INSTALL_PREFIX=/usr/local/qt6 \
  -DCMAKE_BUILD_TYPE=Release -GNinja -DQT_FEATURE_ssl=ON \
  -DQT_FEATURE_system_zlib=ON -DQT_FEATURE_system_png=ON -DQT_FEATURE_system_jpeg=ON \
  -DBUILD_qtbase=ON -DBUILD_qttools=ON -DBUILD_qtimageformats=ON \
  -DBUILD_qtquick3d=ON -DQT_FEATURE_xcb=ON \
  -DQT_QPA_DEFAULT_PLATFORM=xcb
```

You may want to skip building the tests with `-DQT_BUILD_TESTS=OFF`.

## Build and install

Allowing cmake and gcc to parallelise without any explicit bounds can cause hundreds of gcc processes to be spawned, potentially up to the number of OS threads available. This slows the system to a crawl and eats all avaialble RAM. Eventually the OOM killer will start deleting processes, potentially even the shell in which you are building Qt. Setting a limit to one less than the number of processors on the machine is recommended.

```
cmake --build . --parallel $(($(nproc) - 1))
sudo cmake --install .
```

The build process is not fast: at the very least tens of minutes up to several hours depending on your machine.

## Install Qt Creator (optional)

The Qt Creator modelling tools depend on an obsolete module (qmt) that is no longer in the source tree, but is enabled by default. Unless it is disabled the build process will terminate with a linker error.

```
cd ..
git clone https://code.qt.io/qt-creator/qt-creator.git
cd qt-creator
mkdir build && cd build
cmake .. -DCMAKE_PREFIX_PATH=/usr/local/qt6 -DCMAKE_INSTALL_PREFIX=/usr/local/qtcreator \
 -DBUILD_MODELING_TOOLS=OFF -GNinja
cmake --build . --parallel $(($(nproc) - 1))
sudo cmake --install .
```

## System configuration

Your system needs to know where to find the new software:

```
export PATH=/usr/local/qt6/bin:/usr/local/qtcreator/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/qt6/lib:$LD_LIBRARY_PATH
export PKG_CONFIG_PATH=/usr/local/qt6/lib/pkgconfig:/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH
```

## Minimal test program

This is about the smallest test program for Qt that will still show any output:

`qtminimal.cpp`

```cpp
#include <QApplication>
#include <QWindow>

int main(int argc, char *argv[]) {
  QApplication app(argc, argv);
  QWindow window;
  window.setTitle("Qt Minimal");
  window.resize(250, 250);
  window.setVisible(true);
  return app.exec();
}
```

A bare-bones CmakeLists.txt is as follows:

`CmakeLists.txt`

```cmake
cmake_minimum_required(VERSION 3.29)
project(QtMinimalApp LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 26)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_compile_options(-Wall -Wunused)

# Suppress warnings about deprecated is_trivial_v in C++26
if(CMAKE_CXX_STANDARD GREATER_EQUAL 26)
    add_compile_options(-Wno-deprecated)
endif()

set(CMAKE_PREFIX_PATH "/usr/local/qt6")
find_package(Qt6 REQUIRED COMPONENTS Core Gui Widgets)

add_executable(qtminimal qtminimal.cpp)
target_link_libraries(qtminimal PRIVATE Qt6::Core Qt6::Gui Qt6::Widgets)
```

Alternatively, it can be compiled directly with

```bash
g++ -std=c++26 -Wall -Wunused -Wno-deprecated qtminimal.cpp -o qtminimal \
 -I/usr/local/qt6/include -I/usr/local/qt6/include -I/usr/local/qt6/include/QtGui \
 -I/usr/local/qt6/include/QtCore -I/usr/local/qt6/include/QtWidgets -fPIC \
 -L/usr/local/qt6/lib -lQt6Gui -lQt6Core -lQt6Widgets -lGL -lX11 -ldl -lpthread
```
