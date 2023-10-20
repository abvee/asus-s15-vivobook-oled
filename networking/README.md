# Basic Networking
## iwd
The model of of Wifi card on this laptop used `iwlwifi`, and it is an Intel Wifi
card. As such, it plays very nicely with `iwd`. `iwd` is both a connection
manager and a DHCP client, and is able to assign static and dynamic ip
addresses. I am using `iwd` for both DHCP and network connections. Enable DHCP
by putting this in `/etc/iwd/main.conf`:
```
[General]
EnableNetworkConfiguration=true
```

We also need to choose a DNS resolver. I have chosen `systemd-resolved` as it's
preinstalled with most distros that ship systemd as their init system. Note that
you will have to enable/start systemd-resolved.service:
```
systemctl enable systemd-resolved --now
```
We can now tell `iwd` that we want `systemd-resolved` as our DNS manager. Add
this in `/etc/iwd/main.conf`:
```
[Network]
NameResolvingService=systemd
```
Start and enable iwd, and you should have internet. Follow [this](https://wiki.archlinux.org/title/iwd) Arch Wiki page on
iwd for more information on how to connect and do other things
