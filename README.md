GIMX
====

[![Build Status](https://travis-ci.com/matlo/GIMX.svg?branch=master)](https://travis-ci.com/matlo/GIMX)

GIMX is a free software that allows to use a computer as a hub for your gaming devices. It works on Windows® and GNU/Linux platforms. It is compatible with Playstation® and Xbox® gaming consoles. The connection between the computer and the gaming console is performed using a USB adapter – [get one on the GIMX shop!](https://blog.gimx.fr/product/gimx-adapter/) – or a Bluetooth® dongle (PS3/PS4 only). The capabilities depend on the platform, the connection method, and the gaming platform.

Links:
* [Documentation](https://wiki.gimx.fr)  
* [Source code](https://gimx.fr/source)  
* [Issue tracker](https://gimx.fr/buglist)  
* Licence: [GPLv3](https://www.gnu.org/copyleft/gpl.html)  
* [Donations](https://blog.gimx.fr/give/gimx-donations-current/)

Building with CMake
-------------------

The CMake build requires CMake 3.20 or newer, a C and C++ compiler, and the
development packages for wxWidgets, libxml2, libcurl, libusb, and SDL2 on
Windows. Linux builds also require pkg-config, ncursesw, X11/Xi, BlueZ, and
mhash. `msgfmt` is optional and is used to compile translations.

Configure and build from the repository root:

```sh
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --parallel
```

Install using the selected prefix:

```sh
cmake --install build --prefix /usr/local
```

To omit the command-line utility programs, configure with
`-DGIMX_BUILD_UTILS=OFF`. The existing Makefiles remain available.
