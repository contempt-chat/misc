# SSL
Upgrade to  or higher

conf.P-Lines or ircd.conf
```
# SSL
P|127.0.0.1|||6679||DT|
P|127.0.0.1|||6697||DT|
P|::1|||6679||DT|
P|::1|||6697||DT|
```

Run these commands after startup
```
ip rule add from 127.0.0.1/8 iif lo table 123
ip route add local 0.0.0.0/0 dev lo table 123

ip -6 rule add from ::1/128 iif lo table 123
ip -6 route add local ::/0 dev lo table 123
```

apt-get install stunnel4

/etc/stunnel/stunnel.conf
```
pid = /var/run/stunnel.pid
output = /var/log/stunnel.log
foreground = no
fips = no
socket = l:TCP_NODELAY=1
socket = r:TCP_NODELAY=1

[ircd_6679]
accept  = 188.165.173.111:6679
connect = 127.0.0.1:6679
cert = /etc/letsencrypt/live/nova.irc.it/bundle.pem
transparent = source

[ircd_6697]
accept  = 188.165.173.111:6697
connect = 127.0.0.1:6697
cert = /etc/letsencrypt/live/nova.irc.it/bundle.pem
transparent = source

[ircd_6679_ipv6]
accept = 2001:41d0:404:200::aa8:6679
connect = ::1:6679
cert = /etc/letsencrypt/live/nova.irc.it/bundle.pem
transparent = source

[ircd_6697_ipv6]
accept = 2001:41d0:404:200::aa8:6697
connect = ::1:6697
cert = /etc/letsencrypt/live/nova.irc.it/bundle.pem
transparent = source

[hub_contempt_chat]
client = yes
accept = 127.0.0.1:6699
connect = 54.38.219.98:6697
local = 188.165.173.111
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
