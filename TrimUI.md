# Below are my findings from digging arounnd the TrimUI brick filesystem.
An application to track your time played on trimUI devices

## Dev notes


## Boot script breakdown 

## Core TRIMUI functionality

### Led API
Location: `/sys/led_anim`

Used in: 
   - `/etc/led_controller/led_controller_low_battery_led.sh` < -- my script -->
   - `/usr/trimui/apps/fn_editor/com.trimui.switch.cpufreq.sh`
   - `/usr/trimui/apps/fn_editor/com.trimui.toggle.dpad_joystick.sh`
   - `usr/trimui/bin/init_leds.sh`
   - `usr/trimui/bin/runtrimui.sh`
   - `/usr/trimui/bin/low_battery_led.sh`
   
### Input Daemon
Location: `/etc/trimui_inputd`
Used in: 
   Used in:
   - `/usr/trimui/apps/fn_editor/com.trimui.toggle.dpad_joystick.sh` 
   - `/usr/trimui/apps/fn_editor/com.trimui.dpad_joystick.sh`
   - `/usr/trimui/apps/fn_editor/com.trimui.dpad.sh`
   - `/usr/trimui/apps/fn_editor/com.trimui.joystick.sh`
   - `/usr/trimui/apps/fn_editor/com.trimui.toggle.dpad_joystick.sh`
   - `/usr/trimui/apps/fn_editor/com.trimui.turbo_a.sh`
   - `/usr/trimui/apps/fn_editor/com.trimui.turbo_b.sh`
   - `/usr/trimui/apps/fn_editor/com.trimui.turbo_l.sh`
   - `/usr/trimui/apps/fn_editor/com.trimui.turbo_l2.sh`
   - `/usr/trimui/apps/fn_editor/com.trimui.turbo_r2.sh`
   - `/usr/trimui/apps/fn_editor/com.trimui.turbo_r.sh`
   - `/usr/trimui/apps/fn_editor/com.trimui.turbo_x.sh`
   - `/usr/trimui/apps/fn_editor/com.trimui.turbo_y.sh`
   - `/usr/trimui/bin/premainui.sh`
   - `/usr/trimui/bin/runtrimui.sh`
   - ``

While running applications, trimui mounts the io device streams as follows

From reading `/rom/var/.lastlog` of my applications - whenrunning applications, trimui mounts the io device streams as follows:

 `user.notice procd: trimui_sunxi_gpio_init: ver Oct 31 2024`
### Audio Devices
- `/dev/audio`: Audio device
- `/dev/dsp`: Digital Signal Processor device
- `/dev/mixer`: Audio mixer device
- `/dev/snd/pcmC0D0c`: ALSA PCM capture device
- `/dev/snd/pcmC0D0p`: ALSA PCM playback device
- `/dev/snd/controlC0`: ALSA control device

### MIDI/Sequencer Devices
- `/dev/snd/seq`: ALSA sequencer device
- `/dev/sequencer`: MIDI sequencer device
- `/dev/sequencer2`: MIDI sequencer device
- `/dev/snd/timer`: ALSA timer device

### Input Devices
- `/dev/input/event0`: Input event device
- `/dev/input/event1`: Input event device
- `/dev/input/event2`: Input event device (e.g., keyboard, mouse)
- `/dev/input/event3`: Input event device
- `/dev/input/js0`: Joystick device

The log entries you provided indicate that the `procd` process is logging events related to device additions detected by the `udev` system. `udev` is the device manager for the Linux kernel, responsible for managing device nodes in the dev directory.

This means that while we can get normal xbox input from the device controller through SDL [see SDL base](https://github.com/pierceTee/TrimuiLEDController/blob/master/workspace/src/sdl_base.c#L43) all input events are streamed to `/dev/input/event3`. Similarly we can see the events from all the other IO.
 
 Jose Gonzalez loops through the event files in this directory and pulls the specific codes, then converts them to input events in his [launch.sh scripts](https://github.com/josegonzalez/trimui-brick-filebrowser-pak/blob/main/launch.sh#L74)


### Bluetooth Functions
Looks like they're using [BlueZ-5.54](https://www.linuxfromscratch.org/blfs/view/10.0-systemd/general/bluez.html) for their bluetooth functionality. 
Here's more resources on how to use it  [BlueZ.org](https://www.bluez.org/), [Archlinux Guide](https://wiki.archlinux.org/title/Bluetooth)


I found this out from the following digging...

`strings usr/trimui/bin/trimui_btmanager` <-- runs during their bt config app -->
```sh
killall -9 bluetoothctl
killall -9 script
bluetoothctl devices
bluetooth
/tmp/bluetooth_request
killall -9 bluetoothctl
/usr/bin/bluetoothctl pair %s
/usr/bin/bluetoothctl trust %s
/usr/bin/bluetoothctl connect %s
/usr/bin/bluetoothctl info %s
/tmp/bluetooth_result
/usr/trimui/bluetooth_devices
/tmp/bluetooth_ready
/usr/bin/bluetoothctl agent on
/usr/bin/bluetoothctl default-agent
/usr/bin/bluetoothctl power on
script -q -c '/usr/bin/bluetoothctl scan on' /dev/null < /dev/null
/usr/bin/bluetoothctl remove %s
/usr/bin/bluetoothctl disconnect %s
```

`strings usr/trimui/bin/hardwareservice`
```sh
wpa_supplicant
/etc/bluetooth/bluetoothd start
/usr/bin/bluetoothctl power on
/usr/trimui/bin/trimui_btmanager &
turn on bt
bluetooth
rm /tmp/bluetooth_ready
bluetoothd
/etc/bluetooth/bluetoothd stop
killall -9 bluetoothctl
/tmp/bluetooth_turn_on
/tmp/bluetooth_turn_off
/etc/bluetooth/bluetoothd start
/usr/bin/bluetoothctl power on
```
Findings: 

we can run ` /etc/bluetooth/bluetoothd start` to start the bluetooth daemon. Then `./bluetoothctl help` to get the list of commands
```
root@TinaLinux:/usr/bin# ./bluetoothctl help
Menu main:
Available commands:
-------------------
advertise                                         Advertise Options Submenu
scan                                              Scan Options Submenu
gatt                                              Generic Attribute Submenu
list                                              List available controllers
show ÄctrlÅ                                       Controller information
select <ctrl>                                     Select default controller
devices                                           List available devices
paired-devices                                    List paired devices
system-alias <name>                               Set controller alias
reset-alias                                       Reset controller alias
power <on/off>                                    Set controller power
pairable <on/off>                                 Set controller pairable mode
discoverable <on/off>                             Set controller discoverable mode
discoverable-timeout ÄvalueÅ                      Set discoverable timeout
agent <on/off/capability>                         Enable/disable agent with given capability
default-agent                                     Set agent as the default one
advertise <on/off/type>                           Enable/disable advertising with given type
set-alias <alias>                                 Set device alias
scan <on/off>                                     Scan for devices
info ÄdevÅ                                        Device information
pair ÄdevÅ                                        Pair with device
trust ÄdevÅ                                       Trust device
untrust ÄdevÅ                                     Untrust device
block ÄdevÅ                                       Block device
unblock ÄdevÅ                                     Unblock device
remove <dev>                                      Remove device
connect <dev>                                     Connect device
disconnect ÄdevÅ                                  Disconnect device
menu <name>                                       Select submenu
version                                           Display version
quit                                              Quit program
exit                                              Quit program
help                                              Display help about this program
export                                            Print environment variables

```



### Universally Available Compiled C Programs

The following compiled C programs are available in the `usr/trimui/bin` directory. Their usage can be found in other scripts in the same directory along with MinUI files:

- `MainUI`
- `bgcmdhandler`
- `cgdisk`
- `fbscreencap`
- `fixparts`
- `gdisk`
- `hardwareservice`
- `keymon`
- `sdl2play`
- `sdldisplay/sdldisplay`
- `sdlteartest`
- `sdltest`
- `sgdisk`
- `shmvar`
- `systemval`
- `testbitmap`
- `testsprite`
- `trimui_btmanager`
- `trimui_inputd`
- `trimui_scened`
- `udpbcast`

## TrimUI OS Shell Script analysis
### 1-15-25
*Found in `usr/trimui/bin`*

### /usr/trimui/bin/init_lang.sh
* This script sets the system language based on the presence of a user guide.
* If a Chinese guide exists, it sets the system to Chinese.

```sh
#!/bin/sh

sleep 2

if [ -d /mnt/SDCARD/Apps/user_guide_cn ]; then
  echo "set language chinese"

  if [ -e /mnt/UDISK/system.json ]; then
     if [ -e /mnt/UDISK/.inited.lang ]; then
         echo "skip copy"
     else
         touch /mnt/UDISK/.inited.lang
         echo "copy system.json from factory reset"
         cp /usr/trimui/bin/system.json.ch.lang /mnt/UDISK/system.json
         killall -9 MainUI
     fi
  else
     touch /mnt/UDISK/.inited.lang
     echo "copy system.json"
     cp /usr/trimui/bin/system.json.ch.lang /mnt/UDISK/system.json
     killall -9 MainUI
  fi
else
  echo "skip language set"
fi
```
----------------------------------------
### /usr/trimui/bin/init_leds.sh (my version)
* This script configures LED settings based on a configuration file.
* It reads settings such as brightness, color, duration, and effect for each LED and applies them.
```sh
#!/bin/sh

export INSTALL_DIR="/etc/led_controller"
export SETTINGS_FILE="$INSTALL_DIR/settings.ini"
export SYS_FILE_PATH="/sys/class/led_anim"
export SERVICE_PATH="/etc/led_controller"
export BASE_LED_PATH="/sys/class/led_anim"
export SCRIPT_NAME=$(basename "$0")
export LOG_FILE="$INSTALL_DIR/settings_daemon.log"

# Function to apply settings for a specific LED
apply_led_settings() {
    local led=$1
    local brightness=$2
    local color=$3
    local duration=$4
    local effect=$5

    echo "[$SCRIPT_NAME]: Writing $led LED information to configuration files ..." | tee -a "$LOG_FILE"
    if [ "$led" = "f1f2" ]; then
        echo $brightness > "$SYS_FILE_PATH/max_scale_$led"
        echo $color > "$SYS_FILE_PATH/effect_rgb_hex_f1"
        echo $color > "$SYS_FILE_PATH/effect_rgb_hex_f2"
        echo $duration > "$SYS_FILE_PATH/effect_duration_f1"
        echo $duration > "$SYS_FILE_PATH/effect_duration_f2"
        echo $effect > "$SYS_FILE_PATH/effect_f1"
        echo $effect > "$SYS_FILE_PATH/effect_f2"
    else
        echo $brightness > "$SYS_FILE_PATH/max_scale_$led"
        echo $color > "$SYS_FILE_PATH/effect_rgb_hex_$led"
        echo $duration > "$SYS_FILE_PATH/effect_duration_$led"
        echo $effect > "$SYS_FILE_PATH/effect_$led"
    fi
}

# == Start of the script ==
echo "[$SCRIPT_NAME]: LED settings daemon started ..." | tee -a "$LOG_FILE"

# Enable write permissions on LED files
chmod a+w $BASE_LED_PATH/* | tee -a "$LOG_FILE"

echo "[$SCRIPT_NAME]: Writing LED information to configuration files ..." | tee -a "$LOG_FILE"
# Read settings from the settings file and apply 
while IFS= read -r line; do
    if echo "$line" | grep -qE '^\[([a-zA-Z0-9]+)\]$'; then
        led=$(echo "$line" | sed -n 's/^\[\([a-zA-Z0-9]\+\)\]$/\1/p')
    elif echo "$line" | grep -qE '^brightness=([0-9]+)$'; then
        brightness=$(echo "$line" | sed -n 's/^brightness=\([0-9]\+\)$/\1/p')
    elif echo "$line" | grep -qE '^color=0x([0-9A-Fa-f]+)$'; then
        color=$(echo "$line" | sed -n 's/^color=0x\([0-9A-Fa-f]\+\)$/\1/p')
    elif echo "$line" | grep -qE '^duration=([0-9]+)$'; then
        duration=$(echo "$line" | sed -n 's/^duration=\([0-9]\+\)$/\1/p')
    elif echo "$line" | grep -qE '^effect=([0-9]+)$'; then
        effect=$(echo "$line" | sed -n 's/^effect=\([0-9]\+\)$/\1/p')
        apply_led_settings $led $brightness $color $duration $effect
    fi
done < "$SETTINGS_FILE"

# Disable write permissions on LED files
chmod a-w $BASE_LED_PATH/* | tee -a "$LOG_FILE"

echo "[$SCRIPT_NAME]: Finished writing LED information to configuration files ..." | tee -a "$LOG_FILE"
echo "[$SCRIPT_NAME]: LED settings daemon stopped ..." | tee -a "$LOG_FILE"
echo "[$SCRIPT_NAME]: exiting ..." | tee -a "$LOG_FILE"
```
----------------------------------------
### /usr/trimui/bin/init_leds_original.sh (stock OS version)
* This script sets the LED frame color to white and writes it to the LED animation system.
* Parameters:
  - $1: Value to set for the LED (not used in the script)
```sh
#exit 0
value=$1
#valstr=`printf "%02X%02X%02X" $value $value $value`
r=255
g=255
b=255
valstr=`printf "%02X%02X%02X" $r $g $b`
echo "$valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr $valstr "\
      "$valstr $valstr $valstr $valstr $valstr $valstr $valstr" > /sys/class/led_anim/frame_hex

#echo 5 > /sys/class/leds/sunxi_led0r/brightness
#usleep 10000
#echo 5 > /sys/class/leds/sunxi_led0g/brightness
#usleep 10000
#echo 5 > /sys/class/leds/sunxi_led0b/brightness
#usleep 10000

#echo $value > /sys/class/led_anim/max_scale
```
----------------------------------------
### /usr/trimui/bin/kill_apps.sh
* This script kills a process and all its child processes.
* Parameters:
  - $1: PID of the process to kill
```sh
#!/bin/sh

#pid=`ps | grep cmd_to_run | grep -v grep | sed 's/[ ]\+/ /g' | cut -d' ' -f1`
pid=$1

ppid=$pid
echo pid is $pid
while [ "" != "$pid" ]; do
   ppid=$pid
   pid=`pgrep -P $ppid`
done

if [ "" != "$ppid" ]; then
   kill -9 $ppid
fi
echo ppid $ppid quit
```
----------------------------------------
### /usr/trimui/bin/list_script.sh (my script used to compile this list)
* This script lists the contents of all .sh files in a specified directory.
* Parameters:
  - $1: Directory to list files from (default is current directory)
```sh
#!/bin/sh
#Directory to list files from
DIR=${1:-.}

# List all .sh files in the directory
for file in "$DIR"/*.sh; do
    if [ -f "$file" ]; then
        echo "Contents of $file:"
        cat "$file"
        echo "----------------------------------------"
    fi
done
```
----------------------------------------

### /usr/trimui/bin/low_battery_led.sh (Stock fimware)
* This script sets the LED animation for low battery indication.
* Parameters:
* Particularly it sets the back LEDs to blink red.
  - $1: 1 to enter low battery mode, 0 to exit low battery mode
```sh
case "$1" in
1 )
        echo "enter low battery"
        echo "FF0000 " >  /sys/class/led_anim/effect_rgb_hex_lr
        echo "30000" >  /sys/class/led_anim/effect_cycles_lr
        echo "2000" >  /sys/class/led_anim/effect_duration_lr
        # echo "3" >  /sys/class/led_anim/effect_lr
        echo "6" >  /sys/class/led_anim/effect_lr
        ;;
0 )
        echo "exit low battery"
        echo "0" >  /sys/class/led_anim/effect_lr
        # echo "00FFFF " >  /sys/class/led_anim/effect_rgb_hex_lr
        # echo "3" >  /sys/class/led_anim/effect_lr
        # echo "30000" >  /sys/class/led_anim/effect_cycles_lr
        # echo "500" >  /sys/class/led_anim/effect_duration_lr
        # echo "3" >  /sys/class/led_anim/effect_lr
        ;;
*)
        ;;
esac
```


### /usr/trimui/bin/low_battery_led.sh (Firmware  1.0.6 )
* This script sets the LED animation for low battery indication.
* Particularly it sets the front and back LEDs to blink red.
* Parameters:
  - $1: 1 to enter low battery mode, 0 to exit low battery mode
```sh
case "$1" in
1 )
        echo "enter low battery"
        echo "FF0000 " >  /sys/class/led_anim/effect_rgb_hex_lr
        echo "30000" >  /sys/class/led_anim/effect_cycles_lr
        echo "2000" >  /sys/class/led_anim/effect_duration_lr
        # echo "3" >  /sys/class/led_anim/effect_lr
         echo "6" >  /sys/class/led_anim/effect_lr

        echo "FF0000 " >  /sys/class/led_anim/effect_rgb_hex_f1
        echo "30000" >  /sys/class/led_anim/effect_cycles_f1
        echo "2000" >  /sys/class/led_anim/effect_duration_f1
        echo "6" >  /sys/class/led_anim/effect_f1

        echo "FF0000 " >  /sys/class/led_anim/effect_rgb_hex_f2
        echo "30000" >  /sys/class/led_anim/effect_cycles_f2
        echo "2000" >  /sys/class/led_anim/effect_duration_f2
        echo "6" >  /sys/class/led_anim/effect_f2
        ;;
0 )
        echo "exit low battery"
        echo "0" >  /sys/class/led_anim/effect_lr
        echo "0" >  /sys/class/led_anim/effect_f1
        echo "0" >  /sys/class/led_anim/effect_f2
        # echo "00FFFF " >  /sys/class/led_anim/effect_rgb_hex_lr
        # echo "3" >  /sys/class/led_anim/effect_lr
        # echo "30000" >  /sys/class/led_anim/effect_cycles_lr
        # echo "500" >  /sys/class/led_anim/effect_duration_lr
        # echo "3" >  /sys/class/led_anim/effect_lr
        ;;
*)
        ;;
esac

```
----------------------------------------
### /usr/trimui/bin/preload.sh
* This script sets the CPU frequency scaling governor and min/max frequencies.

```sh
#!/bin/sh

echo ondemand > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
echo 408000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq
echo 2000000 > /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
```
----------------------------------------
### /usr/trimui/bin/premainui.sh
* This script removes temporary input configuration files.
```sh
#!/bin/sh

rm -f /tmp/trimui_inputd/input_no_dpad
rm -f /tmp/trimui_inputd/input_dpad_to_joystick
```
----------------------------------------
### /usr/trimui/bin/runtrimui-original.sh (pre MinUI startup script)
* This script initializes various system settings and runs the main UI application.
* It also handles updating the system if an update file is present.
```sh
#!/bin/sh

#echo 0 > /sys/class/graphics/fbcon/cursor_blink
#echo 0 > /sys/class/vtconsole/vtcon1/bind

export PATH=/usr/trimui/bin:$PATH
chmod a+x /usr/bin/notify
UPDATE_FILE_NAME=/tmp/.try_upgrade_file
UPDATE_DEST=/mnt/UDISK/trimui.img
UPDATE_LOG=/mnt/SDCARD/update.log
UDISK_TRIMUI_DIR=/mnt/UDISK/trimui

SDCARD_TRIMUI_DIR=/mnt/SDCARD/trimui

SDCARD_START_SCRIPTS_DIR=/mnt/SDCARD/System/starts

INPUTD_SETTING_DIR_NAME=/tmp/trimui_inputd

#PD11 pull high for VCC-5v
echo 107 > /sys/class/gpio/export
echo -n out > /sys/class/gpio/gpio107/direction
echo -n 1 > /sys/class/gpio/gpio107/value

#rumble motor PH3
echo 227 > /sys/class/gpio/export
echo -n out > /sys/class/gpio/gpio227/direction
echo -n 0 > /sys/class/gpio/gpio227/value

##//TRIMUI Smart Pro IO board power
#Left/Right Pad PD14/PD18
# echo 110 > /sys/class/gpio/export
# echo -n out > /sys/class/gpio/gpio110/direction
# echo -n 1 > /sys/class/gpio/gpio110/value

# echo 114 > /sys/class/gpio/export
# echo -n out > /sys/class/gpio/gpio114/direction
# echo -n 1 > /sys/class/gpio/gpio114/value

#DIP Switch PH19
echo 243 > /sys/class/gpio/export
echo -n in > /sys/class/gpio/gpio243/direction

killprocess(){
  #pid=`ps | grep $1 | grep -v grep | cut -d' ' -f3`
  pid=`pgrep $1`
  while [ "$pid" != "" ] ; do
    echo kill -9 $pid
    kill -9 $pid
    sleep 1
    pid=`pgrep $1`
  done
}

runifnecessary(){
   #a=`ps | grep $1 | grep -v grep`
   a=`pgrep $1`
   if [ "$a" == "" ] ; then
      $2 &
   fi
}

#wait for SDCARD mounted
mounted=`cat /proc/mounts | grep SDCARD`
cnt=0
while [ "$mounted" == "" ] && [ $cnt -lt 6 ] ; do
  sleep 0.5
  cnt=`expr $cnt + 1`
  mounted=`cat /proc/mounts | grep SDCARD`
done

if [ -d $SDCARD_START_SCRIPTS_DIR ] ; then
  for f in `ls $SDCARD_START_SCRIPTS_DIR/*.sh` ; do
        chmod a+x $f
        $f
  done
fi

mkdir $INPUTD_SETTING_DIR_NAME

echo 1 > /sys/class/led_anim/effect_enable
echo "FFFFFF" > /sys/class/led_anim/effect_rgb_hex_lr
echo 1 > /sys/class/led_anim/effect_cycles_lr
echo 1000 > /sys/class/led_anim/effect_duration_lr
echo 1 >  /sys/class/led_anim/effect_lr

syslogd -S

/etc/bluetooth/bluetoothd start
#/usr/bin/bluealsa -p a2dp-source&
#touch /tmp/bluetooth_ready

usbmode=`/usr/trimui/bin/systemval usbmode`

if [ "$usbmode" == "dock" ] ; then
   /usr/trimui/bin/usb_dock.sh
elif [ "$usbmode" == "host" ] ; then
   /usr/trimui/bin/usb_host.sh
else
   /usr/trimui/bin/usb_device.sh
fi

wifion=`/usr/trimui/bin/systemval wifi`
if [ "$wifion" != "1" ] ; then
      ifconfig wlan0 down
      killall -15 wpa_supplicant
      killall -9 udhcpc
fi

while [ 1 ]; do
  if [ -f $UPDATE_FILE_NAME ] ; then
    upfile=`cat $UPDATE_FILE_NAME`
    if [ -f ${upfile} ] ; then
        killprocess keymon
        cd /
        umount  "$UDISK_TRIMUI_DIR"
        sleep 1
        mounted=`mount | grep "$UDISK_TRIMUI_DIR"`
        echo mounted is $mounted

        cat /proc/mounts
        echo start updating $upfile $? | tee $UPDATE_LOG
        updateui >> $UPDATE_LOG &
        sleep 1
        notify 0 extracting package
        cp $upfile $UPDATE_DEST
        notify 100 quit
        sleep 1
        killprocess updateui
       else
          echo $upfile not found | tee $UPDATE_LOG
       fi
    rm -f $UPDATE_FILE_NAME
    rm -f /tmp/state.json
  else
    #exit 0
    udiskOK=0
    uiRunned=0
    ls /dev/mmcblk1* > /tmp/imgrun.log
    cat /proc/mounts >> /tmp/imgrun.log
    if [ -d $SDCARD_TRIMUI_DIR ] ; then
       echo "run from sdcard" >> /tmp/imgrun.log
       export LD_LIBRARY_PATH=${SDCARD_TRIMUI_DIR}/lib
       runifnecessary "keymon" ${SDCARD_TRIMUI_DIR}/app/keymon
       runifnecessary "inputd" ${SDCARD_TRIMUI_DIR}/app/trimui_inputd
       runifnecessary "scened" ${SDCARD_TRIMUI_DIR}/app/trimui_scened
       runifnecessary "trimui_btmanager" ${SDCARD_TRIMUI_DIR}/app/trimui_btmanager
       runifnecessary "hardwareservice" ${SDCARD_TRIMUI_DIR}/app/hardwareservice
       cd ${SDCARD_TRIMUI_DIR}/app/

       #absolute path for ps
       ${SDCARD_TRIMUI_DIR}/app/premainui.sh
       ${SDCARD_TRIMUI_DIR}/app/MainUI
       ${SDCARD_TRIMUI_DIR}/app/preload.sh

       if [ $? -eq 0 ] || [ $? -eq 137 ] || [ $? -eq 143 ] ; then
          uiRunned=1
       else
          uiRunned=0
       fi
    else
       echo "no $SDCARD_TRIMUI_DIR" >> /tmp/imgrun.log
    fi

    if [ $uiRunned -eq 0 ] ; then
       if [ -f $UPDATE_DEST ] ; then
          mkdir -p $UDISK_TRIMUI_DIR
          mounted=`mount | grep $UDISK_TRIMUI_DIR`
          if [ "$mounted" == "" ] ; then
            mount -o loop $UPDATE_DEST $UDISK_TRIMUI_DIR
            if [ $? -eq 0 ] ; then
               echo "new img mounted" >> /tmp/imgrun.log
               udiskOK=1
             fi
          else
            echo "img mounted" >> /tmp/imgrun.log
               udiskOK=1
          fi
       else
          mkdir -p $UDISK_TRIMUI_DIR
          echo "$UPDATE_DEST exists" >> /tmp/imgrun.log
       fi
    fi

   tinymix set 9 1
   tinymix set 1 0
#    tinymix set 7 175 175

    if [ $udiskOK -eq 1 ] && [ -d $UDISK_TRIMUI_DIR ] ; then
       echo "run from img" >> /tmp/imgrun.log
       export LD_LIBRARY_PATH=${UDISK_TRIMUI_DIR}/lib
       runifnecessary "keymon" ${UDISK_TRIMUI_DIR}/bin/keymon
       runifnecessary "inputd" ${UDISK_TRIMUI_DIR}/bin/trimui_inputd
       runifnecessary "scened" ${UDISK_TRIMUI_DIR}/bin/trimui_scened
       runifnecessary "trimui_btmanager" ${UDISK_TRIMUI_DIR}/bin/trimui_btmanager
       runifnecessary "hardwareservice" ${UDISK_TRIMUI_DIR}/bin/hardwareservice
       cd ${UDISK_TRIMUI_DIR}/bin/

       #absolute path for ps
       ${UDISK_TRIMUI_DIR}/app/premainui.sh
       ${UDISK_TRIMUI_DIR}/bin/MainUI
       ${UDISK_TRIMUI_DIR}/bin/preload.sh
       if [ $? -eq 0 ] || [ $? -eq 137 ] || [ $? -eq 143 ] ; then
          uiRunned=1
       else
          uiRunned=0
       fi
    fi

    if [ $uiRunned -eq 0 ] ; then
       export LD_LIBRARY_PATH=/usr/trimui/lib
          cd /usr/trimui/bin
       runifnecessary "keymon" keymon
       runifnecessary "inputd" trimui_inputd
       runifnecessary "scened" trimui_scened
       runifnecessary "trimui_btmanager" trimui_btmanager
       runifnecessary "hardwareservice" hardwareservice
       premainui.sh
       MainUI
       preload.sh
    fi

    if [ -f /tmp/trimui_inputd_restart ] ; then
      #restart before emulator run
      killall -9 trimui_inputd
      sleep 0.2
      runifnecessary "inputd" trimui_inputd
      rm /tmp/trimui_inputd_restart
    fi

    if [ -f /tmp/.cmdenc ] ; then
       /root/gameloader
    elif [ -f /tmp/cmd_to_run.sh ] ; then
      chmod a+x /tmp/cmd_to_run.sh
      udpbcast -f /tmp/host_msg &
         /tmp/cmd_to_run.sh
      rm /tmp/cmd_to_run.sh
         rm /tmp/host_msg
         killall -9 udpbcast
    fi
  fi
done
```
----------------------------------------
### /usr/trimui/bin/runtrimui.sh
* This script waits for the SDCARD to be mounted and then runs the updater if present,
* otherwise it runs the original runtrimui script.

```sh
#!/bin/sh

# becomes /usr/trimui/bin/runtrimui.sh on tg5040/tg3040

#wait for SDCARD mounted
mounted=`cat /proc/mounts | grep SDCARD`
cnt=0
while [ "$mounted" == "" ] && [ $cnt -lt 6 ] ; do
  sleep 0.5
  cnt=`expr $cnt + 1`
  mounted=`cat /proc/mounts | grep SDCARD`
done

UPDATER_PATH=/mnt/SDCARD/.tmp_update/updater
if [ -f "$UPDATER_PATH" ]; then
      "$UPDATER_PATH"
else
      /usr/trimui/bin/runtrimui-original.sh
fi
```
----------------------------------------

### /usr/trimui/bin/scene.sh
* This script runs all scene scripts in the /usr/trimui/scene/ directory with a given parameter.
* Parameters:
  - $1: Parameter to pass to each scene script
```sh
#!/bin/sh

echo "run scene $1"
cd /usr/trimui/scene/

for scene in `ls *.sh`
do
   killall $scene
   echo run script $scene
   ./$scene $1 &
done
```
----------------------------------------

### /usr/trimui/bin/usb_device.sh
* This script configures the system to operate in USB device mode.

```sh
#PH07 smartpro bottom
# echo 231 > /sys/class/gpio/export
# echo -n out > /sys/class/gpio/gpio231/direction
# echo 0 > /sys/class/gpio/gpio231/value

#PL08 back USB host
echo 360 > /sys/class/gpio/export
echo -n out > /sys/class/gpio/gpio360/direction
echo 1 > /sys/class/gpio/gpio360/value

# will disable inside OHCI0/EHCI0
# sleep 0.1
# echo 0 > /sys/devices/platform/soc/usbc0/axp_5v
cat /sys/devices/platform/soc/usbc0/usb_device
```
----------------------------------------
### /usr/trimui/bin/usb_dock.sh
* This script configures the system to operate in USB dock mode.

```sh
cat /sys/devices/platform/soc/usbc0/usb_host
sleep 0.2
echo 0 > /sys/devices/platform/soc/usbc0/axp_5v

#PH07  smartpro bottom
# echo 231 > /sys/class/gpio/export
# echo -n out > /sys/class/gpio/gpio231/direction
# echo 0 > /sys/class/gpio/gpio231/value

#PL08 back USB host
echo 360 > /sys/class/gpio/export
echo -n out > /sys/class/gpio/gpio360/direction
echo 1 > /sys/class/gpio/gpio360/value
```
----------------------------------------

### /usr/trimui/bin/usb_host.sh
* This script configures the system to operate in USB host mode.
```sh
cat /sys/devices/platform/soc/usbc0/usb_host
# sleep 0.1
echo 1 > /sys/devices/platform/soc/usbc0/axp_5v

#PH07 smartpro bottom
# echo 231 > /sys/class/gpio/export
# echo -n out > /sys/class/gpio/gpio231/direction
# echo 1 > /sys/class/gpio/gpio231/value

#PL08 back USB host
echo 360 > /sys/class/gpio/export
echo -n out > /sys/class/gpio/gpio360/direction
echo 1 > /sys/class/gpio/gpio360/value
```

