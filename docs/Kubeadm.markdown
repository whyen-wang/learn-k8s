---
layout: page
title: Kubeadm
permalink: /Kubeadm/
---

## Overview:
Network structure
![Network structure](/assets/Kubeadm/structure.svg)

## Step 1: Setup DHCP server
1. Get network interface name
```bash
ip addr
```
outputs may look like this:
```
1: lo: ...  <- virtual network interface
2: enp3s0f1: ...  <- ethernet network interface
3: wlp2s0: ...  <- wireless network interface
```
wireless network communicates with the external network through the WAN interface
ethernet network communicates with the external network through the LAN interface

2. Edit /etc/netplan/50-cloud-init.yaml
```
network:
    version: 2
    ethernets:
        enp3s0f1:
            dhcp4: false
            dhcp6: false
            addresses:
            - '10.0.0.1/24'
            optional: true
```

3. Install DHCP server and iptables-persistent
```bash
sudo apt install -y isc-dhcp-server iptables-persistent
```

4. Edit /etc/dhcp/dhcpd.conf
```
option domain-name "cluster.home";
option domain-name-servers 8.8.8.8, 8.8.4.4;
authoritative;
subnet 10.0.0.0 netmask 255.255.255.0 {
  range 10.0.0.1 10.0.0.10;
  option subnet-mask 255.255.255.0;
  option broadcast-address 10.0.0.255;
  option routers 10.0.0.1;
}
```
this file defines how the DHCP server assigns IP addresses and other network configuration parameters to client devices on a network.

5. Edit /etc/default/isc-dhcp-server
```
INTERFACESv4="enp3s0f1"
INTERFACESv6="enp3s0f1"
```
setup dhcp server interface

6. Edit /etc/sysctl.conf
```
net.ipv4.ip_forward=1
```
IP forwarding

7. Setup iptables
```bash
sudo iptables -t nat -A POSTROUTING -o wlp2s0 -j MASQUERADE
sudo iptables -A FORWARD -i wlp2s0 -o enp3s0f1 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i enp3s0f1 -o wlp2s0 -j ACCEPT
sudo chmod o+w /etc/iptables/rules.v4
sudo iptables-save > /etc/iptables/rules.v4
sudo chmod o-w /etc/iptables/rules.v4
```

8. Apply setting
```
sudo netplan apply
sudo sysctl -p
sudo systemctl restart netfilter-persistent
sudo systemctl restart isc-dhcp-server
sudo reboot
```

9. Verify
* Connect all device to network switch
```
cat /var/lib/dhcp/dhcpd.leases
```
outputs:
```
lease 10.0.0.1 {
  ...
}
lease 10.0.0.2 {
  ...
}
lease 10.0.0.3 {
  ...
}
lease 10.0.0.4 {
  ...
}
```