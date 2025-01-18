# Below are my findings from digging arounnd the TrimUI brick filesystem.
An application to track your time played on trimUI devices

## Firmware
### 1-15-25
The brick is running OpenWRT, a...router firmware aimed to give folks a fully configurable install but it primarliy focuses on networking funtionality, this is interesting because it is a single user OS (only root) so you can really get in there and mess things up, and it doesn't use any of the standard package managers. 

## Boot order

I need to do A LOT more digging on this. 

For the time being, check Services.md for a comprehensive look at the boot services ran in `/etc/rc.d`

All primary OS-level boot operations are defined there. That should be plenty for doing "OS" development. 

Writing our own FW or OS would require some further digging.




### Available commands (from `/bin`)
### System Commands

#### Core Utils

- `ash`, `busybox`, `sh` - Shells
- `cat`, `cp`, `ls`, `mkdir`, `mv`, `rm`, `rmdir` - File operations
- `chmod`, `chgrp`, `chown` - Permissions
- `date`, `df`, `dmesg`, `hostname` - System info
- `mount`, `umount` - Filesystem management
- `sync`, `sleep`, `usleep` - System control
- `touch`, `ln`, `mknod`, `mktemp` - File creation

#### Text Processing
- `base64` - Encoding
- `grep`, `egrep`, `fgrep`, `sed` - Text searching/manipulation
- `tar`, `gzip`, `gunzip`, `zcat` - Compression

#### Network Tools
- `adb_shell`, `adbd` - Android debug
- `ping`, `ping6` - Network testing
- `traceroute`, `traceroute6` - Route tracing
- `netstat`, `netmsg` - Network statistics
- `wget`, `uclient-fetch` - Download tools

#### Process Management
- `kill`, `pidof`, `ps` - Process control
- `mpstat`, `nice` - Process monitoring
- `lock`, `login` - Access control

#### WiFi Management
```
wifi_connect_ap_test               wifi_on_test
wifi_connect_ap_with_netid_test   wifi_reconnect_ap_test
wifi_connect_chinese_ap_test      wifi_remove_all_networks_test
wifi_disconnect_ap_test           wifi_remove_network_test
wifi_get_connection_info_test     wifi_scan_results_test
wifi_get_netid_test              wifi_wps_pbc_test
wifi_list_networks_test          wifi_longtime_scan_test
wifi_off_test                    wifi_longtime_test
wifi_on_off_test
```

#### Other Utilities
- `ipcalc.sh` - IP calculator
- `opkg` - Package manager
- `passwd` - Password management
- `setusbconfig` - USB configuration
- `softap_*` - Software AP tools
- `ubus` - System/process communication
- `vi` - Text editor


### Here's an AI summary of what these commands do :

===== AI START =====
#### Standard Unix Commands

**File and Directory Management:**
- `cp`, `mv`, `rm`, `ln`, `touch`, `mkdir`, `rmdir`
  - Used for copying, moving, removing files, creating links, and managing directories.

**Text Processing:**
- `cat`, `echo`, `grep`, `sed`, `awk`
  - Used for displaying, searching, and processing text.

**System Information:**
- `uname`, `df`, `du`, `free`, `uptime`
  - Provide information about the system, disk usage, memory usage, and system uptime.

**Process Management:**
- `ps`, `kill`, `top`
  - Used for listing processes, terminating processes, and monitoring system performance.

**Archiving and Compression:**
- `tar`, `gzip`, `gunzip`, `zcat`
  - Used for creating and extracting archives, and compressing and decompressing files.

#### Networking Tools

**Network Configuration and Testing:**
- `ifconfig`, `iwconfig`, `ping`, `traceroute`, `traceroute6`
  - Used for configuring network interfaces, testing network connectivity, and tracing network routes.

**Package Management:**
- `opkg`
  - The package manager used in OpenWRT for installing, updating, and removing software packages.

#### Wi-Fi Management Scripts

**Wi-Fi Connection and Management:**
- `wifi_connect_ap_test`, `wifi_connect_ap_with_netid_test`, `wifi_connect_chinese_ap_test`, `wifi_disconnect_ap_test`
  - Used for connecting to and disconnecting from Wi-Fi access points.

**Wi-Fi Information and Scanning:**
- `wifi_get_connection_info_test`, `wifi_get_netid_test`, `wifi_list_networks_test`, `wifi_scan_results_test`
  - Provide information about the current Wi-Fi connection, list available networks, and scan for Wi-Fi networks.

**Wi-Fi Control:**
- `wifi_on_test`, `wifi_off_test`, `wifi_on_off_test`, `wifi_reconnect_ap_test`, `wifi_remove_all_networks_test`, `wifi_remove_network_test`
  - Used for turning Wi-Fi on and off, reconnecting to access points, and managing saved networks.

#### System Management

**Synchronization and Sleep:**
- `sync`, `sleep`, `usleep`
  - Used for synchronizing cached writes to persistent storage and pausing execution for a specified duration.

**System Utilities:**
- `ubus`, `uclient-fetch`, `umount`, `mount`
  - Used for inter-process communication, fetching files from the network, and mounting and unmounting filesystems.

===== AI END =====

Looks like it uses `opkg` package manager but `opkg update` returns 
```
root@TinaLinux:/bin# opkg update
Downloading /base/Packages.gz.
wget: bad address ''
Downloading /kernel/Packages.gz.
wget: bad address ''
Collected errors:
 * opkg_download: Failed to download /base/Packages.gz, wget returned 1.
 * opkg_download: Failed to download /kernel/Packages.gz, wget returned 1.

``` 
even after enabling the wifi (pings working)...

Whats strang is that TrimUI itself is installed as an opkg (see `usr/lib/opkg/trimUI.list`) that shows all OS related files.
## Opkg
`opkg list installed` gives: 

```

MtpDaemon - 1-1
adb - 0.0.1-1
alsa-conf-aw - 1.0.0-1
alsa-lib - 1.1.4.1-1
alsa-utils - 1.1.0-1
app_usb_storage - 1.0.0-1
avahi-dbus-daemon - 0.6.31-12
aw869b-rftest - 1.0.0-1
base-files - 167-1730823064
benchmarks - 1
bgcmdhandler - 1.0-1
block-mount - 2016-01-10-96415afecef35766332067f4205ef3b2c7561d21
bluez-alsa - 20180913-1
bluez-daemon - 5.54-1
bluez-libs - 5.54-1
bluez-utils - 5.54-1
bluez-utils-extra - 5.54-1
boost - 1.64.0-1
boost-atomic - 1.64.0-1
boost-chrono - 1.64.0-1
boost-container - 1.64.0-1
boost-context - 1.64.0-1
boost-coroutine - 1.64.0-1
boost-date_time - 1.64.0-1
boost-fiber - 1.64.0-1
boost-filesystem - 1.64.0-1
boost-graph - 1.64.0-1
boost-iostreams - 1.64.0-1
boost-libs - 1.64.0-1
boost-log - 1.64.0-1
boost-math - 1.64.0-1
boost-program_options - 1.64.0-1
boost-python - 1.64.0-1
boost-random - 1.64.0-1
boost-regex - 1.64.0-1
boost-serialization - 1.64.0-1
boost-signals - 1.64.0-1
boost-system - 1.64.0-1
boost-test - 1.64.0-1
boost-thread - 1.64.0-1
boost-timer - 1.64.0-1
boost-wave - 1.64.0-1
brcm_patchram_plus - 0.0.1-1
btmanager-core - 0.0.0-1
btmanager-demo - 0.0.0-1
busybox - 1.27.2-3
common - 0.0.1-1
cpu_monitor - 1-1
curl - 7.54.1-1
dbus - 1.10.4-1
dbus-utils - 1.10.4-1
decodertest - 2.0-1
dnsmasq - 2.78-1
e2fsprogs - 1.42.12-1
eudev - 3.2.9-1
fbscreencap - 1.0.0-1
fbtest - 1
fdk-aac - 0.1.6-2
fneditor - 1.0-1
fontconfig - 2.12.1-3
fstools - 2016-01-10-96415afecef35766332067f4205ef3b2c7561d21
g2d_demo - 1
getevent - 1
glib2 - 2.50.1-1
gptfdisk - 1.0.10-1010
hardwareservice - 1
hostapd - 2017-11-08-2
iozone3 - 489-1
iperf - 2.0.10-1
iptables - 1.4.21-2
iw - 4.3-1
jshn - 2016-02-26-5326ce1046425154ab715387949728cfb09f4083
jsonfilter - 2014-06-19-cdc760c58077f44fc40adbbe41e1556a67c1b9a9
kernel - 4.9.191-1-c18835c982db6de7f484e357e2754835
keymon - 1
kmod-cfg80211 - 4.9.191-1
kmod-crypto-aead - 4.9.191-1
kmod-crypto-cmac - 4.9.191-1
kmod-crypto-ecb - 4.9.191-1
kmod-crypto-hash - 4.9.191-1
kmod-crypto-manager - 4.9.191-1
kmod-crypto-pcompress - 4.9.191-1
kmod-fuse - 4.9.191-1
kmod-ge8300-km - 4.9.191-2
kmod-hid - 4.9.191-1
kmod-input-core - 4.9.191-1
kmod-input-evdev - 4.9.191-1
kmod-ipt-core - 4.9.191-1
kmod-lib-crc16 - 4.9.191-1
kmod-net-xr829 - 4.9.191-1
kmod-nls-base - 4.9.191-1
kmod-sunxi-vin - 4.9.191-1
kmod-touchscreen-gt9xxnew_ts - 4.9.191-1
liballwinner-base - 1-1
libarelink - 0.1-1
libatomic - linaro-7.4-2019.02-1
libattr - 20150922-1
libavahi-client - 0.6.31-12
libavahi-compat-libdnssd - 0.6.31-12
libavahi-dbus-support - 0.6.31-12
libavahi-nodbus-support - 0.6.31-12
libblkid - 2.25.2-4
libblobmsg-json - 2016-02-26-5326ce1046425154ab715387949728cfb09f4083
libbz2 - 1.0.6-2
libc - 2.29-1
libcap - 2.24-1
libcedarx - 2.8-1
libconfig - 1.4.9-1
libcurl - 7.54.1-1
libcutils - 1-1
libdaemon - 0.14-5
libdbus - 1.10.4-1
libevdev - 1.4.6-1
libexpat - 2.1.0-3
libext2fs - 1.42.12-1
libffi - 3.3-1
libflac - 1.3.1-3
libfreetype - 2.6.1-2
libgamename - 0.1-1
libgcc - linaro-7.4-2019.02-1
libgcrypt - 1.7.0-1
libgmp - 6.1.0-1
libgnutls - 3.6.5-1
libgomp - 2.29-1
libgpg-error - 1.27-1
libgpu - 1
libical - 1.0-1
libid3tag - 0.15.1b-4
libinput - 1.5.0-1
libip4tc - 1.4.21-2
libip6tc - 1.4.21-2
libjpeg - 9a-1
libjson-c - 0.13.1-1
libjson-script - 2016-02-26-5326ce1046425154ab715387949728cfb09f4083
liblzma - 5.2.2-1
libmad - 0.15.1b-3
libncurses - 5.9-3
libncursesw - 5.9-3
libnettle - 3.4.1-2
libnghttp2 - 1.24.0
libnl-tiny - 0.1-5
libnss - 3.26-2
libogg - 1.3.2-2
liboil - 0.3.17-2
libopenssl - 1.1.0i-1
libopus - 1.1.3-2
libpcre - 8.38-2
libpixman - 0.34.0-1
libpng - 1.2.56-1
libpopt - 1.16-1
libpthread - 2.29-1
libreadline - 6.3-1
librt - 2.29-1
libscenemanager - 1-1
libsdl - 1.2.15-15
libsdl-image - 1.2.12-12
libsdl-mixer - 1.2.12-12
libsdl-ttf - 2.0.11-11
libsdl2 - 2.30.8-29
libsdl2-image - 2.0.1-20
libsdl2-mixer - 2.0.1-21
libsdl2-ttf - 2.0.13-13
libsdl_test - 1.2.15-15
libsec_key - 1-2
libsec_key-demo - 1-2
libshmvar - 0.1-1
libsndfile - 1.0.28-1
libsocket_db - 0.1-1
libsoup - 2.54.1-2
libsqlite3 - 3120200-1
libssp - linaro-7.4-2019.02-1
libstdcpp - linaro-7.4-2019.02-1
libtheora - 1.1.1-1
libthread-db - 2.29-1
libtmenu - 0.1-1
libuapi - 1-1
libubox - 2016-02-26-5326ce1046425154ab715387949728cfb09f4083
libubus - 2016-01-26-619f3a160de4f417226b69039538882787b3811c
libuci - 2016-02-02.1-1
libuclient - 2016-01-28-2e0918c7e0612449024caaaa8d44fb2d7a33f5f3
libump - ec0680628744f30b8fac35e41a7bd8e23e59c39f-1
libuuid - 2.25.2-4
libv4l - 1.20.0-1
libvorbis - 1.3.5-1
libvpx - 1.6.0-1
libxml2 - 2.9.3-1
libxtables - 1.4.21-2
libzstd - 1.4.4-2
live - 2019.02.27
logd - 2016-03-07-fd4bb41ee7ab136d25609c2a917beea5d52b723b
memtester - 4.3.0-1
moonlight - 2.6.1
moonlightui - 1.0-1
mtd - 21
mtdev - 1.1.5-1
netifd - 2016-02-01-3610a24b218974bdf2d2f709a8af9e4a990c47bd
nspr - 4.12-1
ntfs-3g - 2015.3.14-1-fuseint
ntfsprogs_ntfs-3g - 2015.3.14-1-fuseint
opengles_demo - 0.0.1-1
opkg - 9c97d5ecd795709c8584e972bfdf3aee3a5b846d-10
procd - 2016-02-08-57fe34225206c572b5e049826505f738191e4db5
resize2fs - 1.42.12-1
rwcheck - 1-1
sbc - 1.3-2
sdformater - 1.0.0-1
sdl2play - 1.0.0-1
sdl2teartest - 1.0.0-1
sdldisplay - 1.0.0-1
sdlteartest - 1.0.0-1
sdltest - 1.0.0-1
smartlinkd-demo - 0.2.1-1
smartlinkd-lib - 0.2.1-1
softap - 0.0.1-1
softap-demo - 0.0.1-1
stress - 1.0.4-1
systemval - 1.0-1
terminfo - 5.9-3
tinyalsa-lib - 1.1.1-34ffa583936aeb6938636c9c0a26a322b69b0d26
tinyalsa-utils - 1.1.1-34ffa583936aeb6938636c9c0a26a322b69b0d26
tplayerdemo - 1-1
tplayerslave - 1-1
trimui_btmanager - 1.0.0-1
trimui_inputd - 1.0.0-1
trimui_player - 1.0-1
trimui_scened - 1.0.0-1
trimusUI - 1.0-1 <---- weird, maybe I should be instally my stuff as an opkg?>
tslib - 1.15-2
tune2fs - 1.42.12-1
uboot-envtools - 2018.03-2
ubox - 2016-03-07-fd4bb41ee7ab136d25609c2a917beea5d52b723b
ubus - 2016-01-26-619f3a160de4f417226b69039538882787b3811c
ubusd - 2016-01-26-619f3a160de4f417226b69039538882787b3811c
uci - 2016-02-02.1-1
uclibcxx - 0.2.4-3
uclient-fetch - 2016-01-28-2e0918c7e0612449024caaaa8d44fb2d7a33f5f3
udpbcast - 1.0-1
usign - 2015-05-08-cf8dcdb8a4e874c77f3e9a8e9b643e8c17b19131
wget - 1.20.1-3
wifimanager - 0.0.1-1
wifimanager-demo - 0.0.1-1
wireless-tools - 29-5
wpa-cli - 2017-11-08-2
wpa-supplicant - 2017-11-08-2
x264 - 1
xr829-firmware - 1
xr829-rftest - 1.0.0-1
xradio - 0.0.1-1
zlib - 1.2.8-1
zoneinfo-africa - 2018i-1
zoneinfo-asia - 2018i-1
zoneinfo-atlantic - 2018i-1
zoneinfo-australia-nz - 2018i-1
zoneinfo-core - 2018i-1
zoneinfo-europe - 2018i-1
zoneinfo-india - 2018i-1
zoneinfo-northamerica - 2018i-1
zoneinfo-pacific - 2018i-1
zoneinfo-poles - 2018i-1
zoneinfo-simple - 2018i-1
zoneinfo-southamerica - 2018i-1
```