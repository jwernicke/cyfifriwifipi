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

#### Option 1: The Easy Way
Run `kalipi-config`, go to `Network Options`, then `Wi-Fi`. Enter SSID and passphrase. If you don't get an IP address, turn off the interface and bring it back up again:
```
$ ifdown wlan0; ifup wlan0
```
If that doesn't work, reboot the Pi.

#### Option 2: The CLI Way
First, find your wireless interfaces names:
```
iw dev
```
This will list each wireless interface. If you see `Unnamed/non-netdev interface`, you have Network Manager running and need to stop it or it'll interfere with things:
```
systemctl stop network-manager
```
Now you shouldn't see that `Unnamed/non-netdev interface` when you run `iw dev`.

Next, check your wireless interface's link status:
```
iw wlan0 link
```
It might say `Not connected.` so the next thing is to connect to some access point. Scan to see which access points your card can pick up.
```
iw wlan0 scan
```
You'll see a bunch of info so if you want to just find SSID, just pipe that command into `grep`.
```
iw wlan0 scan | grep SSID
```
Now you should just see a list of SSIDs. If you got the error `command failed: Network is down (-100)`, the interface was probably brought down when we brought down Network Manager so just bring it back up.
```
ip link set wlan0 up
```
Notice that's the `ip` command not the `iw` command this time. Now, your `iw wlan0 scan` command should run just fine. Assuming you found an access point you can connect to, configure your security settings for it. I hope your access point is using WPA, because that's what this tutorial is using. We're going to use [wpa_supplicant](https://en.wikipedia.org/wiki/Wpa_supplicant) to connect to our access point. Now, you could just write down the plaintext password for the access point in `/etc/network/interfaces`, but I think we can be a little more secure than that. We're going to use [wpa_passphrase](https://linux.die.net/man/8/wpa_passphrase) to compute a [PSK](https://www.techopedia.com/definition/22921/wi-fi-protected-access-pre-shared-key-wpa-psk) for us instead.
```
cp /etc/wpa_supplicant/wpa_supplicant.conf /etc/wpa_supplicant/wpa_supplicant.conf.orig
wpa_passphrase cyfifriwifipi > /etc/wpa_supplicant/wpa_supplicant.conf

```
You'll have to type in the password, but it won't echo back. Once you hit _Enter_, it'll write the configuration for your access point to `/etc/wpa_supplicant/wpa_supplicant.conf`. Go take a look at that file.
```
cat /etc/wpa_supplicant/wpa_supplicant.conf
```
We're almost there. Let's add the wireless configuration to `/etc/network/interfaces` using the editor of your choice:
```
iface wlan0 inet dhcp
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```
Now we can connect!
```
ifup wlan0
```
You might want to give your interface a static IP address just so you know what IP to hit when you want to remote into the Pi. Let's go back and change the entry in `/etc/network/interfaces` to whatever IP our DHCP server just gave us.
```
auto wlan0
iface wlan0 inet static
address 10.63.66.158
netmask 255.255.255.0
gateway 10.63.66.4
```
Now we can always SSH into `10.63.66.158`.

### Set up Pi-TFT
Sticky Fingers comes with some nifty scripts, one of which is `re4son-pi-tft-setup` used for setting up Pi TFT screens.
```
cd /usr/local/src/re4son-kernel_4*/re4son-pi-tft-setup
./re4son-pi-tft-setup
```
That will list all of the supported TFT screens. Pick your screen an run the command again.
```
./re4son-pi-tft-setup -t 35r
```
Follow the prompts and reboot.
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
