# OnionOS CodeBase Analysis

## Dev Notes
### 1-19-25

#### Current changes to try and get it running
1. Added mising libraries and scripts from [aemiii91/miyoomini-toolchain:latest](https://github.com/OnionUI/dev-miyoomini-toolchain) to [my arm64 toolchain](https://github.com/pierceTee/arm64-tg3040-toolchain).
2.  Changed `.tmp_update/updater` to 'tg3040.sh` to match what the `runtrimui.sh` script is expecting after installing minui (I can change this script later).
3. Added line to copy  `Onion/static/build/miyoo/lib/libpadsp.so` to `SDCARD/=/miyoo/app/.tmp_update/lib` ( I should probably just copy this to `usr/trimui/lib`) in `miyoo/app/.tmp_update/install.sh`.
4. Comment out the `mv ./romscreens/*` bit from the `<SDCARD>/miyoo/appp/MainUI`/`static/dist/miyoo/app/MainUI` script to prevent that failure.
5. Change usage of `-not` flag to `!` from `find` command on line 26 of `<SDCARD>/miyoo/appp/MainUI` since the brick's version of find doesn't support it.
6. set `$DEVICE_ID` to `$MODEL_MMP` in  `miyoo/app/.tmp_update/install.sh` / `/mnt/SDCARD/.tmp_update/install.sh`


- Installation got pretty well along with the above changes but fails with 7z. 

Turns out the linking of the .tmp_update libs and binaries isn't allowing me to run those binaries. 

in fact, I can't even run the binaries directly from the folders themselves...I should try getting my own 7z binary and running that.

### 1-20-25

Looks like all the binaries that Onion relies on are 32-bit armhf and the brick doesn't have all the proper 32bit libraries (in 7zs case, `libc:armhf`) installed to run them. Further, I haven't got `opkg` manager working. Looks like I need to transfer those binaries over manually in the install (unless we're compiling these binaries ourselves but I don't think we do). I should try comiling in their default container for sanity check...

First attempt to resolve is to manually move the linux armf 32-bit compatability library `ld-linux-armhf.so.3` to `<SDCARD>/miyoo/app/.tmp_update/lib` this should resolve the issue during install but I'll probably want to move this to `lib` so we don't run into compatibility issues during normal runtime operations.


**UPDATE**: Idk why I'm just now noticing but the mini is 32-bit arm and the brick is 64bit....this is going to be A LOT more work than I thought lol.

I've written scripts to get some required 32bit libs and put them on the brick but the dependency issues continue to cascade, I think it would be better to just redo everything in 64bit unfortunately.

Here's the changes I'm trying to build everything for 64bit.

1. remove `-m32` flags in `third-party/RetroArch-patch/submodules/RetroArch/Makefile`
2. update `-32` references to `-m64` in 

    a. `third-party/RetroArch-patch/submodules/RetroArch/deps/bearssl-0.6/conf/Unix32.mk`

    b. ~~`third-party/RetroArch-patch/submodules/RetroArch/deps/xxHash/Makefile`~~ I don't think this one is required.

    c. `third-party/RetroArch-patch/submodules/RetroArch/deps/xxHash/tests/bench/Makefile`

3. I need to look into replacing `ARMV7` compiler flags with `AMRV8`?

### 1-22-25

Had a few revalations today about my past attemptt: 

#### Running 32bit applications on the brick
Due to the fact that when the OS reads `LD_LIBRARY_PATH` left to right for libraries to link, if I prepend 32bit libraries, then 32bit applications will work fine, but if I run a 64bit application that depends on a library that we also have a 32bit version of, it will first link with the 32bit version and fail to launch. The inverse will happen if we prepend the 64bit library path. The only real solution is to prepend the path with the correct lib path before launching any application (not a realistic soltution).

This should be resolvable by using the bricks `/lib` and `/lib64` folders to place 32 and 64bit libraries respectively, although in practice this requires us to install 32bit versions of every library Onion would require. I think the correct move going forward would be to create a solid 64bit fork of onion that can be ported to other systems.

The alternative here is to simply 

```
root@TinaLinux:/mnt/SDCARD/.tmp_update/logs# cat install.log
:: Check firmware
:: Backup system
:: Remove configs
File /mnt/SDCARD/.tmp_update/onion.pak exists.
File /mnt/SDCARD/.tmp_update/onion.pak has write permission.
No process is currently writing to /mnt/SDCARD/.tmp_update/onion.pak.
File /mnt/SDCARD/.tmp_update/onion.pak size:  bytes.

:: Install Onion
   - Extract '/mnt/SDCARD/.tmp_update/onion.pak' into /mnt/SDCARD
File /mnt/SDCARD/.tmp_update/onion.pak exists.
File /mnt/SDCARD/.tmp_update/onion.pak has write permission.
No process is currently writing to /mnt/SDCARD/.tmp_update/onion.pak.
File /mnt/SDCARD/.tmp_update/onion.pak size:  bytes.

onion.pak exists
File /mnt/SDCARD/.tmp_update/onion.pak exists.
File /mnt/SDCARD/.tmp_update/onion.pak has write permission.
No process is currently writing to /mnt/SDCARD/.tmp_update/onion.pak.
File /mnt/SDCARD/.tmp_update/onion.pak size:  bytes.

7Z is NOT running and should be
onion.pak still exists
:: Installation failed!
```
looks like 7z isn't running correctly...check line 676 of   `miyoo/app/.tmp_update/install.sh`


## Project overview

### Makefile analysis

- Typical usage is expected to utilize the toolchain to build using `make with-toolchain` and following up with `make release`. Contrary to MinUI, the toolchain used does not dictate the resulting architecture the project it sbuild for.

- **Architecture descsions**: I can't yet say with certainty where they're all made but there are certainly a ton of `-m32` and `ARMV7` flags used for retroarch builds throughout. 

### Rules

- **all**: 
    1. Calls `dist`.

- **version**: 
    1. Used to get the `VERSION` string defined at the top of the Makefile.

- **print-version**: 
    1. Pretty prints `VERSION` along with the RetroArch `RA_SUBVERSION` string defined at the top of the Makefile.

- **$(CACHE)/.setup**: 
    1. Uses the `/cache` directory to sync files to the build directory.
    2. Syncs `static/build` files to `/build`.
    3. Syncs `static/dist` files to `/dist`. This is particularly interesting because `dist` seems to house the distribution/release files (lib files included).
    4. Copies over `/lib` library files to `/miyoo/app/.tmp_update/lib` for use on the device. This is where every library file used in installation comes from.
    5. Sets version information into related info files.
    6. Copies all `res` directories from key applications `gameSwitcher`, `chargingState`, `bootScreen`, `themeSwitcher`, `tweaks`, `randomGamePicker`, and `easter`. This is likely the minimum list of applications I need to ensure are ported first.
    7. Copies all script files from `packageManager` and `themeSwitcher` to `/.tmp_update/script/` (these are used for installation and need to be investigated).
    8. Downloads themes for release.
    9. Copies static config files (should be fine to leave alone).
    10. Copies static packages from `static/packages` to `SDCARD/App|Emu|RaApp` and runs `packages/common/apply.sh` on the directories. This copies all the config files to the relevant app folders. It also runs `packages/common/auto_advmenu.rc.sh` which looks like it copies specific advanced menu settings like `rompath`, `imgpath`, and `rel_dir` to RetroArch configs.
    11. Creates fullres files: Running `/.github/create_fullres_files.sh` it adds blank files for DS and SNK emulation that seem to indicate to the emulators to launch in 560p.
    12. Creates an empty setup file potentially to misdirect the Makefile into skipping building if the cache has been built.

- **build**: 
    1. Runs `core`, `apps`, and `external`.

- **core**: 
    1. Begins by running `$(CACHE)/.setup`.
    2. Enters each `src/[Application Name]` folder and builds their content using a standard `make` command, piping the output into `build/.tmp_update/bin`. Seeing how no direct compiler flags seem to be set at the individual `src/[Application Name]/Makefile` level, we should be able to override `CFLAGS` and `LDFLAGS` at the highest level to use 64-bit libraries in order to compile for ARM64.
    3. Copies build dependencies from `/build/.tmp_update/bin` to `/miyoo/app/.tmp_update/bin`, all of which were built in the previous step except 7z. It might actually be compiled as a RetroArch submodule under `/third-party/RetroArch-patch/submodules/RetroArch/deps/7zip`.

- **apps**: 
    1. Begins by running `$(CACHE)/.setup`.
    2. Enters into the associated `src` directory, builds, and copies the build to the associated `[SDCARD]/Apps` directory for `batteryMonitorUI`, `playActivityUI`, `packageManager`, `clock`, and `randomGamePicker`.

- **$(THIRD_PARTY_DIR)/RetroArch-patch/bin/retroarch_miyoo354**:
    1. Builds `third-party/RetroArch-patch`.

- **external**: 
    1. Begins by running `$(CACHE)/.setup` and `$(THIRD_PARTY_DIR)/RetroArch-patch/bin/retroarch_miyoo354` (effectively setting up the build directories and building RetroArch).
    2. Copies RetroArch `RetroArch-patch/bin` directory to `build/Retroarch` and runs `build/.tmp_update/script/build_ext_cache.sh` which looks like it builds a per-core cache.
    3. Builds third-party `Search`, `Filter`, `Terminal`, and `DinguxCommander` apps.

- **dist**:
    1. Begins by running `build`.
    2. Creates `configs`, `retroarch`, and `Onion` packages using the host machine's `7zip`.

- **release**: 
    1. Begins by running `dist`.
    2. Zips up the release directory into its own package.

- **clean**: 
    1. Removes `build`, `cache`, `test`, `dist`, and `configs` directories.
    2. Ensures `$(CACHE)/.setup` is removed so we rebuild the cache.
    3. Removes any `.o` files from `include` and `src` directories.

- **deepclean**: 
    1. Calls `clean`, but also clears out `RetroArch-patch`, `SearchFilter`, `Terminal`, and `DinguxCommander` builds.

- **dev**:
    1. Calls `clean`.
    2. Calls unset `MAKE_DEV` string (might be nice to override this flag during testing).

- **asan**:
    1. Calls `clean`.
    2. Calls unset `MAKE_ASAN_` string.

- **git-clean**:
    1. Cleans up git files by running `git clean` and excluding vscode files.

- **git-submodules**:
    1. Updates git submodules.

- **pwd**:
    1. Emits the root directory as a string.

- **$(CACHE)/.docker**: 
    1. Re-pulls the toolchain docker image.
    2. Makes the cache directory and creates a `cache/.docker` file to indicate it ran.

- **toolchain**: 
    1. Calls `$(CACHE)/.docker`.
    2. Puts the user in a bash shell of the toolchain image with the project mounted in the `/workspace` directory.

- **with-toolchain**:
    1. Calls `$(CACHE)/.docker`.
    2. Runs the container headless and calls `make` with any of the supplied user commands.

- **patch**: 
    1. Runs `.github/create_patch.sh`.

- **external-libs**:
    1. Rebuilds `include/SDL`.

- **test**:
    1. Builds `external-libs`.
    2. Runs `infoPanel_test_data` test.

- **static-analysis**:
    1. Runs `external-libs`.
    2. Runs `cppcheck` on the `include` and `src` directories.

- **format**:
    1. Applies the clang format styling to your working directory.


So when running `make`, the following sequence of calls is made:

1. `$(CACHE)/.setup`
2. `core`
3. `apps`
4. `$(THIRD_PARTY_DIR)/RetroArch-patch/bin/retroarch_miyoo354`
5. `external`
## Building Onion

Checkout the [official onion dev documentation](https://onionui.github.io/docs/dev/setup) before continuing. 

The Onion project follows the "build in toolchain" motif used by most of these projects. 

The toolchain in question is [aemiii91/miyoomini-toolchain:latest](https://github.com/OnionUI/dev-miyoomini-toolchain) which follows a similar structure to [my own arm64 toolchain fork for the brick](https://github.com/pierceTee/arm64-tg3040-toolchain). 

Notable differences:
- Despite also compiling for arm64. theirs doesn't specify arm64 as their debian image. 
- SDL version differences: 

  | Brick Toolchain      | Onion Toolchain     |
  |-----------|------------|
  | libsdl2-image-2.0-0, libsdl2-image-dev  | libsdl-image1.2-dev  | 
  |libsdl2-ttf-2.0-0  | libsdl-ttf2.0-dev |

- Other library differences: 
  | Brick Toolchain      | Onion Toolchain     |
  |-----------|------------|
  | N/A  | libgtest-dev    |
  | N/A | cppcheck |
  | N/A | p7zip-full |
  | N/A | locales |

- Their toolchain setup sqlite and gtest with 
  ```sh
  RUN ./setup-sqlite.sh

  RUN ./setup-gtest.sh
  ```


For the time being I'll update my toolchain to more closely match [aemiii91/miyoomini-toolchain:latest](https://github.com/OnionUI/dev-miyoomini-toolchain) while maintaing the library versions found on the brick, then try building Onion.


### **_update_**: 

  I've updated my toolchain to include the missing packages on the [onion branch of my toolchain](https://github.com/pierceTee/ arm64-tg3040-toolchain/tree/onion)

  I can now build that toolchain, swap the toolchain flag on in the OnionOS Makefile to `arm64-tg3040-onion-toolchain:latest`, and  build just fine. 

  To package, I just have to mount the OnionOS workspace to a directory in my container, exec into the container and run `make  release`. 

  example: `docker run --platform linux/arm64 -it --rm -v "/home/pierce/dev/TrimUIOnionTest/":/root/workspace   arm64-tg3040-onion-toolchain /bin/bash`

**note**: I'd like to set all this up in scripts once I start building more regularly. 

## Installation process

Similar to MinUI Onion uses a `<SDCARD>/.tmp_update` folder with scripts that I assume get autorun by the miyoo OS on boot. 


1. `<Onion_Release>/.tmp_update` runs two two files: 
 
    a. `/SDCARD/.tmp_update/updater`: A forwarder to run `/mnt/SDCARD/.tmp_update`
      ```sh
      #!/bin/sh
      cd /mnt/SDCARD/.tmp_update
      ./runtime.sh
      ```
      b.  `/SDCARD/.tmp_update/runtime.sh`: The equivalent to `tg3040.sh` in MinUI, its just a forwarder to run `mnt/SDCARD/    miyoo/app`

      ```sh
      #!/bin/sh
      cd /mnt/SDCARD/miyoo/app
      ./MainUI
      ```

2. `<SDCARD>/miyoo/appp/MainUI`

  The primary functios of this script are to:
  - Cancel any currently running instances of MainUI 
  - Clear out old onion files.
  - Move `miyoo/app/.tmp_update/*` to `/mnt/SDCARD/.tmp_update`
  - Run `miyoo/app/.tmp_update/updater` (which is now copied to `/mnt/SDCARD/.tmp_update/updater`)

    ```sh
    #!/bin/sh
    echo $0 $*
    # Sets appdir to CWD
    appdir=`cd -- "$(dirname "$0")" >/dev/null 2>&1; pwd -P`
    sysdir=/mnt/SDCARD/.tmp_update

    # Checks for `SDCARD/miyoo/app/.tmp_update/[install.sh|onion.pak]`
    if [ ! -d "$appdir/.tmp_update" ] || [ ! -f "$appdir/.tmp_update/install.sh" ] || [ ! -f "$appdir/.tmp_update/onion.pak" ]; then
        # Cancel installation if files are missing
        rm -- "$0"
        reboot
        sleep 10
    fi

    if [ -d $sysdir ]; then
        cd $sysdir

        # Move romScreens to new location
        mkdir -p /mnt/SDCARD/Saves/CurrentProfile/romScreens
        mv ./romScreens/* /mnt/SDCARD/Saves/CurrentProfile/romScreens/

        if [ -f ./bin/installUI ]; then
            mkdir -p onionVersion
            ./bin/installUI --version > ./onionVersion/previous_version.txt
        fi

        # Remove old system files, except dotfiles and onionVersion
        find * -maxdepth 0 -not -name "onionVersion" -not -name "config" -exec rm -rf {} \;

        # Move dotfiles to new location
        mkdir -p config
        mv .* config/
    fi

    # Move installation files
    # move miyoo/app/.tmp_update/* to /mnt/SDCARD/.tmp_update
    cd $appdir
    mkdir -p $sysdir
    mv .tmp_update/* $sysdir

    cd $sysdir
    ./updater

    reboot
    sleep 10
    ```

3. `miyoo/app/.tmp_update/updater`/`/mnt/SDCARD/.tmp_update/updater`

    The primary functions here are: 
    1. Ensure we're executing from the correct directory `/mnt/SDCARD/.tmp_update/updater`.
    2. Run the ./install.sh script and reboot the device.

    ```sh
    #!/bin/sh
    # Ensure we're executing from the current working directory.
    sysdir=$(
        cd -- "$(dirname "$0")" > /dev/null 2>&1
        pwd -P
    )

    cd $sysdir
    mkdir -p logs

    ./install.sh 2>&1 > ./logs/install.log

    # Turn off after update
    echo "Install script exited unexpectedly"
    reboot
    ```


4. `miyoo/app/.tmp_update/install.sh` / `/mnt/SDCARD/.tmp_update/install.sh`
    
   ```sh
   #!/bin/sh
   sysdir=$(
       cd -- "$(dirname "$0")" > /dev/null 2>&1
       pwd -P
   )
   miyoodir=/mnt/SDCARD/miyoo
   
   CORE_PACKAGE_FILE="$sysdir/onion.pak"
   RA_PACKAGE_FILE="/mnt/SDCARD/RetroArch/retroarch.pak"
   RA_VERSION_FILE="/mnt/SDCARD/RetroArch/onion_ra_version.txt"
   RA_PACKAGE_VERSION_FILE="/mnt/SDCARD/RetroArch/ra_package_version.txt"
    
    ## -- @Pierce Linking their own lib directory to the system to use whatever libraries they want.
   export LD_LIBRARY_PATH="/lib:/config/lib:/customer/lib:$sysdir/lib"

   # -- @Pierce Adding  /mnt/SDCARD/miyoo/app/.tmp_update to the PATH
   export PATH="$sysdir/bin:$PATH"
   unset LD_PRELOAD
   
   ## -- @Pierce These might need to get changed around
   MODEL_MM=283
   MODEL_MMP=354
   
   install_ra=1
   
   version() {
       echo "$@" | tr -d [:alpha:] | awk -F'[.-]' '{ printf("%d%03d%03d%03d\n", $1,$2,$3,$4); }'
   }
   
   # globals
   total_core=0
   total_ra=0
   
   main() {
       # init_lcd
       if [ "$(ps | grep dev/l | grep -v grep)" == "" ]; then
           cat /proc/ls
           sleep 1
       fi
        ## -- @Pierce These are defined below
       check_device_model
       check_install_ra
   
       # Start the battery monitor
       if [ $DEVICE_ID -eq MODEL_MM ]; then
           # init charger detection
           gpiodir=/sys/devices/gpiochip0/gpio
           if [ ! -f $gpiodir/gpio59/direction ]; then
               echo 59 > /sys/class/gpio/export
               echo "in" > $gpiodir/gpio59/direction
           fi
       fi
   
       # init backlight
       pwmdir=/sys/class/pwm/pwmchip0
       echo 0 > $pwmdir/export
       echo 800 > $pwmdir/pwm0/period
       echo 80 > $pwmdir/pwm0/duty_cycle
       echo 1 > $pwmdir/pwm0/enable
   
       killall keymon
   
       check_firmware
   
       if [ ! -d /mnt/SDCARD/.tmp_update/onionVersion ]; then
           run_installation 1 0
           cleanup
           return
       fi
   
       # Start the battery monitor
       cd $sysdir
       batmon 2>&1 > /dev/null &
   
       detectKey 1
       menu_pressed=$?
   
       if [ $menu_pressed -eq 0 ]; then
           prompt_update
           cleanup
           return
       fi
   
       run_installation 0 0
       cleanup
   }
   
   prompt_update() {
       # Prompt for update or fresh install
       prompt -r -m "Welcome to the Onion installer!\nPlease choose an action:" \
           "Update (keep settings)" \
           "Reinstall (reset settings)" \
           "Update OS/RetroArch only"
       retcode=$?
   
       if [ $retcode -eq 0 ]; then
           # Update (keep settings)
           run_installation 0 0
       elif [ $retcode -eq 1 ]; then
           prompt -r -m "Warning: Reinstall will reset everything,\nremoving any custom emus or apps." \
               "Yes, reset my system" \
               "Cancel"
           retcode=$?
   
           if [ $retcode -eq 0 ]; then
               # Reinstall (reset settings)
               run_installation 1 0
           else
               prompt_update
           fi
       elif [ $retcode -eq 2 ]; then
           # Update OS/RetroArch only
           run_installation 0 1
       else
           # Cancel (can be reached if pressing POWER)
           return
       fi
   }
   
   cleanup() {
       echo ":: Cleanup"
       cd $sysdir
       rm -f \
           /tmp/.update_msg \
           .installed \
           install.sh
   
       # Remove dirs if empty
       rmdir /mnt/SDCARD/Backup/saves
       rmdir /mnt/SDCARD/Backup/states
       rmdir /mnt/SDCARD/Backup
   
       # Clean zips if still present
       rm -f $CORE_PACKAGE_FILE
       rm -f $RA_PACKAGE_FILE
       rm -f $RA_PACKAGE_VERSION_FILE
   
       # Remove update trigger script
       rm -f /mnt/SDCARD/miyoo/app/MainUI
   
       rm -f /mnt/SDCARD/miyoo354/app/MainUI
       rmdir /mnt/SDCARD/miyoo354/app
       rmdir /mnt/SDCARD/miyoo354
   }
   
   DEVICE_ID=0
   
   ## == @Pierce: This obviously needs to be overwritten
   check_device_model() {
       DEVICE_ID=$([ -f /customer/app/axp_test ] && echo $MODEL_MMP || echo $MODEL_MM)
       echo -n "$DEVICE_ID" > /tmp/deviceModel
   }
   
   ## == @Pierce: This is probably fine to leave alone.
   check_install_ra() {
       # An existing version of Onion's RetroArch exist
       if [ -f $RA_VERSION_FILE ] && [ -f $RA_PACKAGE_VERSION_FILE ]; then
           local current_ra_version=$(cat $RA_VERSION_FILE)
           local package_ra_version=$(cat $RA_PACKAGE_VERSION_FILE)
   
           # Skip installation if current version is up-to-date.
           if [ $(version $current_ra_version) -ge $(version $package_ra_version) ]; then
               install_ra=0
               echo "RetroArch is up-to-date!"
           fi
       fi
   
       if [ ! -f "$RA_PACKAGE_FILE" ]; then
           install_ra=0
       fi
   }
   ## == @Pierce: This is unused in this pull but present in the 4.0.3 release
   get_install_stats() {
       total_core=$(zip_total "$CORE_PACKAGE_FILE")
       total_ra=0
   
       if [ -f $RA_PACKAGE_FILE ]; then
           total_ra=$(zip_total "$RA_PACKAGE_FILE")
       fi
   
       echo "STATS"
       echo "Install RA check:" $install_ra
       echo "Onion total:" $total_core
       echo "RetroArch total:" $total_ra
   }
   
   remove_configs() {
       echo ":: Remove configs"
       rm -f /mnt/SDCARD/RetroArch/.retroarch/retroarch.cfg
   
       rm -rf \
           /mnt/SDCARD/Saves/CurrentProfile/config/* \
           /mnt/SDCARD/Saves/GuestProfile/config/*
   }
   
   verify_file() {
       local file="/mnt/SDCARD/.tmp_update/onion.pak"
   
       if [ -f "$file" ]; then
           echo "File $file exists."
   
           if [ -w "$file" ]; then
               echo "File $file has write permission."
           else
               echo "File $file does not have write permission."
           fi
   
           if fuser "$file" > /dev/null; then
               echo "Something is currently writing to $file."
           else
               echo "No process is currently writing to $file."
           fi
   
           size=$(stat -c %s "$file")
           echo "File $file size: $size bytes."
       else
           echo "File $file does not exist."
       fi
   
       echo ""
   }
   
   run_installation() {
       reset_configs=$1
       system_only=$2
   
       killall batmon
      ## == @Pierce: This is unused in this pull but present in the 4.0.3 release
       #get_install_stats
   
       rm -f /tmp/.update_msg 2> /dev/null
       rm -f $sysdir/config/currentSlide 2> /dev/null
   
       # Show installation progress
       cd $sysdir
       installUI &
       sync
       sleep 1
   
       verb="Updating"
       verb2="Update"
   
       if [ $reset_configs -eq 1 ]; then
           verb="Installing"
           verb2="Installation"
           echo "Preparing installation..." >> /tmp/.update_msg
   
           # Backup important stock files
           backup_system
   
           remove_configs
           if [ -f $RA_PACKAGE_FILE ]; then
               maybe_remove_retroarch
               install_ra=1
           fi
   
           # Remove stock folders.
           cd /mnt/SDCARD
           rm -rf App Emu RApp miyoo
   
       elif [ $system_only -ne 1 ]; then
           echo "Preparing update..." >> /tmp/.update_msg
   
           # Ensure packages are fresh !
           rm -rf /mnt/SDCARD/App/PackageManager 2> /dev/null
   
           remove_old_search
       fi
   
       if [ $install_ra -eq 1 ]; then
           verify_file
           install_core "1/2: $verb Onion..."
           install_retroarch "2/2: $verb RetroArch..."
       else
           verify_file
           install_core "1/1: $verb Onion..."
           echo "Skipped installing RetroArch"
           rm -f $RA_PACKAGE_FILE
           rm -f $RA_PACKAGE_VERSION_FILE
       fi
   
       if [ $reset_configs -eq 0 ]; then
           restore_ra_config
       fi
   
       install_configs $reset_configs
   
       run_migration_scripts
   
       if [ -d "/mnt/SDCARD/Emu/drastic" ]; then
           echo "Migrating drastic ..."
           cd /mnt/SDCARD/.tmp_update/script
           ./drastic_migration.sh
       fi
   
       echo "Finishing up - Get ready!" >> /tmp/.update_msg
   
       if [ $reset_configs -eq 1 ]; then
           refresh_roms
       fi
   
       #########################################################################################
       #                                  Installation is done                                 #
       #                           Show quick guide and package manager                        #
       #########################################################################################
   
       if [ $system_only -ne 1 ]; then
           if [ $reset_configs -eq 1 ]; then
               cp -f $sysdir/res/miyoo${DEVICE_ID}_system.json /mnt/SDCARD/system.json
           fi
   
           # Start the battery monitor
           cd $sysdir
           batmon 2>&1 > /dev/null &
   
           # Reapply theme
           system_theme="$(/customer/app/jsonval theme)"
           active_theme="$(cat ./config/active_theme)"
   
           if [ -d "$(cat ./config/active_icon_pack)" ] && [ "$system_theme" == "$active_theme" ] && [ -d  "$system_theme" ]; then
               themeSwitcher --update
           else
               themeSwitcher --update --reapply_icons
           fi
   
           touch $sysdir/.installed
           sync
   
           # Show quick guide
           if [ $reset_configs -eq 1 ]; then
               cd /mnt/SDCARD/App/Onion_Manual/
               ./launch.sh
           fi
   
           # Launch package manager
           cd /mnt/SDCARD/App/PackageManager/
           if [ $reset_configs -eq 1 ]; then
               packageManager --confirm
           else
               packageManager --confirm --auto_update
           fi
           free_mma
   
           cd $sysdir
           # ./config/boot_mod.sh # disabled because of possible incompatibility with new firmware
   
           # Show installation complete
           rm -f .installed
       fi
   
       #########################################################################################
       #                                  Wizards are completed                                #
       #                                           Exit                                        #
       #########################################################################################
   
       if [ $DEVICE_ID -eq MODEL_MM ]; then
           echo "$verb2 complete!" >> /tmp/.update_msg
           touch $sysdir/.waitConfirm
           touch $sysdir/.installed
           sync
       else
           echo "$verb2 complete - Rebooting..." >> /tmp/.update_msg
       fi
   
       installUI &
       sleep 1
   
       if [ $DEVICE_ID -eq MODEL_MM ]; then
           counter=10
   
           while [ -f $sysdir/.waitConfirm ] && [ $counter -ge 0 ]; do
               echo "Press A to turn off (""$counter""s)" >> /tmp/.update_msg
               counter=$((counter - 1))
               sleep 1
           done
   
           killall installUI
           bootScreen "End"
       else
           touch $sysdir/.installed
       fi
   
       rm -f $sysdir/config/currentSlide 2> /dev/null
       sync
   }
   
   install_core() {
       echo ":: Install Onion"
       msg="$1"
   
       if [ ! -f "$CORE_PACKAGE_FILE" ]; then
           echo "CORE FILE MISSING"
           return
       fi
   
       rm -f \
           $sysdir/updater \
           $sysdir/bin/batmon \
           $sysdir/bin/prompt \
           /mnt/SDCARD/miyoo/app/.isExpert
   
       # Remove old lang files
       rm -rf /mnt/SDCARD/miyoo/app/lang_backup 2> /dev/null
   
       # Onion core installation / update.
       cd /
       unzip_progress "$CORE_PACKAGE_FILE" "$msg" /mnt/SDCARD
   
       # Cleanup
       rm -f $CORE_PACKAGE_FILE
   }
   
   install_retroarch() {
       echo ":: Install RetroArch"
       msg="$1"
   
       # Check if RetroArch zip also exists
       if [ ! -f "$RA_PACKAGE_FILE" ]; then
           return
       fi
   
       # Backup old RA configuration
       cd /mnt/SDCARD/RetroArch
       mkdir -p /mnt/SDCARD/Backup
       mv .retroarch/retroarch.cfg /mnt/SDCARD/Backup/
   
       # Remove old RetroArch before unzipping
       maybe_remove_retroarch
   
       # Install RetroArch
       cd /
       unzip_progress "$RA_PACKAGE_FILE" "$msg" /mnt/SDCARD
   
       # Cleanup
       rm -f $RA_PACKAGE_FILE
       rm -f $RA_PACKAGE_VERSION_FILE
   }
   
   maybe_remove_retroarch() {
       if [ -f $RA_PACKAGE_FILE ]; then
           cd /mnt/SDCARD/RetroArch
   
           tempdir=/mnt/SDCARD/.temp
           mkdir -p $tempdir
   
           if [ -d .retroarch/cheats ]; then
               mv .retroarch/cheats $tempdir/
           fi
           if [ -d .retroarch/overlay ]; then
               mv .retroarch/overlay $tempdir/
           fi
           if [ -d .retroarch/filters ]; then
               mv .retroarch/filters $tempdir/
           fi
           if [ -d .retroarch/thumbnails ]; then
               mv .retroarch/thumbnails $tempdir/
           fi
   
           remove_everything_except $(basename $RA_PACKAGE_FILE)
   
           mkdir -p .retroarch
           mv $tempdir/* .retroarch/
           rm -rf $tempdir
       fi
   }
   
   restore_ra_config() {
       echo ":: Restore RA config"
       cfg_file=/mnt/SDCARD/Backup/retroarch.cfg
       if [ -f $cfg_file ]; then
           mv -f $cfg_file /mnt/SDCARD/RetroArch/.retroarch/
       fi
   }
   
   install_configs() {
       reset_configs=$1
       zipfile=$sysdir/config/configs.pak
   
       echo ":: Install configs (reset: $reset_configs)"
   
       if [ ! -f $zipfile ]; then
           return
       fi
   
       cd /mnt/SDCARD
       if [ $reset_configs -eq 1 ]; then
           # Overwrite all default configs
           rm -rf /mnt/SDCARD/Saves/CurrentProfile/config/*
           7z x -aoa $zipfile # aoa = overwrite existing files
       else
           # Extract config files without overwriting any existing files
           7z x -aos $zipfile # aos = skip existing files
       fi
   
       # Set X and Y button keymaps if empty
       cat /mnt/SDCARD/.tmp_update/config/keymap.json |
           sed 's/"mainui_button_x"\s*:\s*""/"mainui_button_x": "app:Search"/g' |
           sed 's/"mainui_button_y"\s*:\s*""/"mainui_button_y": "glo"/g' \
               > ./temp_keymap.json
       mv -f ./temp_keymap.json /mnt/SDCARD/.tmp_update/config/keymap.json
   }
   ## -- @Pierce:  Idk why it would remove the entire CWD and restart without that firmware file.
   ## Maybe things break on miyoo if their missing this.
   ## This should be symlinked to `/mnt/SDCARD/.tmp_update/lib` but I'm not seeing this library there
   ## either, it's used everywhere around the OS so I can't really afford to not have it.

       echo ":: Check firmware"
       if [ ! -f /customer/lib/libpadsp.so ]; then
           cd $sysdir
           infoPanel -i "res/firmware.png"
           rm -rf $sysdir
           reboot
           sleep 10
           exit 0
       fi
   }
   
   backup_system() {
       echo ":: Backup system"
       old_ra_dir=/mnt/SDCARD/RetroArch/.retroarch
   
       # Move BIOS files from stock location
       if [ -d $old_ra_dir/system ]; then
           mkdir -p /mnt/SDCARD/BIOS
           mv -f $old_ra_dir/system/* /mnt/SDCARD/BIOS/
       fi
   
       # Backup old saves
       if [ -d $old_ra_dir/saves ]; then
           mkdir -p /mnt/SDCARD/Backup/saves
           mv -f $old_ra_dir/saves/* /mnt/SDCARD/Backup/saves/
       fi
   
       # Backup old states
       if [ -d $old_ra_dir/states ]; then
           mkdir -p /mnt/SDCARD/Backup/states
           mv -f $old_ra_dir/states/* /mnt/SDCARD/Backup/states/
       fi
   
       # Imgs
       if [ -d /mnt/SDCARD/Imgs ]; then
           mv -f /mnt/SDCARD/Imgs /mnt/SDCARD/Backup/Imgs
       fi
   }
   
   remove_old_search() {
       echo ":: Remove old search"
   
       rm -rf /mnt/SDCARD/Emu/SEARCH 2> /dev/null
       rm -f /mnt/SDCARD/App/Search/data/data_cache6.db 2> /dev/null
   }
   
   run_migration_scripts() {
       local MIGRATION_DIR=$sysdir/script/migration
       local MIGRATION_STATE_FILE=$sysdir/config/migration_state
       local MIGRATION_LIST_FILE=/tmp/active_migrations
       local count=0
       local migration_state=$([ -f "$MIGRATION_STATE_FILE" ] 2> /dev/null && cat "$MIGRATION_STATE_FILE" || echo 0)
   
       [ -n "$migration_state" ] && [ "$migration_state" -eq "$migration_state" ] 2> /dev/null
       if [ $? -ne 0 ]; then
           migration_state=0
       fi
   
       if [ -d "$MIGRATION_DIR" ]; then
           for entry in "$MIGRATION_DIR"/*.sh; do
               if [ ! -f "$entry" ]; then
                   continue
               fi
   
               migration_id=$(basename "$entry" | cut -d'_' -f1)
   
               # Validate migration ID
               [ -n "$migration_id" ] && [ "$migration_id" -eq "$migration_id" ] 2> /dev/null
               if [ $? -ne 0 ]; then
                   continue
               fi
   
               if [ $migration_state -ge $migration_id ]; then
                   continue
               fi
   
               echo "$entry" >> "$MIGRATION_LIST_FILE"
               count=$((count + 1))
           done
       fi
   
       if [ $count -gt 0 ] && [ -f "$MIGRATION_LIST_FILE" ]; then
           sort -f -o temp "$MIGRATION_LIST_FILE"
           rm -f "$MIGRATION_LIST_FILE"
           mv temp "$MIGRATION_LIST_FILE"
   
           echo "0/$count: Running migrations... 0%" >> /tmp/.update_msg
           local n=0
           local max_id=0
   
           while read entry; do
               n=$((n + 1))
               echo "Running migration script ($n/$count):" $(basename "$entry")
               migration_id=$(basename "$entry" | cut -d'_' -f1)
   
               cd $sysdir
               chmod a+x "$entry"
               eval "$entry"
   
               local percent=$((n * 100 / count))
               echo "$n/$count: Running migrations... $percent%" >> /tmp/.update_msg
   
               if [ $migration_id -gt $max_id ]; then
                   max_id=$migration_id
                   echo "$migration_id" > "$MIGRATION_STATE_FILE"
                   sync
               fi
   
               sleep 0.1
           done < "$MIGRATION_LIST_FILE"
   
           rm $MIGRATION_LIST_FILE
       fi
   }
   
   refresh_roms() {
       echo ":: Refresh roms"
       # Force refresh the rom lists
       if [ -d /mnt/SDCARD/Roms ]; then
           cd /mnt/SDCARD/Roms
           find . -type f -name "*_cache[0-9].db" -exec rm -f {} \;
       fi
   }
   
   remove_everything_except() {
       find * .* -maxdepth 0 -not -name "$1" -exec rm -rf {} \;
   }
   
   md5hash() {
       echo $(md5sum "$1" | awk '{ print $1; }')
   }
   
   zip_total() {
       zipfile="$1"
       total=$(unzip -l "$zipfile" | tail -1 | grep -Eo "([0-9]+) files" | sed "s/[^0-9]*//g")
       echo $total
   }
   
   unzip_progress() {
       zipfile="$1"
       msg="$2"
       dest="$3"
   
       echo "   - Extract '$zipfile' into $dest"
   
       verify_file
   
       echo "Verifying package..." > /tmp/.update_msg
       sleep 3
   
       if [ -f "/mnt/SDCARD/.tmp_update/onion.pak" ]; then
           echo "onion.pak exists"
           sleep 1
       else
           echo "onion.pak is missing, extraction will fail"
       fi
   
       verify_file
   
       sleep 1
   
       # Run the 7z extraction command in the background and redirect output to /tmp/.extraction_output
       (
           7z x -aoa -o"$dest" "$zipfile" -bsp1 -bb > /tmp/.extraction_output
           echo $? > "/tmp/extraction_status"
       ) &
   
       sleep 1
   
       if pgrep 7z > /dev/null; then
           echo "7Z is running"
           sleep 1
       else
           echo "7Z is NOT running and should be"
           sleep 1
       fi
   
       if [ -f "/mnt/SDCARD/.tmp_update/onion.pak" ]; then
           echo "onion.pak still exists"
           sleep 1
       else
           echo "onion.pak is still missing, extraction has probably failed"
           sleep 1
           echo "the build will say it's complete but isn't"
       fi
   
       # Continuously update /tmp/.update_msg every 500 milliseconds until the command line finishes
       a=$(pgrep 7z)
       while [ -n "$a" ]; do
           last_line=$(tail -n 1 /tmp/.extraction_output)
           value=$(echo "$last_line" | sed 's/.* \([0-9]\+\)%.*/\1/')
           if [ "$value" -eq "$value" ] 2> /dev/null; then # check if the value is numeric
               if [ $value -eq 0 ]; then
                   echo "Preparing file system..." > /tmp/.update_msg # It gets stuck a bit at 0%, so don't show   percentage yet
               else
                   echo "$msg $value%" > /tmp/.update_msg # Now we can show completion percentage
               fi
           fi
           > /tmp/.extraction_output # to avoid to parse a too big file
           sleep 0.5
           a=$(pgrep 7z)
       done
   
       # Check the exit status of the extraction command
       extraction_status=$(cat "/tmp/extraction_status")
       if [ "$extraction_status" -ne 0 ]; then
           touch $sysdir/.installFailed
           echo ":: Installation failed!"
           sync
           reboot
           sleep 10
           exit 0
       else
           echo "$msg 100%" >> /tmp/.update_msg
       fi
   
       rm /tmp/extraction_status
       rm /tmp/.extraction_output
   }
   
   free_mma() {
       /mnt/SDCARD/.tmp_update/bin/freemma
   }
   
   main
   sync
   reboot
   sleep 10
   ```

    First pass analysis: 
      - This mf big...
      - Some conditional logic based on wether the device is MM or MM+, we want to follow the MM+ path as that has wifi. 
      - We probably want to copy `MINUI/trimui/app/.tmp_update/tg3040.sh and set the CPU to max speed with 
      ```sh
      echo userspace > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
      CPU_PATH=/sys/devices/system/cpu/cpu0/cpufreq/scaling_setspeed
      CPU_SPEED_PERF=2000000
      echo $CPU_SPEED_PERF > $CPU_PATH
      ```
      - I definitely need to override `check_device_model`, `check_firmware` (maybe not with the customer/lib swap).
      - Check firmware simply checks for the existance of `libpadsp.so` and aborts the install/deletes the install folder if its not found. This library is used all throughout the codebase but isn't included in Onion so maybe there was a miyoo mini update that included it and Onion doesn't want to support firmware before that because it breaks audio throughout? 

      The library is a PulseAudio sound server that provides a compatibility layer that allows applications designed to use OSS (Open Sound System) to work with PulseAudio. I'm not sure that the brick needs that though. 

      I either need to remove all references to `libpadsp.so` nothing (ewww) or find a way to include it with the firware.
      I'm going to install it on my docker container and pull the lib file into my install.

      **update**: It looks like `libpadsp.so` is in the repo to build Onion -> `Onion/static/build/miyoo/lib/libpadsp.so`,  I can try just copying that file. 

      Either way, the entire system expects the `LD_PRELOAD` env variable to be set with `libpadsp.so` 
      
      - All instances of `/customer/lib/` should probably be updated to `/usr/trimui/bin`

      - Looks like `customer/app` is used quite a bit throughout the code, this will need to be updated but I don't know if all apps that the shells scripts and c programs expect exist on the brick, it'll probably require replacements across the board.

