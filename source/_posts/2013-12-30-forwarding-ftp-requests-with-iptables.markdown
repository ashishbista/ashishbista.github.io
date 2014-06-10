---
layout: post
title: "Forwarding FTP Requests with iptables"
date: 2013-12-30 21:00:25 +0545
comments: true
categories: ["iptables, active ftp"]
---

### Problem Background:

We don't prefer to assign public IP, floating IP in OpenStack terminology, to all instances in our cloud just because of a couple of reasons - we always run in shortage of public IPs and we don't want some instances be publicly accessible. For some purpose, within our OpenStack cloud, we had to install FTP server on a private instance and access it from outside of our cloud. Since the instance had no public IP, we had to access it through an instance which is publicly accessible. There was an instance designated as *Route Server* and it's purpose was act as an intermediate server for accessing publicly none-routable instances. In addition, our Rails deployment process only works in this server. I used *iptables* to overcome this problem.

### Solution I Though:
Setup FTP server on the private instance.

Configure iptables on the Route Server so that FTP requests to it get routed to the private instance.

Users make FTP requests to the Route Server but the private server, the server in which FTP is installed, responses to requests in real.

### Implementation Description:

I installed `vsftpd` on the  private server (say 192.168.100.11). In order to minimize firewall administration work I decided to run FTP server on active mode. In active mode FTP, the client connects from a random port (N > 1023) to the FTP server's command port, port 21. Then, the client starts listening to port N+1 and sends the FTP command `PORT N+1` to the FTP server. The server will then connect back to the client's specified data port from its local data port, which is port 20. The main problem with active mode FTP actually falls on the client side. The FTP client doesn't make the actual connection to the data port of the server - it simply tells the server what port it is listening on and the server connects back to the specified port on the client. For the client side firewall, this appears to be a remote system initiating a connection to an internal client that is something usually blocked.

To disable passive mode, I added the following line in `/etc/vsftpd.conf`:

`pasv_enable=NO`

In the Route Server, say route.mydomain.com, I ran the following commands sequentially:
    modprobe nf_conntrack_ftp
    modprobe nf_nat_ftp

    sysctl net.ipv4.ip_forward=1
    iptables -t nat -A PREROUTING -p tcp --dport 20 -j DNAT --to-   destination 192.168.100.11:20
    iptables -t nat -A PREROUTING -p tcp --dport 21 -j DNAT --to-   destination 192.168.100.11:21
    iptables -t nat -A POSTROUTING -j MASQUERADE

Now, had to make iptables rules persistent, otherwise iptables configuration would disappear after system reboot.

So, I saved firewall rules to a file
`sudo sh -c "iptables-save > /etc/iptables.rules"`

Then, I added `pre-up iptables-restore < /etc/iptables.rules` in the `/etc/network/interfaces` configuration file to apply the rules automatically as this:

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
