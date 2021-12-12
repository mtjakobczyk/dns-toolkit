# DNS Toolkit

**Domain Name System (DNS)** is a hierarchical name service.  
DNS acts as a directory of network resources (mainly hosts) and maps hostnames to IP addresses.

Worldwide there are 13 root zone (`.`) clusters of servers: https://www.iana.org/domains/root/servers

## Clients
On Linux, the `bind-utils` package ships three programs: `nslookup`,`dig` and `host`. 

The `dig +trace  pogoda.wp.pl` can be used to show the entire DNS resolution process, starting from querying the `.` root zone nameservers, going through the `.com` TLD nameservers, to receiving the list of nameservers for the `wp.pl` zone. In the last step, the zone-specific nameserver will finally return the `A` record for `pogoda.wp.pl`.

You can ask for specific DNS records like this: `dig NS wp.pl` or `nslookup -type=NS wp.pl` or `host -t ns wp.pl`

On Linux, the default nameservers to query for DNS names are defined in `/etc/resolv.conf`.

We categorize DNS queries into two types:
- **Forward DNS Lookup** returns an IP address for a given name.
- **Reverse DNS Lookup** returns a name for a given IP address.

## DNS Record types

### SOA (start of authority)
 The **SOA record** defines authoritative information about a zone:
```bash
# dig SOA +multiline wp.pl
wp.pl.	3592 IN	SOA ns1.wp.pl. dnsmaster.wp-sa.pl. (
		2021120802 ; serial
		900        ; refresh (15 minutes)
		600        ; retry (10 minutes)
		86400      ; expire (1 day)
		3600       ; minimum (1 hour)
		)
```
- The `ns1.wp.pl.` entry defines the zone's primary name server
- The `dnsmaster.wp-sa.pl.` entry means that `dnsmaster@wp-sa.pl` is the e-mail address to the zone admin.
- The **refresh** entry defines how often the secondary server queries the primary server for update
- The **serial** entry must be incremented every time a change is made to the zone file on the primary server. The next time the secondary server queries the primary, it detects that a change was made to the zone file. In this way the secondary knows when to transfer that zone.

### NS (name server)
The **NS record** identifies an authoritative DNS server for a zone.

```bash
# dig NS +multiline wp.pl
wp.pl.			2018 IN	NS ns2.wp.pl.
wp.pl.			2018 IN	NS ns1.task.gda.pl.
wp.pl.			2018 IN	NS ns1.wp.pl.
```

### A (IPv4 address)
The **A record** identifies an IPv4 address for a given domain name.

```bash
# dig A +multiline pogoda.wp.pl
pogoda.wp.pl.		3600 IN	A 212.77.100.133
```

### CNAME 
The **CNAME record** maps one domain name (considered as an alias) to another domain name (considered as canonical name).

```bash
# dig m.wp.pl
m.wp.pl.		600	IN	CNAME	rd.wp.pl.
rd.wp.pl.		1460	IN	A	212.77.100.83
```


## DNS Servers (nameservers)
We differentiate two types of DNS servers:
1. **Authoritative Server** is able to answer queries using its own database without asking other servers.
2. **Non-authoritative Servers** (aka Caching Servers) answer queries using either its local cache or by quering other servers (listed in the local `/etc/resolv.conf`). The responses from other servers are cached for a given (TTL) time.

Examples of available nameservers:
- BIND (worldwide standard for Linux)
- CoreDNS (used on Kubernetes)
- F5 BIG-IP DNS
- Microsoft DNS
- dnsmasq
- djbdns

### BIND
On Linux, the most common DNS software is **BIND** (`bind` package, `named` program). The configuration is stored in `/etc/named.conf`.

#### Zone File
Zone file is a configuration that describes a DNS zone. 

In `/etc/named.conf` you declare a zone and then reference the relevant zone file:
```bash
# part of /etc/named.conf
zone "mylabserver.com" {
   type master;
   file "/var/named/fwd.mylabserver.com.db";
   # additional settings
};
```

The zone files are stored in `/var/named`.

You define the zone in the zone file, starting always with `TTL` and `SOA` record:
```bash
# start of /var/named/fwd.mylabserver.com.db
$TTL    86400
@       IN      SOA     ns1.mylabserver.com. root.mylabserver.com. (
                          10030         ; Serial
                           3600         ; Refresh
                           1800         ; Retry
                         604800         ; Expiry
                          86400         ; Minimum TTL
)
# ... other entries
```

:bulb: Plain-text zone files can be compiled to their binary representations using the `named-compilezone` program. This makes service queries faster.


#### High availability
All name servers are set using `NS` records in the zone file.
```bash
# part of a zone file
@       IN      NS     ns1.mylabserver.com.
@       IN      NS     ns2.mylabserver.com.
```
There can be only one primary nameserver (`type master`). On the other hand, there can be many secondaries (`type slave`). 

:zap: If the primary nameserver fails, you can reconfigure one of the secondary nameservers to become the new primary. You would also need to reconfigure all secondaries to respect the new primary. All the configuration is done in the `/etc/named.conf` files.

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

