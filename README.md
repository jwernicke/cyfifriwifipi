# rpiwifihunter
Hunting rogue access points with a Raspberry Pi

## Getting Started
Things you'll need:
* A wireless AP/router.
* 3 or more Raspberry Pis. This has been tested with 3 Model B v1.2, and Zero W.
* [Re4son's Kali Pi](https://whitedome.com.au/re4son/sticky-fingers-kali-pi-pre-installed-image/)
* USB keyboard and mouse, and a HDMI monitor to interface with the Pi.
* TP-Link TL-WN722N High-Gain WIreless USB Adapter

## Set up Kali
The default user is `root` and password is `toor`.

### Connect to your network

Run `kalipi-config`, go to `Network Options`, then `Wi-Fi`. Enter SSID and passphrase. If you don't get an IP address, turn off the interface and bring it back up again:
```
$ ifdown wlan0; ifup wlan0
```
If that doesn't work, reboot the Pi.

### SSH into Pi
Now that your Pi is connected to the network, write down its IP address. Then, get rid of the pesky monitor, keyboard, and mouse, and SSH into it from your laptop. Remember that the Pi is only configured to connect to that network so you'll have to reconnect the peripherals if you want to add a different network. You could also configure a static IP address on `eth0` or set up an access point on the Raspberry Pi to be able to connect to it when the main network is unavailable.
```
ssh root@<YOUR_PI_IP_ADDRESS>
```

### Set up high-gain wireless card
Plug in your wireless card to a USB slot. Edit `/etc/udev/rules.d/70-persistent-net.rules` to assign `wlan1` to this device.
```
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="", ATTR{dev_id}=="0x0", ATTR{type}=="1", KERNEL=="wlan*", NAME="wlan1"
```

### Set up Kismet
Edit the following parameters in `/etc/kismet/kismet_drone.conf`:
```
servername=<YOUR_PI_IP>
dronelisten=tcp://<YOUR_PI_IP>:<LISTEN_PORT>
allowedhosts=<LIST OF ALLOWED HOSTS>
droneallowedhosts==<LIST OF ALLOWED DRONES>
dronemaxclients=10
droneringlen=65535
gps=false
ncsource=wlan1
```

## Troubleshooting

<a name="troubleshooting-network"></a>
### Can't connect to network
Make sure your `eth0` interface is enabled in `/etc/network/interfaces`:
```
auto eth0
iface eth0 inet dhcp
```
If that isn't the problem, try turning your interface off and back on again:
```
root@kali:~# ifdown eth0
Killed old client process
Internet Systems Consortium DHCP Client 4.3.5
Copyright 2004-2016 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

Listening on LPF/eth0/b8:27:eb:2f:95:11
Sending on   LPF/eth0/b8:27:eb:2f:95:11
Sending on   Socket/fallback
DHCPRELEASE on eth0 to 10.63.66.3 port 67
root@kali:~# ifup eth0
Internet Systems Consortium DHCP Client 4.3.5
Copyright 2004-2016 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

Listening on LPF/eth0/b8:27:eb:2f:95:11
Sending on   LPF/eth0/b8:27:eb:2f:95:11
Sending on   Socket/fallback
DHCPDISCOVER on eth0 to 255.255.255.255 port 67 interval 3
DHCPDISCOVER on eth0 to 255.255.255.255 port 67 interval 3
DHCPREQUEST of 10.63.66.225 on eth0 to 255.255.255.255 port 67
DHCPOFFER of 10.63.66.225 from 10.63.66.3
DHCPACK of 10.63.66.225 from 10.63.66.3
bound to 10.63.66.225 -- renewal in 17354 seconds.
```
