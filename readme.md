# Bind-DNS-how-To

## What is DNS

![DNS](https://github.com/okorsi/Bind-DNS-how-To/blob/main/screenshots/what-is-dns-server.png?raw=true)

### Install Bind DNS Server

First of all we need to install bind9 on our Ubuntu server:
```
apt-get install bind9 bind9utils bind9-dnsutils bind9-doc bind9-host -y
```
Bind 9 service is managed by systemd. We can start the Bind DNS service and enable it to start at system reboot using the following command:
````
systemctl start named
systemctl enable named
````
check the status of the Bind service
````
systemctl status named
````
we get the following result:
````
● named.service - BIND Domain Name Server
     Loaded: loaded (/lib/systemd/system/named.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2022-05-26 21:38:44 UTC; 1min 14s ago
       Docs: man:named(8)
   Main PID: 90096 (named)
      Tasks: 6 (limit: 2188)
     Memory: 7.5M
        CPU: 316ms
     CGroup: /system.slice/named.service
             └─90096 /usr/sbin/named -u bind

Jul 19 21:38:44 ubuntus named[90096]: network unreachable resolving './DNSKEY/IN': 2001:500:2f::f#53
Jul 19 21:38:44 ubuntus named[90096]: network unreachable resolving './NS/IN': 2001:500:2f::f#53
Jul 19 21:38:44 ubuntus named[90096]: network unreachable resolving './DNSKEY/IN': 2001:500:a8::e#53
Jul 19 21:38:44 ubuntus named[90096]: network unreachable resolving './NS/IN': 2001:500:a8::e#53
Jul 19 21:38:44 ubuntus named[90096]: network unreachable resolving './DNSKEY/IN': 2001:500:1::53#53
Jul 19 21:38:44 ubuntus named[90096]: network unreachable resolving './NS/IN': 2001:500:1::53#53
````
Bind DNS server’s configuration files are located inside /etc/bind directory
````
root@ubuntus:/home/oleksii# ls /etc/bind
bind.keys  db.127  db.empty  named.conf                named.conf.local    rndc.key
db.0       db.255  db.local  named.conf.default-zones  named.conf.options  zones.rfc1918
root@ubuntus:/home/oleksii#
````
Need to edit /etc/bind/named.conf.options file and add forwarders. DNS query will be forwarded to the forwarders when your local DNS server is unable to resolve the query
Uncomment and change the following lines:
````
forwarders {
       8.8.8.8;
};
````
Edit the /etc/bind/named.conf.local file to define the zone for our domain.
````
zone "okorsi.com" {
 type master;
 file "/etc/bind/db.local";
};
zone "0.16.172.in-addr.arpa" {
 type master;
 file "/etc/bind/db.172";
};
````
A brief explanation of above file is shown below:

* okorsi.com is our forward zone.
* 0.16.172.in-addr.arpa is our reverse zone.
* db.local is the name of the forward lookup zone file.
* db.172 is the name of the reverse lookup zone file.

Then, verify the configuration file for any error using the following command:
````
named-checkconf
````
if we have not received any information - then everything is fine

## Configure Forward and Reverse Lookup Zone

![DNS](https://github.com/okorsi/Bind-DNS-how-To/blob/main/screenshots/forward-and-reverse-zones.png?raw=true)

Edit the forward lookup zone file:
````
nano /etc/bind/db.local
````
Make the following changes:
````
$TTL    604800
@       IN      SOA     ubuntus.okorsi.com. root.ubuntus.okorsi.com. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
@          IN      NS      ubuntus.okorsi.com.
ubuntus    IN      A       172.16.0.10
web1       IN      A       172.16.0.20
web2       IN      A       172.16.0.30
@          IN      AAAA    ::1
````
Where:

* 172.16.0.10: IP address of DNS server.
* 172.16.0.20: IP address of WEB1 server.
* 172.16.0.20: IP address of WEB2 server.
* NS: Name server record.
* A: Address record.
* SOA: Start of authority record.

Edit the reverse lookup zone file:
````
nano /etc/bind/db.172
````
Make the following changes:
````
$TTL    604800
@       IN      SOA     ubuntus.okorsi.com. root.ubuntus.okorsi.com. (
                              1
                         604800
                          86400
                        2419200
                         604800 )
@       IN      NS      ubuntus.okorsi.com.
ubuntus    IN      A       172.16.0.10
web1       IN      A       172.16.0.20
web2       IN      A       172.16.0.30
10       IN      PTR     ubuntus.okorsi.com.
````
Edit the /etc/resolv.conf file and define our DNS server:
````
nano /etc/resolv.conf
````
Add the following lines:
````
search okorsi.com
nameserver 172.16.0.10
````
restart the Bind DNS service to apply the changes:
```
systemctl restart named
````
Next, check the forward and reverse lookup zone file for any syntax error with the following command:
````
named-checkzone forward.okorsi db.local
````
If everything is fine. We should see the following output:
````
zone forward.okorsi/IN: loaded serial 4
OK
`````
To check the reverse lookup zone file, run the following command:
````
named-checkzone reverse.okorsi db.172
````
If everything is fine. We should see the following output:
````
If everything is fine. You should see the following output:
````

## Verify Bind DNS Server

First, run the dig command against our DNS nameserver:
````
dig ubuntus.okorsi.com
````
We should see the following output:
````
; <<>> DiG 9.16.1-Ubuntu <<>> ubuntus.okorsi.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29810
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: a5919d692166f92001000000627498b926c5a27d5f9c28a1 (good)
;; QUESTION SECTION:
;ubuntus.okorsi.com.	IN	A

;; ANSWER SECTION:
ubuntus.okorsi.com. 604800	IN	A	172.16.0.10
web1.okorsi.com. 604800 IN A 172.16.0.20
web2.okorsi.com. 604800 IN A 172.16.0.30

;; Query time: 0 msec
;; SERVER: 172.16.0.10#53(172.16.0.10)
;; WHEN: Tue May 26 03:40:41 UTC 2022
;; MSG SIZE  rcvd: 96
````
Now, run the dig command against the DNS server’s IP to perform the reverse lookup query as shown below:
````
dig -x 172.16.0.10
````
We will get the following output:
````
; <<>> DiG 9.16.1-Ubuntu <<>> -x 172.16.0.10
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 55197
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 14afadf1d320160e01000000627498d32b3036329829ce2f (good)
;; QUESTION SECTION:
;10.0.16.172.in-addr.arpa.	IN	PTR

;; ANSWER SECTION:
10.0.16.172.in-addr.arpa. 604800 IN	PTR	ubuntus.okorsi.com.

;; Query time: 0 msec
;; SERVER: 172.16.0.10#53(172.16.0.10)
;; WHEN: Tue May 26 03:41:07 UTC 2022
;; MSG SIZE  rcvd: 118
````
We can also use nslookup command against the DNS server to confirm DNS server name resolution.
````
nslookup ubuntus.okorsi.com
````
We should see name to IP resolution in the following output:
````
Server:		172.16.0.10
Address:	172.16.0.10#53

Name:	ubuntus.okorsi.com
Address: 172.16.0.10
````
Now, run the nslookup command against DNS server IP address to confirm the reverse lookup:
````
nslookup 172.16.0.10
````
We should see the IP address to name resolution in the following output:
````
10.0.16.172.in-addr.arpa	name = ubuntus.okorsi.com.
````







