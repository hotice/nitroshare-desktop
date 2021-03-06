## NitroShare

[![Build Status](http://windows.quickmediasolutions.com/job/NitroShare/badge/icon)](http://windows.quickmediasolutions.com/job/NitroShare)
[![](http://img.shields.io/badge/license-MIT-blue.svg?style=flat)](http://opensource.org/licenses/MIT)
[![Join the chat at https://gitter.im/nitroshare/nitroshare-desktop](http://img.shields.io/badge/gitter-JOIN_CHAT-green.svg?style=flat)](https://gitter.im/nitroshare/nitroshare-desktop?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

A cross-platform network file transfer application designed to make transferring any file to any device as painless as possible.

### Features

* Runs on Windows / Mac OS X / Linux
* Automatic discovery of devices on the local network
* Simple and intuitive user interface
* Transfer entire directories
* Completely free and open-source

### Build Requirements

The requirements for building NitroShare are as follows:

* CMake 3.2.0+
* C++ compiler with support for C++11:
    * Microsoft Visual C++ 2013+
    * GCC 4.7+
    * Clang 3.1+
* Qt 5.1+ (Qt 5.3+ required for building installers)

On supported Linux platforms, NitroShare can be built with support for [application indicators](https://unity.ubuntu.com/projects/appindicators/) and Unity integration. These features are enabled when the following development packages are installed:

* GTK+ 2
* libappindicator
* libnotify
* libunity

### Building the Application

The instructions below describe the build process for each of the supported platforms.

#### Ubuntu 14.04, 14.10, 15.04, & 15.10

1. Versions of Ubuntu prior to 15.10 (Wily Werewolf) include an outdated version of CMake and therefore you will need to add the following PPA before installing CMake:

        sudo add-apt-repository ppa:george-edison55/cmake-3.x
        sudo apt-get update

2. Install GCC, CMake, and the Qt 5 development files:

        sudo apt-get install build-essential cmake qtbase5-dev

   To enable support for application indicators and Unity integration, you will need to install the following packages:

        sudo apt-get install libgtk2.0-dev libappindicator-dev libnotify-dev libunity-dev

2. Change to the root of the source directory and build the project:

        mkdir build && cd build
        cmake ..
        make

3. The NitroShare binary will be in the `out/` directory and can be run with:

        out/nitroshare

#### Windows 7, 8, 8.1, and 10

1. Download and install the following tools:

    - [Visual Studio Express 2013 for Windows Desktop](http://go.microsoft.com/?linkid=9832280&clcid=0x409) [requires sign-in]
    - [CMake](https://cmake.org/download/)
    - [Qt Online Installer for Windows](http://www.qt.io/download-open-source/)

2. Ensure that the `bin` directory for Qt has been added to the `PATH` environment variable.

3. Open the appropriate command prompt for Visual C++. In Visual C++ 2013, these shortcuts are labeled as follows:

    - VS2013 x86 Native Tools Command Prompt
    - VS2013 x64 Cross Tools Command Prompt

4. Change to the root of the source directory and build the project

        mkdir build && cd build
        cmake -G"NMake Makefiles" -DCMAKE_BUILD_TYPE=Release ..
        nmake

5. The NitroShare binary will be in the `out\` directory and can be run with:

        out\nitroshare.exe

> **Note:** if you receive an error similar to "The program can't start because [...].dll is missing from your computer.", then you will need to manually copy the Qt DLLs to the same directory as the executable. (This is done automatically for Qt 5.2+.)

#### OS X 10.7, 10.8, 10.9, 10.10, & 10.11

1. Download and install the following tools:

    - [Xcode](https://itunes.apple.com/ca/app/xcode/id497799835?mt=12)
    - [CMake](https://cmake.org/download/)
    - [Qt Online Installer for Mac](http://www.qt.io/download-open-source/)

2. Ensure that the `bin` directory for CMake and Qt have been added to `/etc/paths`.

3. Open the terminal app, change to the root source directory, and build the project:

        mkdir build && cd build
        cmake ..
        make

4. The NitroShare bundle will be in the `out/` directory and can be run with:

        open out/nitroshare.app

### Building the Installers and Packages

The instructions below describe the steps necessary to create installers or packages on each of the supported platforms.

#### Ubuntu

Debian packaging for NitroShare can be found here:
https://github.com/nitroshare/nitroshare-desktop-debian

#### Windows

Building the installers for 32 and 64-bit editions of Windows requires [Inno Setup](http://www.jrsoftware.org/isinfo.php).

Run the following command to build a Windows EXE installer:

    nmake exe

#### OS X

Run the following command to build a compressed disk image:

    make dmg

### Protocol Description

NitroShare uses a custom protocol running atop a TCP connection to transfer files. Communication consists of an exchange of "packets" which have the following format:

    PacketSize [32-bit little-endian signed integer]
    PacketType [8-bit signed integer]
    Data [n-byte stream where "n" is equal to PacketSize]

There are four types of packets:

* **Success (0)** - sent by the receiver to indicate successful transfer. `PacketSize` is always 0.
* **Error (1)** - sent by either peer to indicate an error has occurred. `Data` contains a UTF-8 encoded error message.
* **Json (2)** - sent by the sender. `Data` contains a JSON object.
* **Binary (3)** - sent by the sender. `Data` contains raw binary data.

A transfer consists of the following sequence of steps:

1. Sender opens a TCP connection to the receiver.
2. Sender begins by sending the transfer header - a JSON packet with the following information (note that all fields are strings):

        {
            "name": "",  // name of the sender
            "size": "",  // total size of all files being transferred
            "count": ""  // total number of files being transferred
        }

3. Sender begins sending the files and directories one-by-one. Each item begins with an item header - a JSON packet with the following information:

        {
            "name": "",           // filename (see below)
            "directory": false,   // whether the item is a directory
            "created": "",        // creation date of the item
            "last_modified": "",  // last modification date of the item
            "last_read": ""       // last access time of the item
        }

   The `name` field contains the relative path and filename of the item. The receiver determines the absolute path by combining this value with the directory the user has chosen for receiving files.

4. Once each item header is sent, the file's contents are sent in a sequence of binary packets. The default size for each packet is 64kb.

5. The receiver knows how many items to expect from the `count` field in the transfer header. Once all of the items have been received, the receiver sends a success packet to the sender which then closes the connection.
