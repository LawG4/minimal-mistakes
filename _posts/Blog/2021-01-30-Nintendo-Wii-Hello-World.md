---
title: "Nintendo Wii: Hello world!"
date: 2021-01-29T15:34:30-04:00

header:
  teaser: /assets/Blog/HelloWorldWii/Teaser.png
  og_image: /assets/Blog/HelloWorldWii/Thumb.png
  image: /assets/Blog/HelloWorldWii/Thumb.png

categories:
  - Blog
tags:
  - Alternative Hardware
  - C
  - Homebrew
  - Devkit
  - Nintendo Wii
---
 The Nintendo Wii has always been a favourite console of mine, and it has supplied me with countless hours of joy. Because of this, I want to join my passion for programming with this fantastic console. Thankfully there exists a toolchain called devkitPPC for programming this particular device. Today I would like to introduce you to those tools. This blog entry will be a traditional “Hello, world” style introduction to devkitPPC, taking you through the installation process and your first program. DevkitPPC contains the following: 
- C and C++ compiler.
- libogc a library for interfacing with the hardware.
- A collection of ported libraries.
- Tools, such as a texture converter.

On the Windows installation a MySys environment is also included, this facilitates the use of shell scripts and make files.

## Installation Process
DevkitPro is the name of the organisation that supplies the toolchain. They produce development kits for a range of Nintendo consoles. We are specifically interested in devkitPPC, which is the development kit aimed at the Nintendo Wii platform.


#### Windows

Firstly, navigate over to the latest version of the graphical installer [here](https://github.com/devkitPro/installer/releases). Click through the wizard and ensure that Wii development option is selected, you can also choose to install any other devkits. After this, just let the installer run and do its job.  
![Windows Installer](/assets/Blog/HelloWorldWii/installer.png)

#### Unix Like (macOS and Linux)

On Unix systems, pacman is the tool used to distribute all devkitPro packages. If you are on an Arch-based system, this should be easy. For systems without pacman, devkitPro maintains their version called *dkp-pacman*. Go to their [latest pacman release](https://github.com/devkitPro/pacman/releases/), scroll down to find a .deb file for Debian based systems, and a .pkg file for macOS.
For systems like Ubuntu, you may need to install *gdebi-core*. Then run the gdebi command on the right .deb file for your architecture. On macOS, take the .pkg and *right-click > open.*

With pacman installed onto your system, we can use the following set of commands to download the Wii-dev package group.  
Sync the Pacman database:
```console
sudo (dkp-)pacman -Sy
```
Install any updates:
```console
sudo (dkp-)pacman -Syu
```
Install the Wii devkit 
```console
sudo (dkp-)pacman -S wii-dev
```

On Linux, devkit installs a script to */etc/profile.d/devkit-env.sh*, which will automatically set the necessary environment variables when you log in. The default settings should be fine, but this file will need editing if you installed to a custom location. After ensuring the environment variables will have the correct contents, log out and log back in to set your environment variables. On macOS, you will need to reboot your device.  

 To ensure that this page always links to up to date information I will also include the links to devkit's [getting started](https://devkitpro.org/wiki/Getting_Started) page and their [pacman explanation](https://devkitpro.org/wiki/devkitPro_pacman). I don't own a mac, so if anyone would like to check these steps for macOS and ensure they're correct, that would be appreciated.
## Building Examples
Navigate to the location that you installed devkitPro *(on Linux this should be /opt/devkitpro/)*, and you'll find a folder called */examples/wii/*. Copy the Wii examples folder to wherever you plan to be developing, open a terminal in your folder *(Windows users use ctrl + shift + right-click > Open PowerShell here)* and enter the command: 
```console
make
```
Those familiar with Linux development will know this command well, Windows developers might not be as sure. When starting a compiler, command-line arguments are added; such as which libraries to link, the location of those libraries etc. Make files automate that process so that you don’t have to specify command-line arguments every single time. Packaging a MySys environment on Windows allows the use of the make tool automatically via command-line. Note, on Windows, be careful which terminal emulator you use; I found git bash had a conflicting version of cygwin.dll, but Powershell and Cmder both worked fine.

Calling the make command in this location activates the makefile, which will loop through all the examples and build all of the Wii examples. Inside each example folder, you'll find a .dol and a .elf file. The .dol file is the raw binary file, and the .elf file is the output of the linker.

 .elf files have debugging information, and .dol files are best for distribution. Both file formats can be executed on the Wii, or more conveniently the Dolphin emulator. 
![example](/assets/Blog/HelloWorldWii/Example.png)

To run these executables on an actual Wii, you will need a Wii with homebrew installed. Navigate to the homebrew menu and find your console's IP address, then you can set the environment variable ***"WIILOAD"*** to *"tcp:IP_ADDRESS"*. From here we can use the wiiload tool to send and load the executable to the console. You can find this tool in the following directory: */devkitPro/tools/bin/.* On Windows, you can drag and drop the binary file onto wiiload.exe, Unix users can use the following command:
```console
wiiload EXECUTABLE COMAND_LINE_ARGS
```
[![ezgif-7-da98a6e46993.gif](https://s2.gifyu.com/images/ezgif-7-da98a6e46993.gif)](https://gifyu.com/image/UR1d)


*(Excuse the terrible screen quality, I bought a cheap HDMI adapter)*
## Hello, World!

In the examples folder, you will find some source in a folder called "Template"; this is your standard *"Hello, world!"* example. This example is quite well documented already by the devkitPro team, but I would like to take some time to walkthrough  add some additional information.

Firstly *stdio* and *stdlib* are not the original version of the standard C libraries, but instead the ported ones supplied by devkitPPC. *gccore* stands for GameCube Core; this is the library that handles the primary hardware interaction. The Wii is essentially an overclocked Gamecube, so the GC libraries and games are compatible  with the Wii. *xfb* is a void pointer to the chunk of memory that we will allocate for the framebuffer. The *rmode* variable represents the video mode that the Wii is expecting to output. From this variable, we can extract information such as the refresh rate and screen resolution.

```c
#include <stdio.h>
#include <stdlib.h>
#include <gccore.h>
#include <wiiuse/wpad.h>

static void *xfb = NULL;
static GXRModeObj *rmode = NULL;
```
Enter inside the main function, an interesting thing to note is that we can pass command line arguments. Next, we enter into a series of functions supplied by *gccore* and *wpad*; you can find the header files for these inside the *libogc/include* folder. You can read the implementation for these functions on devkitPro's [GitHub](https://github.com/devkitPro/libogc/tree/master/libogc). For example, *VIDEO_Init()* looks to set some video hardware registers using an internal variable called *"_viReg"*.  
The comments here are quite self-explanatory. First, start the video system and the plugged-in controllers, Ask the Wii what video settings it is currently using; and finally, create a framebuffer matching those specifications.
```c
//---------------------------------------------------------------------------------
int main(int argc, char **argv) {
//---------------------------------------------------------------------------------
	// Initialise the video system
	VIDEO_Init();

	// This function initialises the attached controllers
	WPAD_Init();

	// Obtain the preferred video mode from the system
	// This will correspond to the settings in the Wii menu
	rmode = VIDEO_GetPreferredMode(NULL);

	// Allocate memory for the display in the uncached region
	xfb = MEM_K0_TO_K1(SYS_AllocateFramebuffer(rmode));
```
Next, we set up the character console, so that we can print to it. Notice, *xfb* is just a pointer to the location of the framebuffer; the details of the framebuffer are stored separately in the video mode variable. After setting the registers required for this framebuffer and video mode, flush the register changes to the hardware. Wait for these updates to finish, this will be announced by entering into the virtual blanking period.  
```c
	// Initialise the console, required for printf
	console_init(xfb,20,20,rmode->fbWidth,rmode->xfbHeight,
		rmode->fbWidth*VI_DISPLAY_PIX_SZ);

	// Set up the video registers with the chosen mode
	VIDEO_Configure(rmode);

	// Tell the video hardware where our display memory is
	VIDEO_SetNextFramebuffer(xfb);

	// Make the display visible
	VIDEO_SetBlack(FALSE);

	// Flush the video register changes to the hardware
	VIDEO_Flush();

	// Wait for Video setup to complete
	VIDEO_WaitVSync();
	if(rmode->viTVMode&VI_NON_INTERLACE) VIDEO_WaitVSync();
```
Now we can print to the console. *\x1b[* is an example of an escape code introducer. For example, to use the escape code *2J* (to clear the console), the statement would be *"printf("\x1b[2J");"*.
```c
	// The console understands VT terminal escape codes
	// This positions the cursor on row 2, column 0
	// we can use variables for this with format codes too
	// e.g. printf ("\x1b[%d;%dH", row, column );
	printf("\x1b[2;0H");
	printf("Hello World!");
```
This while loop is similar to a windowing loop; it prevents the program from closing until the user presses the home button. *VIDEO_WaitVSync* will pause the current thread until the virtual blanking period has started; this is essentially waiting for the next frame before scanning the connected controllers.  
```c
	while(1) {
		// Call WPAD_ScanPads each loop, this reads the
		// latest controller states
		WPAD_ScanPads();

		// WPAD_ButtonsDown tells us which buttons were pressed in this 
		// loop this is a "one shot" state which will not fire again 
		// until the button has been released
		u32 pressed = WPAD_ButtonsDown(0);

		// We return to the launcher application via exit
		if ( pressed & WPAD_BUTTON_HOME ) exit(0);

		// Wait for the next frame
		VIDEO_WaitVSync();
	}
	return 0;
```

Finally, let's look at the last component: The make file, which will be the basis for all your projects. Firstly, the file checks for the environment variable, which points to the devkitPPC install. Then, include the instructions of building for the Wii.   
You can look into the wii_rules file, but in summary: wii_rules sets the locations for the default libraries, adds the gekko compiler to the path, sets the details for the Wii Cpu and finally set the output for .dol and .elf files.
```make
ifeq ($(strip $(DEVKITPPC)),)
$(error "Please set DEVKITPPC in your environment. export DEVKITPPC=
		<path to>devkitPPC")
endif
include $(DEVKITPPC)/wii_rules
```
These variables set the structure of the project. Target is the name of the executables, which will be the name of the directory. The .a and .o files get compiled and output into the Sources directory. The files in Build then get linked and outputted as the targets. The directory containing your source code needs to placed inside the Sources variable. Data points to the location for any extra assets. The Includes variable points to the location of any additional header files.
```make
TARGET		:=	$(notdir $(CURDIR))
BUILD		:=	build
SOURCES		:=	source
DATA		:=	data
INCLUDES	:=
```
The final section that you might find yourself customising is the library inputs to the linker. The libraries listed here are the minimum required for full use of the Wii hardware.
- libWiiUse : Wii remote library.
- libbte : Bluetooth to connect to the Wiimotes. 
- libogc : Wii hardware interface library.
- libm : maths library.

Additional library directories also get included so that the linker knows where to look for external libraries. The rest of the make file automates finding of all the source files included in the project.

```make
#---------------------------------------------------------------------------------
# any extra libraries we wish to link with the project
#---------------------------------------------------------------------------------
LIBS	:=	-lwiiuse -lbte -logc -lm

#---------------------------------------------------------------------------------
# list of directories containing libraries, this must be the top level containing
# include and lib
#---------------------------------------------------------------------------------
LIBDIRS	:=
``` 
Now that we understand how the components interact, we can build the template project. This project is simply a "Hello, world!" example, but we know whats going on behind the scenes. We have a good base for building more complicated projects, but for now, let's enjoy the fruits of our labour.


![makefile](/assets/Blog/HelloWorldWii/make.png)


![makefile](/assets/Blog/HelloWorldWii/HelloWorld.png)

## My Development Set Up

A development environment is one of the most important things in development, allow me to show you my personal set up. I use Visual Studio Code, and use it to open the project root folder. (That's the one with the make file in it) Next we need to set up the C and C++ configuration, this allows VSCode to find all the header files and highlight any syntax errors. I've added another environment variable called _DEVKIT_PATH_, I've done this because the Windows environment variable for DevkitPro points to the MySys location, not the Windows location.

As a quick break down, we add the include locations for the ported standard libraries, and libogc. We have to add a define for "_GEKKO_" which is the name of the processor in the Wii. We add a path that points to the compiler used to create the build, this will also outline compiler errors. Finally the intellisense mode was automatically selected by VSCode, but it seems to work well.

```json
{
    "name": "Wii",
    "includePath": [
        "${workspaceFolder}/**",
        "${DEVKIT_PATH}/devkitPPC/powerpc-eabi/include/**",
        "${DEVKIT_PATH}/libogc/include/**"
    ],
    "defines": [
        "GEKKO",
        "_DEBUG",
        "UNICODE",
        "_UNICODE"
    ],
    "compilerPath": "${DEVKIT_PATH}/devkitPPC/bin/powerpc-eabi-gcc",
    "cStandard": "c99",
    "cppStandard": "c++14",
    "intelliSenseMode": "linux-gcc-x86"
}
```

There's one last quality of life improvement, for automatically running builds after compiling. The makefile that we've been supplied with has a run command, this command will usually attempt to run the executable on a real Wii using Wiiload. However I prefer to do the bulk of my development in dolphin, and then after a development cycle ensure that it works on real hardware. So we can change the run rule to the following:

```make
run:
	<Path To Dolphin Emulator>\\Dolphin.exe -b $(OUTPUT).elf
```

The _"-b"_ tag ensures that dolphin runs in "batch" mode, which will run a Dolphin instance running only the .elf file. Finally we have a full development environment in which we can write, compile and run code for the Nintendo Wii.