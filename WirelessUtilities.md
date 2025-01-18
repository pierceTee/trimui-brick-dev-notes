# Below are my findings from digging arounnd the TrimUI brick filesystem.
An application to track your time played on trimUI devices

## Wifi info 
### 1-17-25

So all wifi related actions/configs are located in `etc/wifi`.

Here you'll find `/etc/wifi/wpa_supplicant.conf` which contains wifi configuration params including the plaintext of user stored ssid/password pairs formatted like below 

```sh
ctrl_interface=/etc/wifi/sockets
disable_scan_offload=1
update_config=1
wowlan_triggers=any

network={
      ssid="PT Wifi"
      psk="5625875343"
}
```

 This is used by the [wpa_supplicant](https://wiki.archlinux.org/title/Wpa_supplicant) service in `etc/init.d/wpa_supplicant`. It's an off-the-shelf linux tool to that handles WPA authentication. 

Simply starting the service with `/etc/init.d/wpa_supplicant start` or `/etc/init.d/wpa_supplicant reload` (use `stop` to...stop) is enough to connect to a wireless network that's saved in `/etc/wifi/wpa_supplicant.conf`. `wpa_supplicant` can also handle network discovery/login but it requires an interterface to be built for it. In the meantime, we should just setup the network on the stock OS. 

While this will connect to a network, it won't finish DNS resolution or ip routing. I'm not entirely sure how this works just yet but checking the ip resolution config in `/etc/resolv.conf` gave

```nameserver 192.168.0.1` which is just the default local ip address setting AFAIk.
Checking `ip route` showed `192.168.0.0/24 dev wlan0 scope link  src 192.168.0.95`

then running `ip route add default via 192.168.0.1 dev wlan0` enabled pings.

TLDR: 
to start wifi, run `/etc/init.d/wpa_supplicant stop start && ip route add default via 192.168.0.1 dev wlan0`
For further deep diving, check out these MinUI paks from [Jose Gonzolez](https://github.com/josegonzalez): [wifi toggle](https://github.com/josegonzalez/trimui-brick-toggle-wifi-pak) [File Browser](https://github.com/josegonzalez/trimui-brick-filebrowser-pak), [Terminal](https://github.com/josegonzalez/trimui-brick-terminal-pak), [Remote Terminal](https://github.com/josegonzalez/trimui-brick-remote-terminal-pak), [SSH Server](https://github.com/josegonzalez/trimui-brick-dropbear-server-pak), [HTTP Server](https://github.com/josegonzalez/trimui-brick-dufs-server-pak), [FTP Server](https://github.com/josegonzalez/trimui-brick-sftpgo-server-pak), [Stay Awake Daemon](https://github.com/josegonzalez/minui-developer-pak)


