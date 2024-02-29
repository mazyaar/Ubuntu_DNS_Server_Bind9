

# DNS SERVER WITH BIND9

***Introduction***

An important part of managing server configuration and infrastructure involves maintaining a way to find network interfaces and IP addresses by name. One way to do this is to set up a proper Domain Name System (DNS). Using fully qualified domain names (FQDNs), instead of IP addresses, to specify network addresses optimizes the configuration of services and applications, and increases the maintainability of configuration files. Setting up your own DNS for your private network is a great way to improve the management of your servers.

In this tutorial, you will set up an internal DNS server using two Ubuntu 20.04 servers. You will use the BIND name server software (BIND9) to resolve private hostnames and private IP addresses. This provides a central way to manage your internal hostnames and private IP addresses, which is indispensable when your environment expands to more than a few hosts.


***Prerequisites***

To complete this tutorial, you will need the following infrastructure. Be sure to create each server in the same datacenter private networking enabled:

* A fresh Ubuntu 20.04 server to serve as the Primary DNS server, ns1.
* (Recommended) A second Ubuntu 20.04 server to serve as a Secondary DNS server, ns2.
* At least one additional server. This guide assumes you have two additional servers, which will be referred to as client servers. These client servers must be created in the same datacenter where your DNS servers are located.


***Example Infrastructure and Goals***

For the purposes of this article, we will assume the following:

* You have two servers which will be designated as your DNS name servers. This guide will refer to these as ns1 and ns2.
* You have two additional client servers that will be using the DNS infrastructure you create, referred to as host1 and host2 in this guide. You can add as many client servers as you’d like.
* All of these servers exist in the same datacenter. This tutorial assumes that this datacenter is called nyc3.
* All of these servers have private networking enabled and are on the 10.128.0.0/16 subnet (you will likely have to adjust this for your servers).
* All servers are connected to a project that runs on example.com. This guide outlines how to set up an internal, private DNS system, so you can use any domain name you’d like instead of example.com. The DNS servers will always attempt to first route requests internally, meaning they won’t try to reach the given domain on the public internet. However, using a domain you own may help avoid conflicts with publicly routable domains.

With these assumptions in mind, the examples in this guide will use a naming scheme based around the subdomain nyc3.example.com to refer to the example private subnet or zone. Therefore, host1’s private Fully-Qualified Domain Name (FQDN) will be host1.nyc3.example.com. The following table holds the relevant details used in examples throughout this guide:



| Host  | Role   | Private FGDN   | Private IP Address   |
|---|---|---|---|
| **ns1** | Primary DNS Server | `ns1.nyc3.example.com` | `10.128.10.11` |
| **ns2** | Secondary DNS Server | `ns2.nyc3.example.com` | `10.128.20.12` |
| **host1** | Generic Host 1 | `host1.nyc3.example.com` | `10.128.100.101` |
| **host2** | Generic Host 2 | `host2.nyc3.example.com` | `10.128.200.102` |


> Note: Your setup will be different, but the example names and IP addresses will be used to demonstrate how to configure a DNS server to provide a functioning internal DNS. You should be able to adapt this setup to your own environment by replacing the host names and private IP addresses with your own. It is not necessary to use the region name of the datacenter in your naming scheme, but we use it here to denote that these hosts belong to a particular datacenter’s private network. If you run servers in multiple datacenters, you can set up an internal DNS within each respective datacenter.



## Step 1- Update System
```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
```
***

## Step 2 – Install DNS package
```bash
sudo apt-get install bind9
sudo apt-get install dnsutils
```

```bash
sudo nano /etc/default/named
```
* _Add ''-4'' end of the ''OPTIONS'' parameter:_

```
OPTIONS="-u bind -4"
```

```bash
sudo systemctl restart bind9
```
***

## step 3 - Configure forwarders on /etc/bind/named.conf.options:


On ns1, open the named.conf.options file for editing:
```bash
sudo nano /etc/bind/named.conf.options
```

> Above the existing options block, create a new ACL (access control list) block called trusted. This is where you will define a list of clients from which you will allow recursive DNS queries (i.e. your servers that are in the same datacenter as ns1). Add the following lines to add ns1, ns2, host1, and host2 to your list of trusted clients, being sure to replace the example private IP addresses with those of your own servers:

```
acl "trusted" {
        10.128.10.11;    # ns1 
        10.128.20.12;    # ns2
        10.128.100.101;  # host1
        10.128.200.102;  # host2
        10.128.0.0/16	# Range IPs
};


options {
        directory "/var/cache/bind";
        recursion yes;                 # enables recursive queries
        allow-recursion { trusted; };  # allows recursive queries from "trusted" clients
        allow-query { trusted; localhost; internal-network; 192.168.0.0/23; localnets; };
        listen-on { 10.128.10.0/24; };   # ns1 private IP address - listen on private network only
        allow-transfer { DNS1 IP; };      # zone transfers
     

acl "trusted" {
        10.10.0.0/16;
        localnets;
        localhost;
};
      
options {
       directory "/var/cache/bind";
               
forwarders {
8.8.8.8;
4.2.2.4;
};


	auth-nxdomain no;
        dnssec-validation yes;
        listen-on-v6 { any; };

};

```

## step 4 Configuring the Local File:

edit ``/etc/bind/named.conf.local`` and define db:
```bash
nano /etc/bind/named.conf.local
```

```
zone "nyc3.example.com" {
    type primary;
    file "/etc/bind/zones/db.nyc3.example.com"; # zone file path
    allow-transfer { 10.128.20.12; };           # ns2 private IP address - secondary
};
```

***Assuming that our private subnet is 10.128.0.0/16, add the reverse zone by with the following lines (note that our reverse zone name starts with 128.10 which is the octet reversal of 10.128):***

```
zone "128.10.in-addr.arpa" {
    type primary;
    file "/etc/bind/zones/db.10.128";  # 10.128.0.0/16 subnet
    allow-transfer { 10.128.20.12; };  # ns2 private IP address - secondary
};
```

_or_ 

```bash
sudo vim /etc/bind/named.conf.local
```

***Add the following block to it:***

```
zone “example.com” {
type master;
file “/etc/bind/db.example.com”;
};
```

## Step 4: Creating the Forward Zone File
```bash
sudo mkdir /etc/bind/zones
```

* We will base our example forward zone file on the sample db.local zone file. Copy it to the proper location with the following commands:

```bash
sudo cp /etc/bind/db.local /etc/bind/zones/db.nyc3.example.com
```

* Now edit your forward zone file:

```bash
sudo nano /etc/bind/zones/db.nyc3.example.com
```

```
$TTL    604800
@       IN      SOA     ns1.nyc3.example.com. admin.nyc3.example.com. (
                  3     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
;
; name servers - NS records
IN      NS      ns1.nyc3.example.com.
IN      NS      ns2.nyc3.example.com.

; name servers - A records
ns1.nyc3.example.com.          IN      A       10.128.10.11
ns2.nyc3.example.com.          IN      A       10.128.20.12

; 10.128.0.0/16 - A records
host1.nyc3.example.com.        IN      A      10.128.100.101
host2.nyc3.example.com.        IN      A      10.128.200.102
```
***

## Step 5 - Creating the Reverse Zone File(s):

```bash
sudo cp /etc/bind/db.127 /etc/bind/zones/db.10.128
```
```bash
sudo nano /etc/bind/zones/db.10.128
```
```
$TTL    604800
@       IN      SOA     nyc3.example.com. admin.nyc3.example.com. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
; name servers
      IN      NS      ns1.nyc3.example.com.
      IN      NS      ns2.nyc3.example.com.

; PTR Records
11.10   IN      PTR     ns1.nyc3.example.com.    ; 10.128.10.11
12.20   IN      PTR     ns2.nyc3.example.com.    ; 10.128.20.12
101.100 IN      PTR     host1.nyc3.example.com.  ; 10.128.100.101
102.200 IN      PTR     host2.nyc3.example.com.  ; 10.128.200.102
```

## Step 6 – Primary Master

```bash
sudo vim /etc/bind/named.conf
```

```
include “/etc/bind/named.conf.options”;
include “/etc/bind/named.conf.local”;
include “/etc/bind/named.conf.default-zones”;
```

## Step 7 - Checking the BIND Configuration Syntax

```bash
sudo systemctl restart bind9
```
```bash
dig google.com
```
```bash
sudo named-checkconf /etc/bind/named.conf
```
```bash
sudo named-checkzone nyc3.example.com /etc/bind/zones/db.nyc3.example.com
```
```
Output
zone nyc3.example.com/IN: loaded serial 3
OK
```

## Step 8 - Restarting BIND

```bash
sudo systemctl restart bind9
```
```bash
sudo ufw allow Bind9
```


***

# Step 9 — Configuring the Secondary DNS Server

```bash
sudo nano /etc/bind/named.conf.options
```

```
        recursion yes;
        allow-recursion { trusted; };
        listen-on { 10.128.20.12; };      # ns2 private IP address
        allow-transfer { none; };          # disable zone transfers by default

        forwarders {
                8.8.8.8;
                8.8.4.4;
        };
```

```bash
sudo nano /etc/bind/named.conf.local
```

```
zone "nyc3.example.com" {
    type slave;
    file "db.nyc3.example.com";
    masters { 10.128.10.11; };  # ns1 private IP
};

zone "128.10.in-addr.arpa" {
    type slave;
    file "db.10.128";
    masters { 10.128.10.11; };  # ns1 private IP
};
```

```bash
sudo named-checkconf
```

```bash
sudo systemctl restart bind9
```

```bash
sudo ufw allow Bind9
```


## Step 10 — Testing Clients

***Forward Lookup***

```bash
nslookup host1
```

```
Output
Server:     127.0.0.53
Address:    127.0.0.53#53

Non-authoritative answer:
Name:   host1.nyc3.example.com
Address: 10.128.100.101
```

***Reverse Lookup***
```bash
nslookup 10.128.100.101
```
```
Output
11.10.128.10.in-addr.arpa   name = host1.nyc3.example.com.

Authoritative answers can be found from:
```

## Step 11 — Maintaining DNS Records

Now that you have a working internal DNS, you need to maintain your DNS records so they accurately reflect your server environment.
Adding a Host to DNS

Whenever you add a host to your environment (in the same datacenter), you will want to add it to DNS. Here is a list of steps that you need to take:
Primary Name Server

    Forward zone file: Add an A record for the new host, increment the value of Serial
    Reverse zone file: Add a PTR record for the new host, increment the value of Serial
    Add your new host’s private IP address to the trusted ACL (named.conf.options)

Test your configuration files:

```bash
sudo named-checkconf
sudo named-checkzone nyc3.example.com /etc/bind/zones/db.nyc3.example.com
sudo named-checkzone 128.10.in-addr.arpa /etc/bind/zones/db.10.128
```

Then reload BIND:
```bash
sudo systemctl reload bind9
```

Your primary server should be configured for the new host now.


Secondary Name Server

    Add your new host’s private IP address to the trusted ACL (named.conf.options)

Check the configuration syntax:

```bash
sudo named-checkconf
```

***Then reload BIND:***

```bash
sudo systemctl reload bind9
```

Your secondary server will now accept connections from the new host.
Configure New Host to Use Your DNS

* Configure /etc/resolv.conf to use your DNS servers

* Test using nslookup

Removing a Host from DNS

If you remove a host from your environment or want to just take it out of DNS, just remove all the things that were added when you added the server to DNS (i.e. the reverse of the previous steps).

* _Source_:
![Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-ubuntu-20-04#configuring-the-local-file)

![Source 2](https://www.digitalocean.com/community/tutorials/how-to-configure-bind-as-a-private-network-dns-server-on-ubuntu-22-04)











