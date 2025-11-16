# Adding a little bit of routing

Now, unplug the NIC that goes to the school's network on the 192.168.45.2 (aka .2) host. You should obtain a blind computer (no internet access). We will use the other computer as a router.

## 1) Setting up blind computer (.2)

1.a) Backup the /etc/network/interfaces on .2 node.

1.b) Clean the current /etc/network/interfaces (remove unused interface).

1.c) Add a *gateway* directive to your /etc/network/interfaces. This directive in an alias for *ip route add default via <gateway>*. Why do we need this ? What is *gateway*’s value ?

1.d) Reboot and check routing tables. What can you see ?

1.e) On .1, find the IP of the interface plugged to the school’s network.

1.f) From .2 try to reach .1’s school’s IP address. Does it work ? Confront with .1 routing tables. Use *tcpdump* to find where the problem is.

## 2) Setting up the router (.1)

2.a) Check the *sysctl* key *net.ipv4.ip_forward*. What does it configure ? How can it help ? Why it is called *forward* ?

2.b) Setup the key and reboot to check persistence.

2.c) Retry your ping of the external interface of .1 from .2.

2.d) Target a classmate’s computer and try to reach it from your .1 and .2. Ask your classmate to capture the traffic in both situations and analyze the PCAP. What happened ?

2.e) How can you solve this issue ? You have to use a network facility but I won’t tell you there (this can influence the 2.d step !). But you can use chatgpt to describe your problem or try to find the network facility name with the following enigma. The encrypted network facility’s name is REX and the key is part of the string E4CYB :)
