## NAT Traversal Demo

This project demonstrates how to implement Network Address Translation (NAT) traversal techniques, enabling devices behind different NAT networks to communicate directly. The project includes a simple NAT traversal implementation and a Docker environment for simulating real-world NAT network scenarios.

### What is NAT Traversal?

NAT traversal is a technique that allows devices located behind private networks (such as home or corporate networks) to communicate directly with similar devices, without requiring a public server to relay all data. In this demo, we use a STUN-like server to help two clients discover each other and establish a connection.

### Project Components

- STUN Server: Helps clients discover their public IP addresses and ports
- Two NAT Gateways: Simulate different private network environments
- Two Clients: Attempt to traverse NAT for direct communication
- Docker Environment: Provides isolated networks for testing

### Network Topology
The diagram below shows the network structure of the testing environment: two clients behind different NAT networks communicating through a STUN server.

```mermaid
flowchart LR
    %% Define server and gateway nodes
    tcp["stun Server<br>(Port:8090)"]:::server
    
    %% Define gateways
    subgraph gw1["NAT1 Gateway"]
        direction TB
        nat1_pub["Public Interface<br>(172.20.0.0/16)"]:::pub
        nat1_priv["Internal Interface<br>(172.18.0.0/16)"]:::priv
        nat1_pub <==> nat1_priv
    end
    
    subgraph gw2["NAT2 Gateway"]
        direction TB
        nat2_pub["Public Interface<br>(172.20.0.0/16)"]:::pub
        nat2_priv["Internal Interface<br>(172.19.0.0/16)"]:::priv
        nat2_pub <==> nat2_priv
    end
    
    %% Define client nodes
    peer1["Peer1<br>Client"]:::client
    peer2["Peer2<br>Client"]:::client
    
    %% Define network connections
    subgraph public["Public Network (172.20.0.0/16)"]
        tcp --- nat1_pub
        tcp --- nat2_pub
    end
    
    subgraph nat1net["NAT1 Private Network (172.18.0.0/16)"]
        nat1_priv --- peer1
    end
    
    subgraph nat2net["NAT2 Private Network (172.19.0.0/16)"]
        nat2_priv --- peer2
    end
    
    %% NAT communication flow
    peer1 -. "Default Route<br>via NAT1 Gateway" .-> nat1_priv
    nat1_pub -. "NAT Forwarding & Address Translation<br>(MASQUERADE)" .-> tcp
    
    peer2 -. "Default Route<br>via NAT2 Gateway" .-> nat2_priv
    nat2_pub -. "NAT Forwarding & Address Translation<br>(MASQUERADE)" .-> tcp
    
    %% Add global explanation
    note["<b>NAT Traversal Test Environment</b><br>Two clients behind different NATs<br>Using TCP Server for NAT traversal"]:::note
    
    %% Style definitions
    classDef server fill:#6366F1,color:white,stroke:#4F46E5,stroke-width:2px,rx:10,ry:10
    classDef pub fill:#F59E0B,color:white,stroke:#D97706,stroke-width:1px,rx:4,ry:4
    classDef priv fill:#10B981,color:white,stroke:#059669,stroke-width:1px,rx:4,ry:4
    classDef client fill:#EC4899,color:white,stroke:#DB2777,stroke-width:2px,rx:22,ry:22
    classDef note fill:#EFF6FF,color:#1E40AF,stroke:#DBEAFE,stroke-width:1px,rx:6,ry:6,font-size:12px

    %% Network styles
    style public fill:#F3F4F6,stroke:#E5E7EB,stroke-width:1px,rx:8,ry:8
    style nat1net fill:#ECFDF5,stroke:#D1FAE5,stroke-width:1px,rx:8,ry:8
    style nat2net fill:#FEF3C7,stroke:#FDE68A,stroke-width:1px,rx:8,ry:8
    
    %% Gateway styles
    style gw1 fill:#EFF6FF,stroke:#BFDBFE,stroke-width:2px,stroke-dasharray:5 5,rx:10,ry:10
    style gw2 fill:#EFF6FF,stroke:#BFDBFE,stroke-width:2px,stroke-dasharray:5 5,rx:10,ry:10
```

### How to Use

Please note the following:
1. **stun_server's eth0 subnet** is consistent with **nat1_gateway's eth1**/**nat2_gateway's eth1**.
2. **peer1's eth0 subnet** is consistent with **nat1_gateway's eth0** subnet.
3. **peer2's eth0 subnet** is consistent with **nat2_gateway's eth0** subnet.

Generally, the initialization order of network interfaces in Compose is consistent with the configuration file, but inconsistencies may occur sometimes. If the Compose environment does not meet the above assumptions, this environment is problematic and requires a restart or manual reconfiguration of the iptable NAT settings to match the current environment.

Here is the operation process:

1. Start the Environment and Run the STUN Server `nat-traversal`
```bash
# Start all Docker containers
$ docker-compose up -d
# Enter the STUN server container 
$ docker exec -it stun_server /bin/bash
# Get ip
$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if169: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether b6:ab:14:20:e9:fa brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.19.0.2/16 brd 172.19.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fd22:4d56:961b:1::2/64 scope global nodad 
       valid_lft forever preferred_lft forever
    inet6 fe80::b4ab:14ff:fe20:e9fa/64 scope link 
       valid_lft forever preferred_lft forever
# start the service
$ ./nat-traversal
```

1. Configure Client 1's Network

```bash
❯ docker exec -it nat1_gateway sh
/ # ip addr 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00 
    inet 127.0.0.1/8 scope host lo 
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever 
2: eth0@if170: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 0a:15:73:30:c7:30 brd ff:ff:ff:ff:ff:ff 
    inet 172.20.0.2/16 brd 172.20.255.255 scope global eth0 
       valid_lft forever preferred_lft forever 
    inet6 fd22:4d56:961b:2::2/64 scope global flags 02 
       valid_lft forever preferred_lft forever 
    inet6 fe80::815:73ff:fe30:c730/64 scope link 
       valid_lft forever preferred_lft forever 
3: eth1@if172: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 42:e9:0d:69:4c:c7 brd ff:ff:ff:ff:ff:ff 
    inet 172.19.0.3/16 brd 172.19.255.255 scope global eth1 
       valid_lft forever preferred_lft forever
    inet6 fd22:4d56:961b:1::3/64 scope global flags 02 
       valid_lft forever preferred_lft forever
    inet6 fe80::40e9:dff:fe69:4cc7/64 scope link 
       valid_lft forever preferred_lft forever
/ # exit

$ docker exec -it peer1 /bin/bash
# Set the default gateway (required to connect to the STUN server), copy nat1_gateway's eth0 address
$ ip route del default && ip route add default via 172.20.0.2
$ ip -6 route del default && ip -6 route add default via fd22:4d56:961b:2::2
```

3. Configure Client 2's Network
```bash
❯ docker exec -it nat2_gateway sh
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0@if171: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 9e:ae:ef:3d:b1:a1 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.2/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fd22:4d56:961b::2/64 scope global flags 02 
       valid_lft forever preferred_lft forever
    inet6 fe80::9cae:efff:fe3d:b1a1/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1@if173: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 12:75:82:61:f6:5e brd ff:ff:ff:ff:ff:ff
    inet 172.19.0.4/16 brd 172.19.255.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fd22:4d56:961b:1::4/64 scope global flags 02 
       valid_lft forever preferred_lft forever
    inet6 fe80::1075:82ff:fe61:f65e/64 scope link 
       valid_lft forever preferred_lft forever
/ # exit

$ docker exec -it peer2 /bin/bash
# Set the default gateway, copy nat2_gateway's eth0 address
$ ip route del default && ip route add default via 172.18.0.2
$ ip -6 route del default && ip -6 route add default via fd22:4d56:961b::2
```

4. Test NAT Traversal

Run the following commands on both clients (choose TCP or UDP mode):

#### TCP Mode Testing

1. Traversal with ipv4

```bash
$ ./nat-traversal 172.19.0.2:8090
```

output：
```bash
[2025-03-22T05:37:37Z INFO  nat_traversal_test::tcp] Listening on: 0.0.0.0:32827
[2025-03-22T05:37:51Z INFO  nat_traversal_test::tcp] Received address: [::ffff:172.19.0.4]:40879
[2025-03-22T05:37:51Z INFO  nat_traversal_test::tcp] Failed to connect to NAT: connection refused, Connection refused (os error 111)
[2025-03-22T05:37:51Z INFO  nat_traversal_test::tcp] Received message: "Hello, world!", from: 172.19.0.4:40879
```

2. Traversal with ipv6

```bash
$ ./nat-traversal [fd22:4d56:961b:1::2]:8090
```

output：
```bash
[2025-03-22T05:37:37Z INFO  nat_traversal_test::tcp] Listening on: [::]:44067
[2025-03-22T05:37:51Z INFO  nat_traversal_test::tcp] Received address: [fd22:4d56:961b:1::3]:44067
[2025-03-22T05:37:51Z INFO  nat_traversal_test::tcp] Failed to connect to NAT: connection refused, Connection refused (os error 111)
[2025-03-22T05:37:51Z INFO  nat_traversal_test::tcp] Received message: "Hello, world!", from: [fd22:4d56:961b:1::4]:32783
```

#### UDP Mode Testing

1. Traversal with ipv4
   
```bash
$ ./nat-traversal 172.19.0.2:8090 -p udp
```

output：
```bash
[2025-03-22T05:38:32Z INFO  nat_traversal_test::udp] Received address: [::ffff:172.19.0.4]:54957
[2025-03-22T05:38:32Z INFO  nat_traversal_test::udp] Received message: Hello, world! from 172.19.0.4:54957
[2025-03-22T05:38:32Z INFO  nat_traversal_test::udp] Received message: yes from 172.19.0.4:54957
```

2. Traversal with ipv6

```bash
$ ./nat-traversal "[fd22:4d56:961b:1::2]:8090" -p udp
```

output：
```bash
[2025-03-22T05:38:32Z INFO  nat_traversal_test::udp] Received address: [::ffff:172.19.0.4]:54957
[2025-03-22T05:38:32Z INFO  nat_traversal_test::udp] Received message: Hello, world! from [fd22:4d56:961b:1::4]:54957
[2025-03-22T05:38:32Z INFO  nat_traversal_test::udp] Received message: yes from [fd22:4d56:961b:1::4]:54957
```

### Troubleshooting

- Connection Failures: These are normal and may require multiple attempts. If testing gets stuck, retry using these methods:
  - TCP Mode: Shut down both clients and restart them
  - UDP Mode: Wait for the STUN server to output Udp clear NAT address log before trying again

### Current Limitations

This is a simple demonstration project with the following limitations:

- Each protocol supports only two clients simultaneously
- Birthday attack port detection is not implemented
- Complex scenarios like dns64/dns46 are not supported
- No fallback forwarding when traversal fails
- Testing in real network environments requires one public server and two servers behind NAT
