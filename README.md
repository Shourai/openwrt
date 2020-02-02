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

## Set up VLAN

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

