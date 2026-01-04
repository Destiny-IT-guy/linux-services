# DNS

What is a DNS server. It is a server that help translate website names into IP address. They work for IPv4 and IPv6.
I Will show you how to install and configure a dns server in linux. I will use bind9 as my dns server.

## installation

First lets install are service and useful apps.

```bash
sudo apt install bind9
sudo apt install dnsutils
```

### Forwarders

Now we will st up our Forwarders. A forwarder is another dns server my dns server will request information when it doesn't know the answer. Let say i want to go on the internet and search for google.com. My dns server doesn't have it in our records so it will ask another dns server.  I will use google and cloudflare as DNS forwarders. Go to the file `/etc/bind/named.conf.options`. Uncomment the forwarders section and add the IP addresses.

```bash
forwarders {
                8.8.8.8;
                1.1.1.1;
        };
```

Then restart bind 9.

```bash
sudo systemctl restart bind9.service
```

### Creating zones

We are going to first create the forwarding zone. A forwarding zones defines the mapping of fully qualified domains names (FQDNs) to IP address in a specific domain name. This allow the server to translate names to IP so client can locate them easily.<br>

Go to the file `/etc/bind/named.conf.local` and add your DNS zone. My domain name for that zone will be victim.net. it should look something like this.

```bash
zone "victim.net" {
        type master;
        file "/etc/bind/db.victim.net";
};
```

Now we copy a template for our zone.

```bash
sudo cp /etc/bind/db.local /etc/bind/db.victim.net
```

The first will be a template of the file and the second is my file.

```bash
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     localhost. root.localhost. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      localhost.
@       IN      A       127.0.0.1
@       IN      AAAA    ::1
```

```bash
;
; BIND data file for victim.net
;
$TTL    604800
@       IN      SOA     victim.net. victim.net. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      dns.victim.net.
@       IN      A       192.168.2.231
dns     IN      A       192.168.2.231
dns2    IN      A       192.168.2.232
```

`@       IN      SOA     localhost. root.localhost.` This is the first line you must change. `localhost.` is the primary dns server for the zone. you have to replace it with the FQDN of your dns server. <br>
`root.localhost.` is the email address of the administrator. root.localhost would be `root@localhost`.</br> Lastly all information between `()` are detail for secondary server on how to handle the zone. You should do a increment of 1 whn you modify this file if you use a secondary server.

Dns record stored information about domain, hostname, mail servers and more. In forward zone. This is a list of main record type you might see in forward zone. The secondary server check the main server based off th refresh timer (in seconds). If the serial is lower than the main it will sync with the main to download new zone. There is a way to force a sync and I will explain it later. 

| type | explanation | example |
|------|-------|------|
| A | Maps a hostname to an IPv4 address | server5.victim.net --> 10.10.10.10 |
| AAAA |  Maps a hostname to an IPv6 address | server5.victim.net --> 2001:db8:85a3::8a2e:370:7334 |
| CNAME | Create an alias for an existing cname. Points to other hostname. | ftp --> serv2.victim.net |
| NS | Name server. specifies hostname of DNS servers. |@ IN ns dns.victim.net |
| SOA | Define primary server and zone info | @ IN SOA dns.victim.net. admin.victim.net. |
| MX | defines mail server | @ IN MX mail.victim.net |
| TXt | Holds text data. | ... |
| SRV | Service record. specifies IP as well as port | ... |

`@` is placeholder for the domain name. `IN` stands for internet. You will almost always See `IN` but there used to be other values that were useful. IF you replace `@` with a value without `.` at the end its acts as a prefix for the domain. so dns will be dns.victim.local. If I leave it blank it will take the value of th record on top.

So i added a record for my domain name, one to give dns server a hostname and 1 for a second server.

After this just restart bind9 services.

### Reverse zone

The DNS reverse zone is used to map IP addresses bact to their hostnames. It can be considered as the opposite of the forward zone. A forward can have multiple reverse zone because reverse zone are identified by their network ID.

The steps of configuring a reverse zone is similar to the forward zone. First wi will need to modify the `/etc/bind/name.conf`. We will add our reverse zone. since we are using /24 and our subnet is 192.168.2.0 we will use the following zone name,

```bash
zone "2.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/db.192";
};
``` 

I would suggest you make a folder in `/bind` for your domain zone to make it mor clean. after this we will copy our template to the reverse zone file.

```bash
sudo cp /etc/bind/db.127 /etc/bind/db.192
```

Now we will modify the file nd add our 2 PTR.

```bash
;
; BIND reverse data file for 192.168.2.xxx net
;
$TTL    604800
@       IN      SOA     dns.victim.net. root.victim.net. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      dns.
@       IN      NS      dns2.
231     IN      PTR     dns.victim.net.
232     IN      PTR     dns2.victim.net.
```

The serial Number increments has to happen here as well. For the pointer you have to add the last octet of the ip address so 192.168.2.231 for /24 would be 231 and you need to add the complete hostname with a `.` at the end.

## Secondary server

After configuring the first server you can configure a secondary server. IT is very useful because it acts like a backup if the first one goes down. You can have multiple secondary dns server. I would also recommend using Keepalive or a reverse proxy if you use multiple dns server. 

First off we need to add the option `allow-transfer` in both zone. your `name.conf.local file should look like this.   

```bash
zone "victim.net" {
        type master;
        file "/etc/bind/db.victim.net";
        allow-transfer { 192.168.2.232; };
};

zone "2.168.192.in-addr.arpa" {
        type master;
        file "/etc/bind/db.192";
        allow-transfer { 192.168.2.232; };
};
```

Restart bind 9 services.

Then install bind9 on your secondary and add these line to its conf.local file.

```bash
zone "victim.net" {
        type secondary;
        file "db.victim.net";
        masters { 192.168.2.231; };
};

zone "2.168.192.in-addr.arpa" {
        type secondary;
        file "db.192";
        masters { 192.168.2.231; };
};
```

Make sure not to write full path for both files.<br>
Then restart bind9 services. 

Then we will go back on the master dns server. we will add a NS record for server 2 adjust the increment add a line to name.conf.

In name.conf you can add this with each zone: `also-notify { 192.168.2.232; };`. This will make synchronization instantaneously after each serial increment.

Now we will add th NS record for server 2. Go back to server 1 and make your bout your zones look like this.

```bash
$TTL    604800
@       IN      SOA     dns.victim.net. admin.victim.net. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      dns.victim.net.
@       IN      NS      dns2.victim.net.

@       IN      A       192.168.2.231
dns     IN      A       192.168.2.231
dns2    IN      A       192.168.2.232
```

```bash
$TTL    604800
@       IN      SOA     dns.victim.net. root.victim.net. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      dns.victim.net.
@       IN      NS      dns2.victim.net.
231     IN      PTR     dns.victim.net.
232     IN      PTR     dns2.victim.net.
```

Now reload bind9 in both server.

If you added `also-notify` you can check the logs in `/var/log/syslog` to check that the transfer of files worked.

## Commands

### named-check

Let first make sure bind9 is running with this command `systemctl status bind9`

There are 2 commands to quickly check your configuration syntax. there is `named-checkconf` and `named-checkzone`. Conf check all bind configuration files for syntax error. If no output appears that means everything is ok.  check zone checks specific files. Here is and expemple of how to used it.

```bash
sudo named-checkzone victim.net /etc/bind/db.victim.net
sudo named-checkzone 2.168.192.in-addr.arpa /etc/bind/db.192
```

### nslookup

This is a tool used to query DNS server. It can retrieve information like domain name, IP address, mail servers and more. 

The following is a command to get domain IP address. `nslookup victim.net`

The following is a command to find domain name from IP address (reverse lookup). `nslookup 192.168.2.231`

The following is to get a query from a specific dns server. i added a record for server with ip of ...233. This is the command and the output.

```bash
nslookup 192.168.2.233 192.168.2.231
233.2.168.192.in-addr.arpa      name = serv3.victim.net.
```

The first value after nslookup is the dns record i am searching and the second value is the dns server it look for the record. I recommend using the IP address of the dns server for second value because you will still be able to query the dns server even if you aren't connected to the domain but connected in the same network.

The following command will allow you to query the server for specific type of record. So you can do a query for A record, NS record ptr etc. here is how they would look like.

```bash
nslookup -type=ns victim.net 192.168.2.231
nslookup -type=a serv3.victim.net 192.168.2.231
nslookup -type=ptr 192.168.2.233 192.168.2.232
```

### Dig

