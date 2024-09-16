# SSL
Setup:
ircd listens on:
* 127.0.0.1 6679 and 6697
* ::1 6679 and ::1 6697

stunnel listens on:
* <your-external-ipv4> 6679 and 6697
* <your-external-ipv6> 6679 and 6697
.. and tunnels traffic to the local ports above

In this example IPv4 is `45.141.0.18` and IPv6 is `2001:470:60a1::1`.

We will use apache to request and renewal the certificate. After the initial setup you can stop apache. It will be restarted automatically during the renewal process (see below).

Upgrade your ircd to 2.11.3p3+ircnet2-1.0.8 or higher.

```
apt-get install certbot python3-certbot-apache apache2
mkdir /var/www/irc.warszawa.pl
```

Append to /etc/apache2/apache2.conf
```
<VirtualHost *:80>
  ServerName irc.warszawa.pl
  DocumentRoot /var/www/irc.warszawa.pl
</VirtualHost>
```

```
apachectl reload
certbot --apache
```

Request a certificate for irc.warszawa.pl
```
certbot --apache
```

Now there should be a valid certificate at /etc/letsencrypt/live/irc.warszawa.pl/

Create bundle.pem
```
cat /etc/letsencrypt/live/irc.warszawa.pl/fullchain.pem /etc/letsencrypt/live/irc.warszawa.pl/privkey.pem > /etc/letsencrypt/live/irc.warszawa.pl/bundle.pem 
```

conf.P-Lines or ircd.conf
```
# SSL
P|127.0.0.1|||6679||T|::ffff:45.141.0.18|
P|127.0.0.1|||6697||T|::ffff:45.141.0.18|
P|::1|||6679||T|2001:470:60a1::1|
P|::1|||6697||T|2001:470:60a1::1|
```

/etc/network/if-up.d/stunnel
```
#!/bin/bash
[ "$IFACE" = "lo" ] || exit 0
ip rule add from 127.0.0.1/8 iif lo table 123
ip route add local 0.0.0.0/0 dev lo table 123
ip -6 rule add from ::1/128 iif lo table 123
ip -6 route add local ::/0 dev lo table 123
```
```
chmod +x /etc/network/if-up.d/stunnel
ifup -v -f lo
```

Install and configure stunnel:
```
apt-get install stunnel4
```

/etc/stunnel/stunnel.conf
```
pid = /var/run/stunnel.pid
output = /var/log/stunnel4/stunnel.log
foreground = no
fips = no
socket = l:TCP_NODELAY=1
socket = r:TCP_NODELAY=1

[ircd_6679]
accept  = 45.141.0.18:6679
connect = 127.0.0.1:6679
cert = /etc/letsencrypt/live/irc.warszawa.pl/bundle.pem
transparent = source

[ircd_6697]
accept  = 45.141.0.18:6697
connect = 127.0.0.1:6697
cert = /etc/letsencrypt/live/irc.warszawa.pl/bundle.pem
transparent = source

[ircd_6679_ipv6]
accept = 2001:470:60a1::1:6679
connect = ::1:6679
cert = /etc/letsencrypt/live/irc.warszawa.pl/bundle.pem
transparent = source

[ircd_6697_ipv6]
accept = 2001:470:60a1::1:6697
connect = ::1:6697
cert = /etc/letsencrypt/live/irc.warszawa.pl/bundle.pem
transparent = source
```

```
systemctl start stunnel4.service
```

Test each port
```
openssl s_client -4 -connect irc.warszawa.pl:6679
openssl s_client -4 -connect irc.warszawa.pl:6697
openssl s_client -6 -connect irc.warszawa.pl:6679
openssl s_client -6 -connect irc.warszawa.pl:6697
```
You should get:
```
:irc.warszawa.pl 020 * :Please wait while we process your connection.
```

## Renew certificates
/etc/letsencrypt/renewal-hooks/pre/before-renewal.sh
```
#!/bin/bash
systemctl start apache2
```

/etc/letsencrypt/renewal-hooks/post/after-renewal.sh
```
#!/bin/bash
# If you started apache only for certificate renewal, you can stop it now.
systemctl stop apache2
cat /etc/letsencrypt/live/irc.warszawa.pl/fullchain.pem /etc/letsencrypt/live/irc.warszawa.pl/privkey.pem > /etc/letsencrypt/live/irc.warszawa.pl/bundle.pem
kill -1 `cat /var/run/stunnel.pid`
```

```
chmod +x /etc/letsencrypt/renewal-hooks/pre/before-renewal.sh /etc/letsencrypt/renewal-hooks/post/after-renewal.sh
```

Crontab for root:
```
@daily certbot renew --apache > /dev/null 2>&1
```

## Link over SSL
Append to stunnel.conf:
```
[hub_contempt_chat]
client = yes
accept = 127.0.0.1:6696
connect = <hub-ip>:6697
local = 45.141.0.18
```

Restart stunnel
Test connection:
```
$ nc 127.0.0.1 6696
:hub.contempt.chat 020 * :Please wait while we process your connection.
```

Change C/N Lines to connect to 127.0.0.1 6696
```
c|127.0.0.1|xxxxxxxxxxx|hub.contempt.chat|6696|1000||
N|127.0.0.1|xxxxxxxxxxx|hub.contempt.chat||1000|

```
