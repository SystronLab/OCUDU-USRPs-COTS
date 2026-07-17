# Dockerised Open5GS and gNB Setup Instructions

## 1. Add Configuration Files

Place the following files in the `open5gs` directory:

- `open5gs-5gc.yml`: This is the Docker Compose configuration file for running the Open5GS 5G Core.
- `open5gs.env`: This file contains subscriber-specific parameters such as:
  - `IMSI`
  - `K`
  - `OPc`

## 2. Start the Open5GS Container

Navigate to the `open5gs` directory and start the container.

## 3. Start the gNB

Once the Open5GS container is running successfully, you can start the gNB using the provided gNB configuration file.



# Enabling UE Internet Access with Dockerized Open5GS

## Overview

The UE successfully registered with the 5G Core and obtained an IP address, but initially had **no Internet access**.

The issue was caused by **three separate problems**:

1. A stale `ogstun` interface on the host from a previous native Open5GS installation.
2. Missing host routing/NAT configuration.
3. Open5GS advertising Google's public DNS servers (`8.8.8.8`, `8.8.4.4`), which are blocked on the University of York network.

After fixing these issues, the UE had full Internet connectivity.

---

## Step 1. Verify UE Registration

Confirm that the UE has successfully registered and received an IP address.

Example:

```text
UE IP: 10.45.0.3
```

If the UE does not obtain an IP address, resolve the registration or PDU session establishment issue before proceeding.

---

## Step 2. Verify the Host Internet Interface

Check which interface the host uses to access the Internet.

```bash
ip route get 8.8.8.8
```

Example:

```text
8.8.8.8 via 144.32.194.1 dev enp1s0
```

In this setup, the Internet-facing interface is:

```text
enp1s0
```

---

## Step 3. Check for an Existing Host `ogstun` Interface

If Open5GS was previously installed natively, the host may already contain an `ogstun` interface.

Check:

```bash
ip addr show ogstun
```

Then verify how packets are routed to the UE subnet:

```bash
ip route get 10.45.0.3
```

If the output is:

```text
10.45.0.3 dev ogstun
```

then the host is routing traffic to the wrong interface.

---

## Step 4. Remove the Old Host `ogstun`

Delete the stale host interface:

```bash
sudo ip link delete ogstun
```

Verify:

```bash
ip addr show ogstun
```

Expected:

```text
Device "ogstun" does not exist.
```

The Docker container's `ogstun` interface is unaffected.

---

## Step 5. Verify the Container `ogstun`

Check inside the Open5GS container:

```bash
sudo docker exec open5gs_5gc ip addr show ogstun
```

Expected:

```text
state UP
10.45.0.1/24
10.45.1.1/24
...
10.45.255.1/24
```

---

## Step 6. Find the Open5GS Container IP Address

Obtain the IP address assigned to the Open5GS container.

```bash
sudo docker inspect -f \
'{{range $name,$net := .NetworkSettings.Networks}}{{$name}}: IP={{$net.IPAddress}} Gateway={{$net.Gateway}}{{println}}{{end}}' \
open5gs_5gc
```

Example:

```text
docker_ran: IP=10.53.1.2 Gateway=10.53.1.1
```

---

## Step 7. Route UE Traffic Back to the Container

Configure the host route:

```bash
sudo ip route replace 10.45.0.0/16 via 10.53.1.2
```

Verify:

```bash
ip route get 10.45.0.3
```

Expected:

```text
10.45.0.3 via 10.53.1.2 dev br-af168a081efc
```

---

## Step 8. Enable IPv4 Forwarding

Enable forwarding on the host:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Enable forwarding inside the container:

```bash
sudo docker exec open5gs_5gc \
sysctl -w net.ipv4.ip_forward=1
```

Expected:

```text
net.ipv4.ip_forward = 1
```

---

## Step 9. Configure NAT

Remove duplicate NAT rules:

```bash
while sudo iptables -t nat -C POSTROUTING \
-s 10.45.0.0/16 ! -o ogstun -j MASQUERADE 2>/dev/null
do
    sudo iptables -t nat -D POSTROUTING \
    -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
done
```

Add a single NAT rule:

```bash
sudo iptables -t nat -A POSTROUTING \
-s 10.45.0.0/16 \
-o enp1s0 \
-j MASQUERADE
```

---

## Step 10. Configure Forwarding Rules

Allow UE traffic to leave the host:

```bash
sudo iptables -I DOCKER-USER 1 \
-s 10.45.0.0/16 \
-o enp1s0 \
-j ACCEPT
```

Allow return traffic:

```bash
sudo iptables -I DOCKER-USER 1 \
-d 10.45.0.0/16 \
-i enp1s0 \
-m conntrack --ctstate ESTABLISHED,RELATED \
-j ACCEPT
```

Verify:

```bash
sudo iptables -t nat -L POSTROUTING -n -v
```

Expected:

```text
MASQUERADE 10.45.0.0/16 -> enp1s0
```

---

## Step 11. Verify Container Internet Access

Check that the Open5GS container can reach the Internet.

```bash
sudo docker exec open5gs_5gc ping -c 3 8.8.8.8
```

This should succeed.

---

## Step 12. Test UE Connectivity

On the UE:

```bash
ping 1.1.1.1
```

If this succeeds but

```bash
ping google.com
```

fails, the remaining issue is DNS.

---

## Step 13. Check Whether Public DNS Is Reachable

On the host:

```bash
dig @8.8.8.8 google.com
```

On the University of York network this timed out:

```text
communications error to 8.8.8.8#53: timed out
```

Therefore Google's public DNS servers cannot be used.

---

## Step 14. Determine the Correct DNS Servers

Check the DNS servers configured on the host.

```bash
resolvectl status enp1s0
```

Example:

```text
DNS Servers:
144.32.128.242
144.32.128.243
```

Verify that they respond:

```bash
dig @144.32.128.242 google.com
```

Expected:

```text
status: NOERROR
```

---

## Step 15. Configure DNS in Open5GS

Edit `open5gs-5gc.yml`.

Replace:

```yaml
smf:
  dns:
    - 8.8.8.8
    - 8.8.4.4
```

with

```yaml
smf:
  dns:
    - 144.32.128.242
    - 144.32.128.243
```

Restart Open5GS:

```bash
sudo docker restart open5gs_5gc
```

Reconnect the UE so that it receives the updated DNS configuration.

---

## Final Verification

The following should all work:

```bash
ping 1.1.1.1
```

```bash
ping google.com
```

Open a web browser on the UE and verify that normal Internet connectivity is available.

---

# Root Cause Summary

The lack of Internet connectivity was caused by three independent issues:

1. **Host routing conflict:** A leftover `ogstun` interface from a previous native Open5GS installation intercepted return traffic intended for the Docker container.

2. **Host routing/NAT:** The Docker container required an explicit route for the UE subnet (`10.45.0.0/16`) and host NAT rules to forward UE traffic to the Internet.

3. **DNS configuration:** Open5GS advertised Google's public DNS servers (`8.8.8.8` and `8.8.4.4`), but the University of York network blocks direct queries to those servers. Replacing them with the university DNS servers (`144.32.128.242` and `144.32.128.243`) resolved the issue.
