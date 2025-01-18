# MinUI CodeBase Analysis

## General
At the highest level, MinUI (much like OnionOS) is not an OS, but rather a UI/configuration platform laid on top of the device's stock OS. This comes with some interesting limitations and benefits. 

The cool part is that boring things like input, bluetooth/wifi, display drivers come free - we don't have to implement any of those. The annoying part is that we're locked into existing architecture/drivers. In the event of something like the RGXX where the manufacturer hasn't extracted all the device's power out of it at launch, we can't throw our own drivers at it.

## How does it work? 

Start by reading the `/usr/trimui/bin/runtrimui.sh` section of Services.md before diving in here.

As mentioned there, the Bricks initialization service looks for 3 shell scripts (`/mnt/SDCARD/trimui/app/premainui.sh`, `/mnt/SDCARD/trimui/app/MainUI` and  `/mnt/SDCARD/trimui/app/preload.sh`) , and runs them as root before the stock OS if they exist. This opens up a ton of potential for loading...we can literally do whatever we want here. 

In MinUIs case, they just replace `/usr/trimui/bin/runtrimui.sh` and have it call `/mnt/SDCARD/.tmp_update/updater`
```sh
#!/bin/sh

# NOTE: becomes .tmp_update/updater

INFO=`cat /proc/cpuinfo 2> /dev/null`
case $INFO in
*"sun8i"*)
	if [ -d /usr/miyoo ]; then
		PLATFORM="my282"
	else
		PLATFORM="trimuismart"
	fi
	;;
*"SStar"*)
	PLATFORM="miyoomini"
	;;
*"TG5040"*)
	PLATFORM="tg5040"
	;;
*"TG3040"*)
	PLATFORM="tg3040"
	;;
esac

# fallback for tg5040 20240413 recovery firmware
# TODO: doublecheck interaction with tg3040
# might need/want to strings /usr/trimui/bin/MainUI during install/update
# and store platform in a text file
if [ -z "$PLATFORM" ] && [ -f /usr/trimui/bin/runtrimui.sh ]; then
	PLATFORM="tg5040"
fi

/mnt/SDCARD/.tmp_update/$PLATFORM.sh # &> /mnt/SDCARD/boot.txt

# force shutdown so nothing can modify the SD card
echo s > /proc/sysrq-trigger
echo u > /proc/sysrq-trigger
echo o > /proc/sysrq-trigger
```

This will set the platform env variable and run the associated platform specific start script (e.g `/mnt/SDCARD/.tmp_update/tg3040.sh`).

In the case of `/mnt/SDCARD/.tmp_update/tg3040.sh` we have the following script which

- Sets the CPU to the max performance of 2.0Ghz
- Checks if there's an update to run (MinUI.zip) and extracts it to the ``/mnt/SDCARD/.system` directory if necessary (this is part of the core installation process).
- Runs the MinUI.pak/launch.sh and continues to run and prevents any other processes from doing anything afterward by immediately powering off.
(I don't quite understand how the while loop exits other than if `MinUI.pak/launch.sh` deletes itself?).



```sh
#!/bin/sh
# NOTE: becomes .tmp_update/tg3040.sh

PLATFORM="tg3040"
SDCARD_PATH="/mnt/SDCARD"
UPDATE_PATH="$SDCARD_PATH/MinUI.zip"
SYSTEM_PATH="$SDCARD_PATH/.system"

echo userspace > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
CPU_PATH=/sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed
CPU_SPEED_PERF=2000000
echo $CPU_SPEED_PERF > $CPU_PATH

# install/update
if [ -f "$UPDATE_PATH" ]; then 
	echo ok
	export LD_LIBRARY_PATH=/usr/trimui/lib:$LD_LIBRARY_PATH
	export PATH=/usr/trimui/bin:$PATH

	# leds_off
	echo 0 > /sys/class/led_anim/max_scale
	echo 0 > /sys/class/led_anim/max_scale_lr
	echo 0 > /sys/class/led_anim/max_scale_f1f2

	cd $(dirname "$0")/$PLATFORM
	if [ -d "$SYSTEM_PATH" ]; then
		./show.elf ./updating.png
	else
		./show.elf ./installing.png
	fi

	./unzip -o "$UPDATE_PATH" -d "$SDCARD_PATH" # &> /mnt/SDCARD/unzip.txt
	rm -f "$UPDATE_PATH"

	# the updated system finishes the install/update
	# $SYSTEM_PATH/$PLATFORM/bin/install.sh # &> $SDCARD_PATH/install.txt
fi

LAUNCH_PATH="$SYSTEM_PATH/$PLATFORM/paks/MinUI.pak/launch.sh"
while [ -f "$LAUNCH_PATH" ] ; do
	"$LAUNCH_PATH"
done

poweroff # under no circumstances should stock be allowed to touch this card
```

`/mnt/SDCARD/.system/tg3040/paks/MinUI.pak/launch.sh` is copied from `skeleton/SYSTEM/tg3040/paks/MinUI.pak/launch.sh` which at a high level: 

- "Mounts" the SD caard paths for standard video game stuff (Roms, Bios, etc)
- Brilliantly exports the platform specific linker/PATH directories.
- Ensures inputs are turned on.
- Sets the CPU to the max speed of 2 Ghz
- It looks like it checks for a operation that was request to execute at `/tmp/minui_exec`, then execute anything that was request to run at `/tmp/next`.


```sh
#!/bin/sh
# MiniUI.pak

# recover from readonly SD card -------------------------------
# touch /mnt/writetest
# sync
# if [ -f /mnt/writetest ] ; then
# 	rm -f /mnt/writetest
# else
# 	e2fsck -p /dev/root > /mnt/SDCARD/RootRecovery.txt
# 	reboot
# fi

#######################################

export PLATFORM="tg3040"
export SDCARD_PATH="/mnt/SDCARD"
export BIOS_PATH="$SDCARD_PATH/Bios"
export ROMS_PATH="$SDCARD_PATH/Roms"
export SAVES_PATH="$SDCARD_PATH/Saves"
export SYSTEM_PATH="$SDCARD_PATH/.system/$PLATFORM"
export CORES_PATH="$SYSTEM_PATH/cores"
export USERDATA_PATH="$SDCARD_PATH/.userdata/$PLATFORM"
export SHARED_USERDATA_PATH="$SDCARD_PATH/.userdata/shared"
export LOGS_PATH="$USERDATA_PATH/logs"
export DATETIME_PATH="$SHARED_USERDATA_PATH/datetime.txt"

mkdir -p "$BIOS_PATH"
mkdir -p "$ROMS_PATH"
mkdir -p "$SAVES_PATH"
mkdir -p "$USERDATA_PATH"
mkdir -p "$LOGS_PATH"
mkdir -p "$SHARED_USERDATA_PATH/.minui"

#######################################

#PD11 pull high for VCC-5v
echo 107 > /sys/class/gpio/export
echo -n out > /sys/class/gpio/gpio107/direction
echo -n 1 > /sys/class/gpio/gpio107/value

#rumble motor PH3
echo 227 > /sys/class/gpio/export
echo -n out > /sys/class/gpio/gpio227/direction
echo -n 0 > /sys/class/gpio/gpio227/value

#DIP Switch PH19
echo 243 > /sys/class/gpio/export
echo -n in > /sys/class/gpio/gpio243/direction

#######################################

export LD_LIBRARY_PATH=$SYSTEM_PATH/lib:/usr/trimui/lib:$LD_LIBRARY_PATH
export PATH=$SYSTEM_PATH/bin:/usr/trimui/bin:$PATH

# leds_off
echo 0 > /sys/class/led_anim/max_scale
echo 0 > /sys/class/led_anim/max_scale_lr
echo 0 > /sys/class/led_anim/max_scale_f1f2

# start stock gpio input daemon
trimui_inputd &

echo userspace > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
CPU_PATH=/sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed
CPU_SPEED_PERF=2000000
echo $CPU_SPEED_PERF > $CPU_PATH

# disable internet stuff
# killall MtpDaemon
# killall wpa_supplicant
# ifconfig wlan0 down

keymon.elf & # &> $SDCARD_PATH/keymon.txt &

#######################################

AUTO_PATH=$USERDATA_PATH/auto.sh
if [ -f "$AUTO_PATH" ]; then
	"$AUTO_PATH"
fi

cd $(dirname "$0")

#######################################

EXEC_PATH="/tmp/minui_exec"
NEXT_PATH="/tmp/next"
touch "$EXEC_PATH"  && sync
while [ -f $EXEC_PATH ]; do
	minui.elf &> $LOGS_PATH/minui.txt
	echo $CPU_SPEED_PERF > $CPU_PATH
	echo `date +'%F %T'` > "$DATETIME_PATH"
	sync
	
	if [ -f $NEXT_PATH ]; then
		CMD=`cat $NEXT_PATH`
		eval $CMD
		rm -f $NEXT_PATH
		echo $CPU_SPEED_PERF > $CPU_PATH
		echo `date +'%F %T'` > "$DATETIME_PATH"
		sync
	fi
done

poweroff # just in case
```

`<MINUI_CODE>workspace/all/minui/minui.c:945` manages putting the next command in the queue (`NEXT_PATH`) in  its `static void queueNext(char* cmd)` function.



# Dev stuff
## Code Pre-requisites
It is recommended (but not entirely necessary) to do all development in an x86_64 environment such as common AMD/Intel CPUs as this project relies on some bespoke docker containers that are only built for one platform. This can be overcome on an M1 mac by changing some of the docker run calls to explicitly use the `--platform linux/arm64/v8` docker flag. However, there will be other minor build issues that need to be addressed. For the easiest experience, avoid working on Apple Silicon.

Some cmake library along with zip are required to build/package the project:

```bash
sudo apt update
sudo apt install cmake gcc zip
```

Note: There are some hiccups with the makefile that make it difficult to build without modification. The tidy command ignores your PLATFORM flag and assumes you've built for the rg35xxplus and rg40xxcube.

## Code Overview

As mentioned above, we're dealing with a UI layer on top of stock OS. The code structure is as follows:

```
MinUI/
├─ build/
├─ github/
├─ toolchains/
├─ workspace/
├─ makefile
└─ makefile.toolchain
```

### Makefile & Makefile.toolchain
These files lay out the instructions for building the project. The `makefile.toolchain` file is used to clone/build/run the current toolchains container and define environment variables that distinguish the paths of the host machine and toolchain container's workspace.

### Build Directory
This is where your compiled binaries will end up after you've built the project.

### Github Directory
Files necessary to generate/host the readme screenshots.

### Toolchains Directory
The dependent docker build toolchains for specific devices. They setup a similar linux environment to that of the host device (install the same libraries) and drop you into a host shell to run your test applications from. For example:
- tg3040-toolchain directs you to [Trimui's own toolchain](https://github.com/trimui/toolchain_sdk_smartpro/releases)
- RG35XX requires community-made toolchain: [union-rg35xxplus-toolchain](https://github.com/shauninman/union-rg35xxplus-toolchain)

### Workspace Directory
This is where the magic happens - all UI/device specific code is located in this directory.

## Build Overview

### Base Release Contents
The general release of MinUI (`MinUI-<version>-0-base`) contains:

```
├── Bios/           # Empty Bios folders for users to fill in
├── MinUI.zip       # Core OS files used for all platforms
│   ├── .system     # build/SYSTEM folder after 'make tidy'
│   └── .tmp_update # build/BOOT/common folder
├── README.txt
├── Roms/           # Empty Roms folders for users to fill in
├── Saves/          # Empty Saves folders for users to fill in
├── em_ui.sh
└── [device folders]# Device-specific install scripts/files
```

### Extras Release Contents
The extras release (`MinUI-<version>-0-extras`) contains additional/experimental emulators and device-specific apps:

```
├── Bios/           # Additional system BIOS folders
├── Emus/           # Device-specific emulators
├── README.txt
├── Roms/           # Additional system ROM folders
├── Saves/          # Additional system save folders
└── Tools/          # Device-specific tools and utilities
```
## TrimUI Brick (tg3040) Specifics

### Base Release Contents

For the TrimUI Brick, the base release includes:

```
├── Bios/
├── MinUI.zip
│   ├── .system
│   └── .tmp_update
├── README.txt
├── Roms/
│   ├── MD/
│   ├── PS/
│   └── SFC/
├── Saves/
├── em_ui.sh
└── device_folders/
  ├── gkdpixel/
  ├── miyoo/
  ├── miyoo354/
  ├── rg35xx/
  ├── rg35xxplus/
  └── trimui/
```

### Extras Release Structure

The extras release contains additional components:

```
├── Bios/
│   ├── GG/
│   ├── MGBA/
│   ├── NGPC/
│   ├── P8/
│   ├── PCE/
│   ├── PKM/
│   ├── SGB/
│   ├── SMS/
│   └── VB/
├── Emus/
│   └── tg3040/
├── README.txt
├── Roms/
│   ├── Game Boy Advance (MGBA)/
│   ├── Neo Geo Pocket Color (NGPC)/
│   ├── Pico-8 (P8)/
│   ├── Pokémon mini (PKM)/
│   ├── Sega Game Gear (GG)/
│   ├── Sega Master System (SMS)/
│   ├── Super Game Boy (SGB)/
│   ├── Super Nintendo Entertainment System (SUPA)/
│   ├── TurboGrafx-16 (PCE)/
│   └── Virtual Boy (VB)/
├── Saves/
│   └── [matching ROM folders]/
└── Tools/
  └── tg3040/
    ├── Bootlogo.pak/
    ├── Clock.pak/
    ├── Input.pak/
    └── Remove Loading.pak/
```
