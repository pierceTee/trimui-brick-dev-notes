# Services and Daemons

## General information
To my knowledge, all services/daemons are located in the `/etc` directory

Initialization scripts/daemon are expected to be managed by `/etc/rc.common`. [Check out the official OpenWrt documentation ](https://openwrt.org/docs/techref/initscripts) for a better explanation of how boot scripts work that I can provide.

Breifly, the simplest scripts take the form of 
```sh
#!/bin/sh /etc/rc.common
# Example script
# Copyright (C) 2007 OpenWrt.org
 
START=10
STOP=15
 
start() {        
        echo start
        # commands to launch application
}                 
 
stop() {          
        echo stop
        # commands to kill application 
}
```
Key notes: 
  - `Calls /etc/rc.common` first to manage all script execution.
  - `START`: The 0-99 ordering of when this script should run in the init sequence (scripts with START=9 start before scripts with START=10)
  - `END (optional)`: The 0-99 ordering of when the script should stop in the init sequence.
  - `Implements start() and stop()`

The contents of the functions take standard bash commands but also seem to run C, check the documentation for more information but generally doing something like my [settings daemon](https://github.com/pierceTee/TrimuiLEDController/blob/master/workspace/service/led-settings-daemon) that simply calls another shell script on boot should be sufficient for most app work.

To enable the service, run `<scriptname> enable` ([example](https://github.com/pierceTee/TrimuiLEDController/blob/master/workspace/scripts/runtime/install.sh#L16)). 

This will symlink your script file to `/etc/rc.d/S<START_VALUE><DAEMON_NAME>` and `/etc/rc.d/K<END_VALUE><DAEMON_NAME>`.

These are files that OpenWrt looks for to run during the boot sequence. 

to `uninstall` run `<scriptname> enable` which will remove the symlinks.

### Extras notes

Look into the usage of 
- `boot()`: Runs everytime the system turns on. 
- `shutdown()`: Runs everytime the system turns off.

Query the state of all scripts with the following command 
`for F in /etc/init.d/* ; do $F enabled && echo $F on || echo $F **disabled**; done
`
## Boot/Initialization Services
All boot daemons are located in `/etc/init.d`. 

As discussed above, on boot OpenWRT runs all the daemons found installed from init.d. 

The boot order of installed services is as follows : 


```
S00sysfixtime -> ../init.d/sysfixtime
S10boot -> ../init.d/boot
S10dbus -> ../init.d/dbus
S10system -> ../init.d/system
S11sysctl -> ../init.d/sysctl
S12log -> ../init.d/log
S20network -> ../init.d/network
S40fstab -> ../init.d/fstab
S50cron -> ../init.d/cron
S50fontconfig -> ../init.d/fontconfig
S60dnsmasq -> ../init.d/dnsmasq
S61avahi-daemon -> ../init.d/avahi-daemon
S80adbd -> ../init.d/adbd
S80hciattach -> ../init.d/hciattach
S80udev -> ../init.d/udev
S95done -> ../init.d/done
S96wpa_supplicant -> ../init.d/wpa_supplicant
S98mtp -> ../init.d/mtp
S98sysntpd -> ../init.d/sysntpd
S99runtrimui -> ../init.d/runtrimui
```

## 1. [/etc/init.d/sysfixtime](https://github.com/openwrt-mirror/openwrt/blob/master/package/base-files/files/etc/init.d/sysfixtime)
### Standard OpenWRT script that sets up the real time and hardware clocks. 

## 2. [/etc/init.d/boot](https://github.com/openwrt-mirror/openwrt/blob/master/package/base-files/files/etc/init.d/boot)
### Standard OpenWRT script that mounts the filesystem, loads some kernel modules, detects components from the board, etc.

## 3. [/etc/init.d/dbus]
### Standard OpenWRT script that initializes [dbus](https://dbus.freedesktop.org/doc/dbus-tutorial.html) for interprocess communication

## 4. [/etc/init.d/system](https://github.com/openwrt-mirror/openwrt/blob/master/package/base-files/files/etc/init.d/system)
### Standard OpenWRT script that loads OpenWRT system configs.

## 5. [/etc/init.d/sysctl](https://github.com/openwrt-mirror/openwrt/blob/master/package/base-files/files/etc/init.d/sysctl)
### Standard OpenWRT script that loads kernel config files from `/etc/sysctl.d/*.conf` 

## 6. [/etc/init.d/log](https://github.com/openwrt-mirror/openwrt/blob/master/package/base-files/files/etc/init.d/sysctl)
### Standard OpenWRT script that loads kernel config files from `/etc/sysctl.d/*.conf`

## 7. /etc/init.d/network
### Custom script that starts/stops the network interface for wifi communications.

## 8. /etc/init.d/fstab
### Standard OpenWRT script that mounts /sbin/block to redirect kernel messages.

## 9. /etc/init.d/cron
### Standard OpenWRT script that manages the [cron](https://openwrt.org/docs/guide-user/base-system/cron) (task scheduler)

## 10. /etc/init.d/fontconfig
### Script that rebuilds the [FontConfig](https://wiki.archlinux.org/title/Font_configuration) configuration.

## 10. /etc/init.d/dnsmasq
### Standard OpenWRT script that configures the dnsmasq. See [Dnsmasq DHCP server](https://openwrt.org/docs/guide-user/base-system/dhcp.dnsmasq) for more information.

## 11. /etc/init.d/avahi-daemon
### Standard OpenWRT script that initializes the Avahi daemon for network service discovery. See [Zero configuration networking in OpenWrt](https://openwrt.org/docs/guide-user/network/zeroconfig/zeroconf) This is essential for network discovery via Bonjour.

## 12. /etc/init.d/adbd
### Standard OpenWRT script that starts the Android Debug Bridge daemon. Theres not much info about it online as the main page is offline. [Here's the best I could find](https://openwrt.org/toh/adb/start)

## 13. /etc/init.d/hciattach
### Standard OpenWRT script that attaches a serial device to a Bluetooth stack.

## 14. /etc/init.d/udev
### Custome script that manages device detection.

## 15. /etc/init.d/done
### Standard OpenWRT script that signals the completion of the boot process. It starts after `fstab`,`nativepower`,`mtp`, and `adbd`. 

Initilizes the root filesystem, then runs `/etc/rc.local` which is user configurable to run commands after the system initializes, TrimUI have overridden it as follows. 

```sh
rootÉTinaLinux:/etc# cat rc.local
# Put your custom commands here that should be executed once
# the system init finished. By default this file does nothing.

# hpout playback
amixer -D hw:audiocodec cset name='Headphone Switch' 1
amixer -D hw:audiocodec cset name='Headphone Volume' 3
amixer -D hw:audiocodec cset name='HpSpeaker Switch' 1

# lineout playback
#amixer -D hw:audiocodec cset name='LINEOUT Output Select' 'DAC_DIFFER'
#amixer -D hw:audiocodec cset name='LINEOUT Switch' 1
#amixer -D hw:audiocodec cset name='LINEOUT volume' 20

# aec reference capture
amixer -D hw:audiocodec cset name='ADCL Input MIC1 Boost Switch' 1
amixer -D hw:audiocodec cset name='ADCR Input MIC2 Boost Switch' 1
amixer -D hw:audiocodec cset name='MIC1 gain volume' 0
amixer -D hw:audiocodec cset name='MIC2 gain volume' 0

# mic capture
amixer -D hw:sndac10710036 cset name='Channel 1 PGA Gain' 20
amixer -D hw:sndac10710036 cset name='Channel 2 PGA Gain' 20

exit 0
```

It seems they just initialized the sounds system.

## 16. [/etc/init.d/wpa_supplicant](https://wiki.archlinux.org/title/Wpa_supplicant)
### Standard OpenWRT script that manages the WPA supplicant for Wi-Fi security. see the Wifi section of WirelessUtilites.md

## 18. /etc/init.d/mtp
### Standard OpenWRT script that initializes the Media Transfer Protocol. It starts after `fstab` and `system`. 

## 19. /etc/init.d/sysntpd
### Standard OpenWRT script that synchronizes the system time using NTP. Checkout [OpenWrt's NTP documentation](https://openwrt.org/docs/guide-user/advanced/ntp_configuration)  for more info.

## 20. /etc/init.d/runtrimui
### Custom script that starts the TrimUI application.

This is the meat and potatoes of what users will experience with the device. Everything that runs before this is simply necessary setup in order for the Operating system to run.


```sh
#!/bin/sh /etc/rc.common

/usr/sbin/pic2fb /etc/splash.png

START=99
PROG=/usr/trimui/bin/runtrimui.sh

start() {
	$PROG
}

```
 This service is basically just a forwarder for `/usr/trimui/bin/runtrimui.sh`.

Custom OS's override this script to do their setip processes, below is an example of what MinUI does.
You'll see it checkes for the `SDCARD/.tmp_update` file to load its own OS. Otherwise it runs the stock OS script.

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
 That then calls the original version of that script which I've placed below.
 
 There's some interesting things to note:
 - /tmp/.try_upgrade_file is unconditionally read, probably not how you'd want to install a custom OS...but you could.
 - Daemons aren't even necessary because all .sh files in `/mnt/SDCARD/System/starts` will always be run on boot.
 - It is already setup to run a custom OS before running stock OS, so long as you implement `/mnt/SDCARD/trimui/app/premainui.sh`, `/mnt/SDCARD/trimui/app/MainUI` and  `/mnt/SDCARD/trimui/app/preload.sh`, and `/mnt/SDCARD/trimui/app/preload.sh` returns 0, you can power on and have the console only run your app.  
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
      # [MY NOTES ] This looks like it was put here for custom OS's to launch?
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
            
      # [MY NOTES ] If a custom OS didn't run, mount the stock OS 
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

      # [MY NOTES ] If the stock OS was able to mount, then run the OS from the device storage.
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

    # [MY NOTES ] If no OS has run up to this point, then run the OG UI found in `/usr/trimui/bin`
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


TrimUI has defined their own OS daemon to start last in the order 
As seen in `etc/init.d/runtrimui`

Looks like MinUI actually does exactly what I'm thinking of doing here, they move `usr/trimui/bin/runtrimui.sh` `usr/trimui/bin/runtrimui-original.sh` and replace it with `skeleton/BOOT/trimui/app/runtrimui.sh` which then launches MinUI's `.tmp_update/updater` (or `runtrimui-original.sh` if the SDCARD isn't found). 

`usr/trimui/bin/trimui_inputd` is the input daemon written in C so maybe this is where runtime daemons go?

Tons of more interesting things in `usr/trimui`, all the stock apps are in `usr/trimui/apps` and can easily be moved to the SDCARD for use in other OSs, alternatively we can move our app in here for a "permanent" install. `usr/trimui/gamecontrollerdb.txt` seemingly has the mapping of controllers to SDL inputs, if we wanted to add support for a certain controller, it would go there.
