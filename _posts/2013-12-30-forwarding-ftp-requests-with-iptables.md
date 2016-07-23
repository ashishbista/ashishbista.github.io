---
layout: post
title: "Forwarding FTP Requests with iptables"
date: 2013-12-30 21:00:25 +0545
comments: true
categories: [iptables, active ftp]
---

### Problem Background:

Normally, we'd prefer not to assign public IP, floating IP in OpenStack terminology, to each server in our cloud just because of a couple of reasons — public (IPv4) Internet addresses are a scarce public resource, and exposing all servers to the Internet is a security concern. 

For some purpose, within our OpenStack cloud, we had to install FTP server on a private instance and access it from outside of our cloud. We use an intermediate server (we call Route Server) to create a tunnel between our network and private servers. But, the FTP server has to be publicly accessible without establishing any tunnel. I used *iptables* to accomplish it.

### Solution:
Setup FTP on a private server.

Configure iptables on the Route Server so that FTP requests to it get routed to the actual FTP server.

### Implementation Description:
I installed `vsfdpd` on a  private server (say 192.168.100.11). To minimize firewall administration work, I decided to run FTP server on active mode. 

In active mode FTP, a client connects from a random port (N > 1023) to the FTP server's command port, port 21. Then, the client starts listening to port N+1 and sends the FTP command `PORT N+1` to the FTP server. The server will then connect back to the client's specified data port (N+1) from its local data port, which is port 20. The main problem with active mode FTP falls on the client side. The FTP client doesn't make the actual connection to the data port of the server — it simply tells the server what port it is listening on, and the server connects back to the specified port on the client. For the client side firewall, this appears to be a remote system initiating a connection to an internal client that is something usually blocked.

The issue of the server initiating the connection can be solved using it in passive mode. In passive mode FTP the client initiates both connections to the server, solving the problem of firewalls filtering the incoming data port connection to the client from the server. When opening an FTP connection, the client opens two random unprivileged ports locally. 

To disable passive mode, I added the following line in `/etc/vsftpd.conf`:

`pasv_enable=NO`

Then, I forwarded incoming FTP connections on the Route Server to the FTP server using `iptables`:

    modprobe nf_conntrack_ftp
    modprobe nf_nat_ftp

    sysctl net.ipv4.ip_forward=1
    iptables -t nat -A PREROUTING -p tcp --dport 20 -j DNAT --to-   destination 192.168.100.11:20
    iptables -t nat -A PREROUTING -p tcp --dport 21 -j DNAT --to-   destination 192.168.100.11:21
    iptables -t nat -A POSTROUTING -j MASQUERADE

Now, had to make iptables rules persistent. Otherwise iptables configuration would disappear after the system reboot.

So, I saved firewall rules to a file
`sudo sh -c "iptables-save > /etc/iptables.rules"`

Then, I added `pre-up iptables-restore < /etc/iptables.rules` in the `/etc/network/interfaces` file apply the rules automatically on system startup:

    auto lo
    iface lo inet loopback

    auto eth0
    iface eth0 inet static
            pre-up iptables-restore < /etc/iptables.rules
            address 192.168.100.6
            netmask 255.255.255.0
            broadcast 192.168.100.255
            gateway 192.168.100.1
            dns-nameservers 8.8.4.4

It's all how the FTP service running on a private server became accessible over the Internet.
