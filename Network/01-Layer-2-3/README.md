# Getting familiar with layer 2 and 3 network basics.

## 1) Plug everything (two-persons group)

The current situation is 2 computers with:
- A network interface connected to school network ;
- Additional network adapter with X interfaces ;
- A switch.

1.a) Install wireshark (graphical) and tcpdump on each host.

1.b) Connect the first free interface of each computer on the switch.

1.c) We will consider the network 192.168.45.0/24. Set IP address 192.168.45.1 on the first computer and 192.168.45.2 on the second. Here is a template for Debian distributions (adapt to your NIC name):

```bash
cat /etc/network/interfaces
auto enp0s3
iface enp0s3 inet static
        address 192.168.45.1/24
```

Explain the *auto* and *static* keywords.

1.d) Bring up your interface with *ifup <nicname>*.

1.e) Check your IP address with *ip*.

1.f) Display routing table with *ip route show*. Explain the output.

1.g) Check with *ping* that network stacks of both computers are reachable through network 192.168.45.0/24.

1.h) Reboot to check if your configuration is persistent.

## 2) Exploring ARP

In this section we will capture network traffic with *tcpdump* and analyze it with *wireshark* (GUI).

2.a) Use the *ip neigh* to check the ARP table. If you see ARP entries related to 192.168.45.0/24, flush them (*neigh* has a *flush* subcommand).

2.b) Capture an ICMP exchange between both computers with tcpdump through network 192.168.45.0/24. You should consider the *-w* that writes a PCAP file and filters like *(arp or (ip and net 192.168.45.0/24))*. You may also specify the relevant NIC with *-i*.

2.c) Analyze the ARP WHO-HAS and ARP IS-AT and draw the exchanges with relevant information, especially MAC addresses.

2.d) Compare both ICMP request and ICMP reply sent / received on both computers. You should pay attention to Ethernet and IP headers.



