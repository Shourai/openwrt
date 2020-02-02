# Documentation of openWRT settings

All configuration files are located at /etc/config/
The goal of this configuration is to place the guest wifi into a seperate vlan completely
separated from the rest of the ecosystem.

## Use WLAN port as lan port
In a dumb AP setup, it would be nice to be able to use the WAN port
as the port that is connected to the modem.
The default settings of openwrt puts the WAN interface on vlan 2 with WAN untagged.
and CPU tagged.

Set up the WAN interface with a static address.
With a private ip in the 192.168.2.0/24 range.
E.g. if you have the modem set to 192.168.2.254, set the private ip as 192.168.2.253.
For the gateway and DNS server set the modem as ip address i.e. 192.168.2.254.

If you like to like the router to also act as the DHCP server, activate that.

/etc/config/network
```
config interface 'wan'
        option ifname 'eth0.2'
        option proto 'static'
        option netmask '255.255.255.0'
        option gateway '192.168.2.254'
        option ipaddr '192.168.2.253'
        list dns '192.168.2.254'

config switch_vlan
        option device 'switch0'
        option vlan '2'
        option ports '1 0t'
        option vid '2'
```

/etc/config/dhcp
```
config dhcp 'wan'
        option interface 'wan'
        option start '100'
        option leasetime '12h'
        option limit '150'
```

## Set up VLAN for guest
Add a new vlan with a specific vlan id.
Make sure the CPU is tagged.
Any other port that is tagged will need the vlan id to access.

In the switch section, a new vlan should be created.
Also in the interfaces section, a new interface should be created for
the vlan.
This vlan should define the wireless access point (e.g. SSID) that will be in this vlan.
To do this, you need to bridge the wireless guest interface.

Give this vlan an static address with its own static ip.
Gateway needs not to be set up.
For the DNS server you can use the modem's DNS server (e.g. 192.168.2.254).
Which is in a different vlan.

DHCP will also be setup to let guests get a DHCP address.
The gateway needs not be set.

Also a firewall needs to be set up so the guests cannot access the rest of the network.

/etc/config/network
```
config switch_vlan
        option device 'switch0'
        option vlan '3'
        option vid '10'
        option ports '3 0t'

config interface 'vlan10'
        option proto 'static'
        option netmask '255.255.255.0'
        option ifname 'eth0.10'
        option ipaddr '192.168.10.1'
        option type 'bridge'
        list dns '192.168.2.254'
```

/etc/config/dhcp
```
config dhcp 'vlan10'
        option start '100'
        option leasetime '12h'
        option interface 'vlan10'
        option limit '50'
```

## Set up Firewall
The firewall will be set up to block the guest VLAN from accessing other VLANs
VLAN 10 is the guest VLAN. 
We want to first `reject` both the input and forward zones.
This will disallow vlan10 accessing any other zone.

We will however allow output, so the vlan can reach the internet.
The 'internet' is connected to the WAN port.

Finally we would like to allow people connecting to the guest wifi (or vlan10) to
be able to get a private ip address and can reach the DNS server.
For this we need to allow port 67 for DHCP and port 53 for DNS.

/etc/config/firewall
```
config zone
        option network 'vlan10'
        option name 'vlan10'
        option output 'ACCEPT'
        option input 'REJECT'
        option forward 'REJECT'

config forwarding
        option dest 'wan'
        option src 'vlan10'

config rule
        option src 'vlan10'
        option name 'Allow Guest DHCP'
        option target 'ACCEPT'
        option proto 'udp'
        option dest_port '67-68'

config rule
        option dest_port '53'
        option src 'vlan10'
        option name 'Allow Guest DNS'
        option target 'ACCEPT'
        option proto 'tcp udp'

config forwarding
        option dest 'lan'
        option src 'wan'
```

## Other settings
If you are using the openwrt router as a dumb access point.
And are using vlan2 as the wan vlan connected to the modem,
you need to also set the modem it is connected to a private ip
in the subnet of vlan2.
Something like  192.168.2.254.

