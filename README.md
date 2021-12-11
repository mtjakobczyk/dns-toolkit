# DNS Toolkit

**Domain Name System (DNS)** is a hierarchical name service.  
DNS acts as a directory of network resources (mainly hosts) and maps hostnames to IP addresses.

Worldwide there are 13 root zone (`.`) clusters of servers: https://www.iana.org/domains/root/servers

We categorize DNS queries into two types:
- **Forward DNS Lookup** returns an IP address for a given name.
- **Reverse DNS Lookup** returns a name for a given IP address.

A DNS Record 
DNS Record types:
- **NS (name server) record** identifies an authoritative DNS server for a zone
- **SOA (start of authority) record** stores information about DNS zone(s)

## Clients
On Linux, the nameservers for quering are defined in `/etc/resolv.conf`. When you run `nslookup`, the utility will query the server(s) defined in the `/etc/resolv.conf`.

## DNS Servers (nameservers)
We differentiate two types of DNS servers:
1. **Authoritative Server** is able to answer queries using its own database without asking other servers.
2. **Non-authoritative Servers** (aka Caching Servers) answer queries using either its local cache or by quering other servers (listed in the local `/etc/resolv.conf`). The responses from other servers are cached for a given (TTL) time.

Examples of available nameservers:
- BIND
- dnsmasq
- djbdns (open-sourced in 2001

### BIND
On Linux, the most common DNS software is **BIND** (`bind` package, `named` program). The configuration is stored in `/etc/named.conf`.

#### Zone File
Zone file is a configuration that describes a DNS zone.



#### RNDC
It is possible to control BIND DNS server remotely over TCP/IP using RNDC (Remote Name Daemon Control). The `rndc` is a **name server control utility**. 

The `rndc` and the BIND server must share a secret (RNDC key).
```bash
# RNDC configuration
# part of /etc/rndc.conf
key "rndc-key" {
	algorithm hmac-md5;
	secret "LKoeFOqgzwjSmnFHdY4DQg==";
};
```
```bash
# BIND configuration
# part of /etc/named.conf
key "rndc-key" {
        algorithm hmac-md5;
        secret "LKoeFOqgzwjSmnFHdY4DQg==";
};

controls {
        inet 127.0.0.1 port 953
                allow { 127.0.0.1; } keys { "rndc-key"; };
};
```

BIND must be configured to accept control commands on TCP port.

