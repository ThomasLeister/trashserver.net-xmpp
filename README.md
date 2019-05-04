# trashserver.net-xmpp
Repository with copies of my XMPP server configuration for public interest / investigation.

## Notes on this setup

* Database backend: PostgreSQL
* Admin web interface and API are disabled
* In-Band registration and web registration are enabled
* Captchas enabled
* Nginx is used as HTTP / Websocket Proxy
* Port 5223 used as TLS port for "[XMPP over TLS](https://xmpp.org/extensions/xep-0368.html)" feature (Port 443 is forwarded to 5223)


## OS and Ejabberd version

I'm using Debian 9 Stretch as an OS. Ejabberd 18.12 was installed from the stretch-backports repo:

```
nano /etc/apt/sources.list
```

```
# Stretch Backports
deb http://ftp.debian.org/debian stretch-backports main
```

```
apt update
apt install -t stretch-backports ejabberd erlang-p1-pgsql
apt install imagemagick
```

*Make sure you install ```erlang-p1-pgsql``` from the backport repo!*


## TLS certificates and Diffie Hellman parameters

I'm using acme.sh for Let's Encrypt: https://github.com/Neilpang/acme.sh

DH parameters have been generated via:

```
openssl dhparam -out /etc/ejabberd/dh4096.pem 4096
```


## PostgreSQL

Creation and setup of the Postgres database:

```
su -c "psql" - postgres
create user ejabberd with password 'ejabberdpassword';
create database ejabberd_trashserver;
alter database ejabberd_trashserver owner to ejabberd;
\q
su -c "psql ejabberd_trashserver < /usr/share/ejabberd/sql/pg.new.sql" - ejabberd
```


## Nginx HTTP / WS Proxy

Nginx terminates TLS and provides nice URLs without any port notation. It listens on the primary IP address and the host name "xmpp.trashserver.net" as well as the hostname "upload.trashserver.net" for file upload processing.


## HTTP Upload

Ejabberd is using [Prosody Filer](https://github.com/ThomasLeister/prosody-filer) for file upload processing.


## XMPP over TLS

Port 443 of a second IP address (just for that purpose) is forwarded to port 5223 (TLS) on Ejabberd. Necessary iptables rules:

| Origin address (sec)  | Origin port   | Destination address (prim)    | Destination port  |
|-----------------------|---------------|-------------------------------|-------------------|
| 2a01:360:50e:1::252   | 443           | 2a01:360:50e:1::251           | 5223              |
| 5.1.92.252            | 443           | 5.1.92.251                    | 5223              |

IPv6:
```
ip6tables -t nat -A PREROUTING -d 2a01:360:50e:1::252/128 -p tcp -m tcp --dport 443 -j DNAT --to-destination [2a01:360:50e:1::251]:5223
ip6tables -t nat -A POSTROUTING -d 2a01:360:50e:1::252/128 -p tcp -m tcp --dport 5223 -j SNAT --to-source [2a01:360:50e:1::251]:5223
```

IPv4:
```
iptables -t nat -A PREROUTING -d 5.1.92.252/32 -p tcp -m tcp --dport 443 -j DNAT --to-destination 5.1.92.251:5223
iptables -t nat -A POSTROUTING -d 5.1.92.252/32 -p tcp -m tcp --dport 5223 -j SNAT --to-source 5.1.92.251:5223
```

I recommend [iptables-persistent](https://packages.debian.org/stretch/iptables-persistent) to save iptables: https://wiki.debian.org/iptables#Making_Changes_permanent




