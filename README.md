# Kuiil -  Multi k8s clusters via wireguard

### Connect multiple private kubernetes clusters via wireguard. 

There are many approaches how to do that. In this case, we have chosen to have wireguard only in the masters node and use the masters node as a cluster router. You can also have wireguard installed in all the nodes and avoid the single-point-of-failure. In that case, you will need to adjust the routes accordingly. 

 Setup:

  - One public IP
  - Cluster 1
  - Cluster 2

### Network 

![alt text](https://github.com/keniack/kuiil/blob/master/network2.png?raw=true)

### Installation

##### Step1: Network design
You need to have different subnets and k8s-service-cidr for the clusters. 
Let's assume:
- Cluster 1 -  subnet : 192.168.100.0/24 and k8s-service-cidr: 10.244.0.0/16
- Cluster 2 -  subnet : 192.168.0.0/24 and k8s-service-cidr: 10.233.0.0/16
- Wireguard subnet : 192.168.99.0/24


##### Step2: wireguard

Install wireguard and setup according to the [wireguard documentation](https://www.wireguard.com/install/).

##### Step3: Wireguard post-installation setup 

After sucessful wireguard installation, you need to add the network subnets.

**Server setup**

Make sure the ip forwarding is enabled. 
```sh
$ sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
```

Adjust the server file __/etc/wireguard/wg0.conf__ to include the defined subnets in the allowed-ips.

```sh
[Peer-cluster1]
AllowedIPs = 192.168.99.10/32, 192.168.100.0/24,10.244.0.0/16

[Peer-cluster2]
AllowedIPs = 192.168.99.14/32, 192.168.0.0/24, 10.233.0.0/16
```

Restart wireguard service.

**Peer setup: cluster 1 master node**

Adjust the file __/etc/wireguard/wg0.conf__ to include the defined subnets.

```sh
[Interface]
Address =  192.168.99.10/24

PostUp = iptables -A FORWARD -d 192.168.99.0/24 -o wg0 -j ACCEPT;iptables -A FORWARD -d 192.168.100.0/24 -o eth0 -j ACCEPT;iptables -A FORWARD -d 192.168.0.0/24 -o wg0 -j ACCEPT;iptables -A FORWARD -d 10.233.0.0/16 -o wg0 -j ACCEPT;

PostDown = iptables -A FORWARD -d 192.168.99.0/24 -o wg0 -j ACCEPT;iptables -A FORWARD -d 192.168.100.0/24 -o eth0 -j ACCEPT;iptables -A FORWARD -d 192.168.0.0/24 -o wg0 -j ACCEPT;iptables -A FORWARD -d 10.233.0.0/16 -o wg0 -j ACCEPT;

[Peer]
AllowedIPs = 192.168.99.0/24,192.168.0.0/24,10.233.0.0/16
```

**Peer setup: cluster 2 master node**

Adjust the file __/etc/wireguard/wg0.conf__ to include the defined subnets.

```sh
[Interface]
Address = 192.168.99.14/24


PostUp = iptables -A FORWARD -d 192.168.99.0/24 -o wg0 -j ACCEPT;iptables -A FORWARD -d 192.168.0.0/24 -o eth0 -j ACCEPT; iptables -A FORWARD -d 192.168.100.0/24 -o wg0 -j ACCEPT;iptables -A FORWARD -d 10.244.0.0/16 -o wg0 -j ACCEPT;

PostDown = iptables -A FORWARD -d 192.168.99.0/24 -o wg0 -j ACCEPT;iptables -A FORWARD -d 192.168.0.0/24 -o eth0 -j ACCEPT;iptables -A FORWARD -d 192.168.100.0/24 -o wg0 -j ACCEPT;iptables -A FORWARD -d 10.244.0.0/16 -o wg0 -j ACCEPT;


[Peer]
AllowedIPs = 192.168.99.0/24,10.244.0.0/16,192.168.100.0/24
```

**Iptables exaplation:**  since we are using masters node as router for each cluster, we need to forward the packets from the interfaces. Therefore, the iptables rules needs to be included.

##### Step4: Add routes in the clusters nodes

In **every** node of each cluster, we need to include the routes from the other cluster. 

**In cluster 1 nodes**:

In cluster 1 nodes,  include routes from cluster 2:

```sh
#add cluster2 subnet via cluster1-master-node-ip
ip r add 192.168.0.0/24 via 192.168.100.100

#add wg subnet via cluster1-master-node-ip
ip r add 192.168.99.0/24 via 192.168.100.100

#add cluster2-service-cidr via cluster1-master-node-ip
ip r add 10.233.0.0/16 via 192.168.100.100
```
**explanation:** whenever this node needs to communicate with any ip from the subnets[192.168.0.0/24,192.168.99.0/24,10.233.0.0/16] it will forward the packets to the cluster1-master-node-ip [eth0:192.168.100.100]

**In cluster 2 nodes**:

In cluster 2 nodes,  include routes from cluster 1:

```sh
#add cluster1 subnet via cluster2-master-node-ip
ip r add 192.168.100.0/24 via 192.168.0.100

#add wg subnet via cluster2-master-node-ip
ip r add 192.168.99.0/24 via 192.168.0.100

#add cluster1-service-cidr via cluster2-master-node-ip
ip r add 10.244.0.0/16 via 192.168.0.100
```
**explanation:** whenever this node needs to communicate with any ip from the subnets [192.168.100.0/24,192.168.99.0/24,10.244.0.0/16] it will forward the packets to the cluster2-master-node-ip [eth0:192.168.0.100]


Now you should be able to connect to any pod/node from the alien cluster. 

### Kuiil
Kuiil will enable such configuration and more. While the tool is not ready, the configuration can be done manually as described above!
