# Hyper-V-Lab-Networking-Ubuntu-Internet-Connectivity-Through-Kali
Incident Summary


## Hyper-V Lab Networking — Ubuntu Internet Connectivity Through Kali

### Incident Summary

The Ubuntu fintech VM had a functioning private connection to the Kali security-testing VM through the Hyper-V `PurpleTeam-Lab` internal network, but Ubuntu could not access the Internet.

The intended architecture was:

```text
                         Windows 11 Pro Host
                                │
              ┌─────────────────┴─────────────────┐
              │                                   │
       PurpleTeam-Lab                         Default Switch
        Internal LAN                         Internet/NAT
        10.10.10.0/24
              │
        ┌─────┴─────┐
        │           │
     Ubuntu        Kali
  10.10.10.20   10.10.10.10
                    │
              IP forwarding
                    │
                MASQUERADE
                    │
              Default Switch
                    │
                 Internet
```

The objective was to provide Ubuntu with Internet access **without removing or replacing the existing PurpleTeam-Lab connection** between Ubuntu and Kali.

---

# 1. Reconnaissance

The first stage was to establish the actual network topology rather than immediately changing configuration.

### Hyper-V topology

The Windows host contained:

* `PurpleTeam-Lab` — Internal Hyper-V switch
* `Default Switch` — Hyper-V NAT/Internet connectivity

VM adapters were identified as:

| VM     | Interface | Network        | Address          |
| ------ | --------- | -------------- | ---------------- |
| Ubuntu | `eth0`    | PurpleTeam-Lab | `10.10.10.20/24` |
| Kali   | `eth1`    | PurpleTeam-Lab | `10.10.10.10/24` |
| Kali   | `eth0`    | Default Switch | `172.30.x.x/20`  |

Kali therefore had two network paths:

```text
eth1 → PurpleTeam-Lab → Ubuntu
eth0 → Default Switch → Internet
```

This made Kali a suitable routing/NAT gateway for Ubuntu.

### Ubuntu initial routing

Ubuntu initially had the private route:

```text
10.10.10.0/24 dev eth0
```

but did not have a persistent Internet default route.

The temporary route:

```bash
sudo ip route add default via 10.10.10.10 dev eth0
```

was subsequently used to prove that Kali should be the gateway.

---

# 2. Verification

The next stage verified each network layer independently.

### Ubuntu → Kali

Ubuntu successfully reached Kali:

```bash
ping -c 3 10.10.10.10
```

Result:

```text
3 packets transmitted, 3 received
0% packet loss
```

This established that the `PurpleTeam-Lab` private network was functioning.

### Kali → Internet

Kali independently reached the Internet:

```bash
ping -c 4 8.8.8.8
```

and successfully resolved and reached external hosts such as:

```bash
ping google.com
```

Therefore the Hyper-V `Default Switch` Internet path was functioning.

### Ubuntu → Internet

Ubuntu initially failed:

```bash
ping -c 3 8.8.8.8
```

with:

```text
100% packet loss
```

and:

```bash
ping google.com
```

returned:

```text
Temporary failure in name resolution
```

Because raw IP connectivity to `8.8.8.8` was also failing, DNS was determined not to be the primary problem.

---

# 3. Debugging

The investigation then moved to Kali's routing/NAT layer.

### IP forwarding

Kali initially reported:

```text
net.ipv4.ip_forward = 0
```

This meant Kali was not configured to route packets between Ubuntu and the Internet.

IP forwarding was enabled:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Verification:

```text
net.ipv4.ip_forward = 1
```

### Firewall/NAT inspection

Initially there were no required forwarding/NAT rules.

The required path was established with:

```bash
sudo iptables -A FORWARD \
  -i eth1 -o eth0 \
  -s 10.10.10.0/24 \
  -j ACCEPT

sudo iptables -A FORWARD \
  -i eth0 -o eth1 \
  -d 10.10.10.0/24 \
  -m conntrack --ctstate ESTABLISHED,RELATED \
  -j ACCEPT

sudo iptables -t nat -A POSTROUTING \
  -s 10.10.10.0/24 \
  -o eth0 \
  -j MASQUERADE
```

### Duplicate-rule cleanup

During troubleshooting, the forwarding and NAT commands had been executed multiple times.

The resulting ruleset contained duplicate entries.

The duplicate rules were removed and replaced with one clean forwarding/NAT configuration.

Final relevant rules:

```text
-A FORWARD -s 10.10.10.0/24 -i eth1 -o eth0 -j ACCEPT
-A FORWARD -d 10.10.10.0/24 -i eth0 -o eth1 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A POSTROUTING -s 10.10.10.0/24 -o eth0 -j MASQUERADE
```

---

# 4. Reverse-Path Filtering

Kali's reverse-path filtering configuration was also adjusted for this two-interface routing topology.

Configured:

```bash
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.eth0.rp_filter=0
net.ipv4.conf.eth1.rp_filter=0
```

Verification:

```text
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.eth0.rp_filter = 0
net.ipv4.conf.eth1.rp_filter = 0
```

This removed reverse-path filtering as a potential source of asymmetric-routing problems in the lab router configuration.

---

# 5. Packet-Level Verification

Firewall counters were used to verify that Ubuntu traffic was actually reaching Kali's forwarding path.

Example:

```text
FORWARD
238 packets
42424 bytes
eth1 → eth0
10.10.10.0/24
```

The return rule also received traffic:

```text
33 packets
16068 bytes
eth0 → eth1
RELATED,ESTABLISHED
```

The NAT rule was also hit:

```text
MASQUERADE
10.10.10.0/24 → eth0
```

This demonstrated that Ubuntu traffic was entering Kali, being forwarded toward the Internet, and receiving return traffic.

A packet capture was also used during troubleshooting to distinguish routing/firewall problems from upstream Internet problems.

---

# 6. Successful Connectivity Test

After correcting forwarding and NAT, Ubuntu successfully reached Google's public DNS server:

```bash
ping -c 5 8.8.8.8
```

Result:

```text
5 packets transmitted
5 received
0% packet loss
```

Observed latency:

```text
min/avg/max:
208.804 / 243.507 / 278.394 ms
```

This established successful end-to-end IP connectivity:

```text
Ubuntu
  ↓
PurpleTeam-Lab
  ↓
Kali eth1
  ↓
Kali forwarding
  ↓
Kali MASQUERADE
  ↓
Kali eth0
  ↓
Hyper-V Default Switch
  ↓
Internet
  ↓
8.8.8.8
```

---

# 7. Persistence / Remediation

The initial fix was not sufficient because manually configured runtime settings could disappear after reboot.

The configuration was therefore converted into persistent system configuration.

## Kali — persistent IP forwarding

A dedicated sysctl configuration was created:

```text
/etc/sysctl.d/99-kali-router.conf
```

with:

```text
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.eth0.rp_filter=0
net.ipv4.conf.eth1.rp_filter=0
```

Applied with:

```bash
sudo sysctl --system
```

Verification:

```bash
sysctl net.ipv4.ip_forward
```

Result:

```text
net.ipv4.ip_forward = 1
```

## Kali — persistent firewall/NAT

`iptables-persistent` and `netfilter-persistent` were installed.

Current rules were saved:

```bash
sudo iptables-save | sudo tee /etc/iptables/rules.v4 >/dev/null
```

The persistence service was enabled:

```bash
sudo systemctl enable netfilter-persistent
```

This ensures the forwarding/NAT configuration is restored during system startup.

---

# 8. Ubuntu — Persistent Gateway

The Ubuntu Netplan configuration originally defined the static address:

```text
10.10.10.20/24
```

but did not define a default gateway.

The configuration was changed so that the gateway is part of the persistent network configuration:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      dhcp4: false
      addresses:
        - 10.10.10.20/24
      routes:
        - to: default
          via: 10.10.10.10
      dhcp6: false
```

After applying Netplan, the routing table showed:

```text
default via 10.10.10.10 dev eth0 proto static
10.10.10.0/24 dev eth0 proto kernel scope link src 10.10.10.20
```

The `proto static` default route confirms that the gateway is now supplied by the persistent Netplan configuration rather than a temporary manual route.

---

# 9. Final Configuration

### Ubuntu

```text
eth0:
    10.10.10.20/24

default gateway:
    10.10.10.10
```

### Kali

```text
eth1:
    10.10.10.10/24
    PurpleTeam-Lab

eth0:
    172.30.x.x/20
    Default Switch
```

Routing:

```text
IPv4 forwarding:
    enabled

rp_filter:
    disabled on router paths
```

NAT:

```text
10.10.10.0/24
        ↓
MASQUERADE
        ↓
Kali eth0
```

---

# 10. Result

The original failure was not caused by the PurpleTeam-Lab network itself.

The private Ubuntu ↔ Kali network was functioning correctly.

The missing components were in the **routing/NAT/persistence layer**:

1. Kali IPv4 forwarding was disabled.
2. Kali did not initially have the required forwarding/NAT rules.
3. The Ubuntu default gateway was initially not persistent.
4. Runtime firewall rules needed to be converted into persistent configuration.

The resulting architecture preserves the isolated lab network while providing Ubuntu with Internet access through Kali:

```text
             ┌───────────────────────┐
             │     Windows Host      │
             │       Hyper-V         │
             └──────────┬────────────┘
                        │
              ┌─────────┴─────────┐
              │                   │
       PurpleTeam-Lab        Default Switch
        10.10.10.0/24        Internet/NAT
              │                   │
       ┌──────┴──────┐            │
       │             │            │
    Ubuntu          Kali──────────┘
 10.10.10.20     10.10.10.10
       │             │
       │             ├─ IPv4 forwarding
       │             ├─ FORWARD rules
       │             └─ MASQUERADE
       │
       └──── Internet via Kali
```

### Validation status

| Component                      | Status                                 |
| ------------------------------ | -------------------------------------- |
| Ubuntu → Kali private network  | ✅ Verified                             |
| Kali → Internet                | ✅ Verified                             |
| Kali IPv4 forwarding           | ✅ Enabled                              |
| Kali FORWARD rules             | ✅ Configured                           |
| Kali MASQUERADE                | ✅ Configured                           |
| Ubuntu default gateway         | ✅ Persistent                           |
| Kali firewall/NAT persistence  | ✅ Configured                           |
| Kali sysctl persistence        | ✅ Configured                           |
| Ubuntu → `8.8.8.8`             | ✅ Verified                             |
| Ubuntu DNS hostname resolution | Pending final post-reboot verification |
| Reboot survival test           | Final validation step                  |

### Operational principle

The remediation deliberately preserved the existing private lab topology instead of replacing it with a second Internet-connected adapter on Ubuntu.

This provides a controlled architecture:

> **Ubuntu uses Kali as its Internet gateway while remaining connected to the isolated PurpleTeam-Lab network.**

The configuration is also documented as infrastructure-as-code-style system configuration rather than relying on undocumented temporary commands.
