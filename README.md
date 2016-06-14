syncookied
==========

How to run
==========

On server you want to protect
------------------------------
1. Install tcpsecrets linux kernel mode to expose tcp syncookie key and timestamp
2. Start syncookied in `server` mode: `syncookied server <ip:port>`. Running this 
commands automatically sets `net.ipv4.tcp_syncookies` to 2 (always) and starts a UDP server on specified ip/port.

On server you want to use for packet processing
-----------------------------------------------
1. Install [netmap](https://github.com/luigirizzo/netmap) and make sure it works (pkt-gen)
2. Disable NIC offloading features on the interface you want to use (eth2 here):
```
ethtool -K eth2 gro off gso off tso off lro off rx off tx off 
ethtool -A eth2 rx off tx off
ethtool -G eth2 rx 2048 tx 2048
```
3. Set up queues and affinities. Here we bind 12 queues to first 12 cpu cores:
```
QUEUES=12
ethtool -L eth2 combined $QUEUES
./set_irq_affinity -x 0-11 eth2
```
set_irq_affinity is available at https://github.com/majek/ixgbe/blob/master/scripts/set_irq_affinity
4. Create hosts.yml file in the working directory, which looks like this
```
- ip: 185.50.25.4
  local_ip: 192.168.3.231:1488
  mac: 0c:c4:7a:6a:fa:bf
```
Here ip is the ip you want to protect, local_ip is the address where you run the UDP server and mac is the protected server's mac address.
5. Run `syncookied -i eth2`. It will print something like this:
```
Configuration: 185.50.25.4 -> c:c4:7a:6a:fa:bf
interfaces: [Rx: eth2/3c:fd:fe:9f:a8:82, Tx: eth2/3c:fd:fe:9f:a8:82] Cores: 24
12 Rx rings @ eth2, 12 Tx rings @ eth2 Queue: 1048576
Starting RX thread for ring 0 at eth2
Starting TX thread for ring 0 at eth2
Uptime reader for 185.50.25.4 starting
...
```
6. Configure your network equipment to direct traffic for protected ip to syncookied.
7. You can reload configuration at any time by changing hosts.yml and sending HUP signal to syncookied. 
It will print something like this:
```
Uptime reader for 185.50.25.4 exiting
All uptime readers dead
Old readers are dead, all hail to new readers
Uptime reader for 185.50.25.4 starting
...
```
8. Enjoy your ddos protection

Notes
-----
`syncookied` has some options you may want to tune, see `syncookied --help`.
If you have more than 1 interface on your server, you may want to look into -O to use second one for TX. 
This greatly improves performance and latency as forwarding and syn-reply traffic is separated.