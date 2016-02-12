---
layout: post
title:  "HowTo: Develop on the TI Tiva LaunchPad using Linux"
date:   2014-06-28 14:45:00 -0600
categories: ARM LaunchPad Linux programming tools
---

For years I've almost exclusively used Atmel's AVR series of 8-bit
microcontrollers in projects. AVRs get along quite nicely with Linux,
which I primarily development on. But I've been hearing great things
about TI's line of inexpensive development boards and their toolchain.
So I took the plunge and ordered a dev board.

Well as it turns out I ordered the wrong board. Woops! Instead of the
$10 [MSP-EXP430G2
Launchpad](http://www.ti.com/tool/MSP-EXP430G2) (featuring their MSP430
~~8-bit~~ 16-bit microcontroller) development board I accidentally
ordered the $13 [Tiva C series TM4C123GXL
Launchpad](http://www.ti.com/tool/ek-tm4c123gxl). Oh well, there are
worse things than mistakenly getting an ARM dev board for a few bucks.

So what's the toolchain even like on this thing? Will I need to download
some massive IDE? After some quick searching I was happy to find
that building, flashing, and debugging can all be done with command line
Linux tools.

The following setup instructions were taken from the [Tiva
Template](https://github.com/uctools/tiva-template) project and the
[Recursive Labs
Blog](http://recursive-labs.com/blog/2012/10/28/stellaris-launchpad-gnu-linux-getting-started),
along with a few of my own notes and helpful tidbits. I'll be working in
a folder named *Embedded* in my home directory, on a computer running
Ubuntu 13.10 Saucy Salamander.

Dependencies
------------

This section only needs to be performed once to get your development
computer setup.

1. Let's create a working directory to hold our projects and toolchain.

        mkdir ~/Embedded
        cd ~/Embedded

2. Install some dependencies using the commands below. This step should
    hopefully work for both 32- and 64-bit systems now (thanks
    Mayank!) And if you happen to be an Arch Linux user and are
    following along be sure to check out Vivek's advice in the comments
    for further 64-bit instructions.

        # For 64-bit systems:
        sudo dpkg –add-architecture i386
        # For everybody:
        sudo apt-get update
        sudo apt-get install flex bison libgmp3-dev libmpfr-dev \
          libncurses5-dev libmpc-dev autoconf texinfo build-essential \
          libftdi-dev python-yaml zlib1g-dev libtool
        # For 64-bit systems yet again:
        sudo apt-get install libc6:i386 libncurses5:i386 libstdc++6:i386
        # I needed these as well flashing over USB:
        sudo apt-get install libusb-1.0-0 libusb-1.0-0-dev

3. Go to the [GNU Tools for ARM Embedded
    Processors](https://launchpad.net/gcc-arm-embedded/+download)
    page and download the most recent tarball for Linux. Then extract it
    (including the top-level "*gcc-arm-none-eabi-...*" folder) into
    *\~/Embedded*.

        tar -xvf gcc-arm-none-eabi-4_8-2014q2-20140609-linux.tar.bz2 -C
        ~/Embedded

4. Add the extracted directory's bin folder to your user's path.

        export PATH=$PATH:$HOME/Embedded/gcc-arm-none-eabi-4_8-2014q1/bin

   This will only last for as long as you're logged in however, and
   will need to be rerun each time you want to do development. For a
   more permanent solution you can:

   * copy the contents of *gcc-arm-none-eabi-4\_8-2014q1/bin* to one
        of the directories in your path (not recommended)
   * create/modify your .bashrc file so it performs the above command
        at the beginning of every login. To do this open/create
        \~/.bashrc and add the above command (at the bottom or
        wherever appropriate).

5. Download the "TivaWare for Tiva C Series" package from TI's [Tiva C
    Series Software
    section](http://software-dl.ti.com/tiva-c/SW-TM4C/latest/index_FDS.html).
    You will be asked to create a login after clicking the download link
    in order to get the file. The current version was
    *SW-TM4C-2.1.0.12573.exe* when I downloaded it.
6. Extract the exe file to a new *tivaware* folder in *Embedded* using
    the commands below.

        cd ~/Embedded
        mkdir tivaware
        cd tivaware/
        # Go download exe from link above, and place it in the tivaware
        folder
        mv ~/Downloads/SW-TM4C-2.1.0.12573.exe . #Don't miss the dot
        unzip SW-TM4C-2.1.0.12573.exe

7. Compile with `make`

8.  The [Tiva Template](https://github.com/uctools/tiva-template)
    project has the stuff we need for compiling, as well as some
    example code. Go grab it off of GitHub with the commands below.

        cd ~/Embedded
        git clone git@github.com:uctools/tiva-template

9.  lm4flash is the utility we'll be using to flash our target board.
    Grab it from GitHub and compile the source.

        cd ~/Embedded
        git clone git://github.com/utzig/lm4tools.git
        cd lm4tools/lm4flash/
        make

10. lm4flash will work out of the box, but will require sudo to talk to
    the board through USB. You have the option of fixing this by setting
    up a udev rule for the device and adding your user to the *dialout*
    group with the following commands. **After that unplug the board,
    logout, login, and reconnect the board.**

        cd /etc/udev/rules.d
        echo "SUBSYSTEM==\"usb\", ATTRS{idVendor}==\"1cbe\",
        ATTRS{idProduct}==\"00fd\", MODE=\"0660\"" | sudo tee
        99-tiva-launchpad.rules
        # Remember to unplug and logout

11. We even have debugging capabilities. OpenOCD, the Open On-Chip
    Debugger, now supports this board. Grab and compile.

        cd ~/Embedded
        git clone http://openocd.zylin.com/openocd
        cd openocd
        git fetch http://openocd.zylin.com/openocd refs/changes/63/2063/1
        git checkout FETCH_HEAD
        git submodule init
        git submodule update
        ./bootstrap
        ./configure --enable-ti-icdi --prefix=`pwd`/..
        make -j3
        make install

Building Firmware
-----------------

We'll be building a simple blinking LED app using the example source
code and makefile included in the Tiva Template.

1.  Go look at *Embedded/tiva-template-master/src/main.c* to see the
    blinky goodness.
    
2.  Configure our project's Makefile in
    *Embedded/tiva-template-master* as follows
&nbsp;{% highlight bash %}
#######################################
# user configuration:
#######################################
# TARGET: name of the output file
TARGET = main
# MCU: part number to build for
MCU = TM4C123GH6PM
# SOURCES: list of input source sources
SOURCES = main.c startup\_gcc.c
# INCLUDES: list of includes, by default, use Includes directory
INCLUDES = -IInclude
# OUTDIR: directory to use for output
OUTDIR = build
# TIVAWARE\_PATH: path to tivaware folder
TIVAWARE\_PATH = $(HOME)/Embedded/tivaware
{% endhighlight %}
3.  Run *make* to build the app.

        cd ~/Embedded/tiva-template-master
        make

4.  This should hopefully build and place our output files in *build*.

Flashing
--------

1.  Run this to flash over USB. Remember to run with sudo if you did not
    follow the udev and group instructions in step
    10 under Dependencies.

        cd ~/Embedded/lm4tools/lm4flash/
        ./lm4flash ~/Embedded/tiva-template-master/build/main.bin

2.  Woohoo! Hopefully the RGB LED under the reset button is blinking
    red. Now go play with the delay time and colors in *main.c*.

References
----------

-   GitHub - Tiva Template: [A template for building firmware for Texas
    Stellaris ARM
    microcontrollers](https://github.com/uctools/tiva-template)
-   Recursive Labs Blog: [Programming the Stellaris Launchpad with
    GNU/Linux](http://recursive-labs.com/blog/2012/10/28/stellaris-launchpad-gnu-linux-getting-started/)
-   Stack Overflow: [Sample UDEV file for Stellaris
    Launchpad](http://stackoverflow.com/questions/14174601/sample-udev-file-for-stellaris-launchpad)
-   Reactivated: [Writing udev
    rules](http://www.reactivated.net/writing_udev_rules.html#ownership)
    