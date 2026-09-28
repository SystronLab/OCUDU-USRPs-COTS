# OCUDU with USRP x310 and COTS UEs

Guide to setup OCUDU with USRP x310 and Google Pixel

## USRP Setup

https://github.com/uoy-research/srsRAN-USRPs-OTA?tab=readme-ov-file#steps-for-setting-up-usrp-x310

## SIM Setup

#### Install Dependencies

```bash
sudo apt-get install pcscd pcsc-tools libccid libpcsclite-dev python3-pyscard
```

#### Connect SIM Card Reader

1. Connect your SIM card reader to your computer.
2. Insert a programmable SIM card into the reader.

Check the status of the connection:

```bash
pcsc_scan
```

If the SIM card reader is recognized, you should see **"Card inserted"**.

---

The SIM programming section on the OCUDU website is outdated as of **23 Apr 2025**.

Use the following repository to program the SIM:

- [pysim Repository](https://gitea.osmocom.org/sim-card/pysim)

```bash
cd pysim
```

---

#### SIM Programming Commands

1. Check the current ISIM configuration:

   ```bash
   ./pySim-read.py -p0
   ```

2. Program the SIM (replace values accordingly):

   ```bash
   ./pySim-prog.py -p0 -s (ICCID) --mcc=001 --mnc=01 -a (ADM-sent to you) --imsi=(IMSI)
   ```

   New IMSI will start with MCC and MNC
   **Example:**

   ```bash
   ./pySim-prog.py -p0 -s 8949440000001343753 --mcc=001 --mnc=01 -a 79375942 --imsi=001010000134375
   ```

- As soon as the programming is done, the **Ki** and **OPC** will be generated automatically. Please note them down as they will be needed while inserting info in the 5g core.

#### SUCI Configuration via pySIM-shell

```text
./pySim-shell.py -p0
pySIM-shell (MF)> select MF
pySIM-shell (MF)> select ADF.USIM
pySIM-shell (MF/ADF.USIM)> select EF.UST
pySIM-shell (MF)> verify_adm <ADM-KEY>
pySIM-shell (MF/ADF.USIM/EF.UST)> ust_service_deactivate 124
pySIM-shell (MF/ADF.USIM/EF.UST)> ust_service_deactivate 125
```

## Pixel Phone Setup

- MCC and MNC in the APN on the phone are auto-filled based on the first 5 digits of the IMSI.
- The OCUDU website uses OnePlus 8T which connects better with roaming (PLMN: 90170).
- For Pixel phones, roaming hacks are usually unnecessary. Use PLMN **00101** (MCC+MNC), and the IMSI must start with **00101**.

1. Enable developer mode: Tap **Build Number** multiple times in phone settings.
2. Open dialer and enter `*#*#4636#*#*`, then set **Preferred Network Type** to **NR only**.
3. The phone can see signal without SIM registration in the 5G core.

If the Pixel device keeps disconnecting:

1. Dial `*#*#0702#*#*`
2. Adjust timers:
   - Set `NR_TIMER_WAIT_IMS_REGISTRATION` to `-1` (infinite timeout)
   - Set `SUPPORT_IMS_NR_REGISTRATION_TIMER` to `0` (disable timeout)

These settings are **SIM-specific** and persist across reboots.

SIM info

<img src="assets/sim_settings.png" alt="Alt text" width="400"/>

Access Point

<img src="assets/access_point_1.png" alt="Alt text" width="400"/>
<img src="assets/access_point_2.png" alt="Alt text" width="400"/>

---

### Connectivity Test

Once the phone is connected:

- Use **Termux app** on the phone:
  - **Uplink Test**: `ping 10.45.0.1`
- From the 5G core:
  - **Downlink Test**: `ping 10.45.1.2`

## Prerequisites 

If you get buffer warning while running gNB, run:

```text
sudo ip link set ens7f0 mtu 9000

sudo sysctl -w net.core.rmem_max=24912805
sudo sysctl -w net.core.wmem_max=24912805
sudo sysctl -w net.core.rmem_default=24912805
sudo sysctl -w net.core.wmem_default=24912805
net.core.rmem_max = 24912805
net.core.wmem_max = 24912805
net.core.rmem_default = 24912805
net.core.wmem_default = 24912805
```

If don't have Internet access on the UE after connection, run:

```text
sudo ip tuntap add name ogstun mode tun
sudo ip addr add 10.45.0.1/16 dev ogstun
sudo ip addr add 2001:db8:cafe::1/48 dev ogstun
sudo ip link set ogstun up
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
sudo ip6tables -t nat -A POSTROUTING -s 2001:db8:cafe::/48 ! -o ogstun -j MASQUERADE
sudo ufw disable
sudo iptables -I INPUT -i ogstun -j ACCEPT
```
