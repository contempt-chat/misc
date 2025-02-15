In this example the interface is `ens3`.

```
apt-get install networkd-dispatcher
```

/etc/networkd-dispatcher/routable.d/50-ifup-hooks

```
#!/bin/bash
echo "START: $IFACE `date`" >> /root/logging_50-ifup-hooks
[ "$IFACE" = "ens3" ] || exit 0
ip rule add from 127.0.0.1/8 iif lo table 123
ip route add local 0.0.0.0/0 dev lo table 123
ip -6 rule add from ::1/128 iif lo table 123
ip -6 route add local ::/0 dev lo table 123
echo "END: $IFACE `date`" >> /root/logging_50-ifup-hooks
```

```
chmod +x /etc/networkd-dispatcher/routable.d/50-ifup-hooks
systemctl enable networkd-dispatcher.service
systemctl start networkd-dispatcher.service
```
