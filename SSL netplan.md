# Introduction

We recently experienced disconnects due to missing routing rules required by stunnel.

The root cause was that our custom ip rule and ip route entries were lost whenever systemd-networkd reconfigured the interfaces. This can happen during normal events such as:
* **interface state changes (e.g. DHCP renewal, link flap)**
* running `netplan apply`
* `systemctl restart systemd-networkd`
* a system reboot
  
Because Netplan dynamically generates `.network` files under `/run/systemd/network/` and `systemd-networkd reloads them, any rules added manually with `ip ..` would disappear after such reconfigurations.

# systemd-networkd
`systemd-networkd` natively supports both policy routing rules and Type=local routes.
To make the rules always present, create a dedicated .network file for the loopback interface:

`/etc/systemd/network/10-lo.network`

```
[Match]
Name=lo

[Network]
# No addresses needed here

# IPv4 policy rule: from 127.0.0.0/8, incoming on lo → use table 123
[RoutingPolicyRule]
From=127.0.0.0/8
Table=123
Priority=1000

# IPv6 policy rule: from ::1/128, incoming on lo → use table 123
[RoutingPolicyRule]
From=::1/128
Table=123
Priority=1000

# Local catch-all in table 123 (equivalent to: ip route add local 0.0.0.0/0 dev lo table 123)
[Route]
Type=local
Destination=0.0.0.0/0
Table=123

# IPv6 equivalent (equivalent to: ip -6 route add local ::/0 dev lo table 123)
[Route]
Type=local
Destination=::/0
Table=123
```

## Applying the configuration
```
chmod 644 /etc/systemd/network/10-lo.network
systemctl daemon-reexec
networkctl reload
networkctl reconfigure lo
```

These will be automatically applied every time the loopback interface is configured, such as during boot, after a systemd-networkd reload, or after a netplan apply.

# systemd timer
systemd timer that runs every minute and re-applies routing rules/local routes if necessary.

## 1. The script (does the actual work)
`/usr/local/sbin/stunnel-rules-apply`

```
#!/bin/bash
set -euo pipefail
IP=/usr/sbin/ip

# Add rules; ignore errors if they already exist
$IP rule add from 127.0.0.0/8 iif lo table 123 2>/dev/null || true
$IP -6 rule add from ::1/128      iif lo table 123 2>/dev/null || true

# Add local routes; ignore errors if they already exist
$IP route add local 0.0.0.0/0 dev lo table 123 2>/dev/null || true
$IP -6 route add local ::/0      dev lo table 123 2>/dev/null || true
```

Make it executable:
```
chmod +x /usr/local/sbin/stunnel-rules-apply
```

## 2. The service (runs the script once)
`/etc/systemd/system/stunnel-rules.service`

```
[Unit]
Description=Ensure stunnel policy routing is present
After=network-online.target systemd-networkd.service
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/stunnel-rules-apply
```

## 3. The timer (runs the service every minute)
`/etc/systemd/system/stunnel-rules-verify.timer`

```
[Unit]
Description=Run stunnel-rules.service every minute

[Timer]
OnBootSec=1min
OnUnitActiveSec=1min
AccuracySec=10s
Persistent=true
Unit=stunnel-rules.service

[Install]
WantedBy=timers.target
```

## 4. Enable and start
```
systemctl daemon-reload
systemctl enable --now stunnel-rules-verify.timer
```

## 5. Verify it works
* Check if the timer is scheduled:
```
systemctl list-timers --all | grep stunnel-rules
```
* Check recent service runs:
```
journalctl -u stunnel-rules.service -S -5m
```
* Confirm rules exist in the kernel:
```
ip rule show | grep 123
ip -6 rule show | grep 123
ip route show table 123
ip -6 route show table 123
```

# networkd-dispatcher
```
apt-get install networkd-dispatcher
```

/etc/networkd-dispatcher/routable.d/50-stunnel-rules

```
#!/bin/bash
ip rule add from 127.0.0.1/8 iif lo table 123 2>/dev/null || true
ip route add local 0.0.0.0/0 dev lo table 123
ip -6 rule add from ::1/128 iif lo table 123 2>/dev/null || true
ip -6 route add local ::/0 dev lo table 123
```

```
chmod +x /etc/networkd-dispatcher/routable.d/50-stunnel-rules
ln -s ../routable.d/50-stunnel-rules /etc/networkd-dispatcher/carrier.d/50-stunnel-rules
ln -s ../routable.d/50-stunnel-rules /etc/networkd-dispatcher/degraded.d/50-stunnel-rules
systemctl restart networkd-dispatcher
systemctl enable networkd-dispatcher.service
systemctl start networkd-dispatcher.service
```
