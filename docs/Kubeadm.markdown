---
layout: page
title: Kubeadm
permalink: /Kubeadm/
---

## Step 1: Setup DHCP server
Network structure:
![Network structure](/assets/Kubeadm/network_structure.svg)

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
net.ipv4.ip_forward=1  # IP forwarding
```

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
```bash
sudo netplan apply
sudo sysctl -p
sudo systemctl restart netfilter-persistent
sudo systemctl restart isc-dhcp-server
sudo reboot
```

9. Verify
* Connect all devices to network switch
```bash
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

## Step 2: Install Docker Engine and Setup CRI (Container Runtime Interface)
Install docker engine and setup CRI all nodes.
Container Runtime Interface:
![Container Runtime Interface](/assets/Kubeadm/container_runtime_interface.svg)

1. Configure prerequisites
```bash
# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```
Verify:
```bash
sysctl net.ipv4.ip_forward
```

2. Set up Docker's apt repository
```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

3. Install the Docker packages
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

4. Configure the containerd
  - /etc/containerd/config.toml
  ```
  # remove "cri" from the list of disabled_plugins
  disabled_plugins = []
  ```

5. Restart containerd
```bash
sudo systemctl restart containerd
```

6. Verify:
```bash
sudo systemctl status docker.service
sudo systemctl status docker.socket
sudo systemctl status containerd.service
```

## references
1. [docker engine](https://docs.docker.com/engine/install/ubuntu/)
2. [kubernetes](https://kubernetes.io/docs/setup/production-environment/)