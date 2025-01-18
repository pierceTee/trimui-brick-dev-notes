# Below are my findings from digging arounnd the TrimUI brick filesystem.
An application to track your time played on trimUI devices

## Brick Firmware General
### 1-15-25

## Device Information

### Ram: 
- 975mb with ~400mb used by the OS.

### CPU: 
- [ARM Cortex-A53](https://developer.arm.com/documentation/ddi0500/j?lang=en) processor - ARMv8 64-bit 
- Clock speeds look like they range from 1.0GHz - 2.0GHz

The clock speeds can be manually set, as shown in the FnKeySettings script below.

####  Excerpt from `usr/trimui/apps/fn_editor/com.trimui.switch.cpufreq.sh`
```sh
#!/bin/sh
echo "============= toggle CPUFREQ ============"
CPUFREQ_MODE=0
CPUFREQ_MAX=`cat /sys/devices/system/cpu/cpufreq/policy0/scaling_max_freq`
CPUFREQ_MIN=`cat /sys/devices/system/cpu/cpufreq/policy0/scaling_min_freq`

echo "cpu scaling freq $CPUFREQ_MIN ~ $CPUFREQ_MAX"

#cpu is 2GHz

if test $CPUFREQ_MIN -gt 1700000; then
    CPUFREQ_MODE=4
else
    if test $CPUFREQ_MIN -gt 1500000; then
    #cpu is 1.8GHz
        CPUFREQ_MODE=3
    elif test $CPUFREQ_MIN -gt 1400000; then
    #cpu is 1.44GHz
        CPUFREQ_MODE=2
    fi
fi


if test $CPUFREQ_MIN -lt 1200000; then
if test $CPUFREQ_MAX -lt 1600000; then
#cpu is 1.0~1.44GHz
	CPUFREQ_MODE=1
else
#CPU is 1.0GHz ~ over 1.6GHz
	CPUFREQ_MODE=0
fi
fi

echo "get CPU MODE:"$CPUFREQ_MODE

# 	case "$1" in
# 	1 ) 
# 		echo "cpu save"
# 		echo ondemand > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
# 		;;
# 		echo "cpu normal"
# 		echo ondemand > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor                                                    
# 		;;
# 	*)
# 		;;
# 	esac


#1.2GHZ~2.0GHz
cpufreq0() {
	echo -n "1200000" > /sys/devices/system/cpu/cpufreq/policy0/scaling_min_freq 
	echo -n "2000000" > /sys/devices/system/cpu/cpufreq/policy0/scaling_max_freq 
}
#1.0GHZ~1.44GHz
cpufreq1() {
	echo -n "1008000" > /sys/devices/system/cpu/cpufreq/policy0/scaling_min_freq 
	echo -n "1440000" > /sys/devices/system/cpu/cpufreq/policy0/scaling_max_freq 
}
#1.44GHZ~1.8GHz
cpufreq2() {
	echo -n "1440000" > /sys/devices/system/cpu/cpufreq/policy0/scaling_min_freq 
	echo -n "1800000" > /sys/devices/system/cpu/cpufreq/policy0/scaling_max_freq 
}
#1.8GHZ
cpufreq3() {
	echo -n "1600000" > /sys/devices/system/cpu/cpufreq/policy0/scaling_min_freq 
	echo -n "1800000" > /sys/devices/system/cpu/cpufreq/policy0/scaling_max_freq 
}
#2.0GHz
cpufreq4() {
	echo -n "1800000" > /sys/devices/system/cpu/cpufreq/policy0/scaling_min_freq 
	echo -n "2000000" > /sys/devices/system/cpu/cpufreq/policy0/scaling_max_freq 
}

...


case "$CPUFREQ_MODE" in
0 ) 
    echo "set cpufreq mode 1"
    cpufreq1
    led_blink1
    ;;
1 ) 
    echo "set cpufreq mode 2"
    cpufreq2
    led_blink2
    ;;
2 ) 
    echo "set cpufreq mode 3"
    cpufreq3
    led_blink3
    ;;
3 ) 
    echo "set cpufreq mode 4"
    cpufreq4
    led_blink4
    ;;
4 ) 
    echo "set cpufreq mode 0"
    cpufreq0
    led_blink0
    ;;
* )
    echo "set cpufreq mode 0"
    cpufreq0
    led_blink0
    ;;
esac


#check again
CPUFREQ_MAX=`cat /sys/devices/system/cpu/cpufreq/policy0/scaling_max_freq`
CPUFREQ_MIN=`cat /sys/devices/system/cpu/cpufreq/policy0/scaling_min_freq`

echo "set cpu scaling freq $CPUFREQ_MIN ~ $CPUFREQ_MAX"


```

### Dev tips and tricks
- **ADB**  to get shell access and transfer files to and from the brick. Simply launch a terminal in the directory with files you want to transfer to/from and: 
  - Plug in the brick via the bottom usb-c port.
  - Run `adb devices` and copy the device ID that's listed. 
  
  ```sh
  sample output
  List of devices attached
  9c000c64a5424801d9d device
  ```
  - To execute a terminal instance, run `adb -s <DEVICE_ID> shell` i.e `adb -s 9c000c64a5424801d9d shell`
  - To transfer a file from the device, run `adb -s <DEVICE_ID> pull <SRC_DIR> <TARGET_DIR> ` i.e `adb -s 9c000c64a5424801d9d pull file.txt /documents`
  - To tranfer a file to the device, run  `adb -s <DEVICE_ID> push <SRC_DIR> <TARGET_DIR> ` i.e `adb -s 9c000c64a5424801d9d push /documents/file.txt /mnt/SDCARD/Tools`


- **Serial Debugging**: Alternative to ADB you can solder the TX/RX/GND lines on the back of the pcb, connect those to a USB to TTL device, use an application like putty to open serial communication, set the baud rate to 115200 and let it rip. The advantage of this over adb debugging is that serial communication is always running and doesn't require the adb service to be started. In most cases its best ot stick with adb though as its more consistent and user friendly.

Once in a serial terminal, run:
- `echo 0 > /proc/sys/kernel/printk` to disable kernel message logging to the console. This keeps the serial output clean by preventing system messages from cluttering your terminal session. 
- `dmesg -n 1` which sets the kernel message logging level to 1 (the least verbose)

```
1. Emergency (level 1)
2. Alert (level 2)
3. Critical (level 3)
4. Error (level 4)
5. Warning (level 5)
6. Notice (level 6)
7. Info (level 7)
8. Debug (level 8)
```
The combination of these two will prevent your inputs from being overwritten by the kernal logs.



- **Keeping the device awake**: `echo 1 >/tmp/stay_awake` to keep the device from going to sleep.

- **Package Management**: Brick used OpenWRT where all the system package information is stored in `/usr/lib/opkg/info/`, files here tell OpenWRT what files are associated with what packages, eg `/usr/lib/opkg/info/wifimanager-demo.list` has all the associated packages to `wifimanager`

- **Stock OS Resources**: All OS-level resources are in `/usr/trimui/res`, have fun swapping images and fonts out!


## Misc Dev notes
- Analyzing the usb_storage app (`usr/trimui/apps/usb_storage`) it would be sweet if I could mount the brick as an external storage device to swap files in and out.
