# BFI Acquisition Tutorial

<div align="left">

[![Purple Mountain Laboratories](https://img.shields.io/badge/Purple_Mountain_Laboratories-PML-7030A0?style=for-the-badge)](https://www.pmlabs.com.cn/)
[![SEU School of Cyber Security](https://img.shields.io/badge/Southeast_University-School_of_Cyber_Science_and_Engineering-005A3C?style=for-the-badge)](https://cyber.seu.edu.cn/)
[![SEU National Mobile Communications Lab](https://img.shields.io/badge/Southeast_University-National_Mobile_Communications_Research_Lab-005A3C?style=for-the-badge)](https://ncrl.seu.edu.cn/)
</div>
<br>
This tutorial is specially supported by the "Five-Color Stone Project" of Purple Mountain Laboratories.

---

## 0. Prerequisites 🛠️

Before starting, you must ensure that your hardware meets the **802.11ax** collection requirements; otherwise, you will not be able to capture HE format frames.

1.  **Sniffer Adapter:**
    *   Standard Wi-Fi adapters (like TL-WN722N or other 11n/ac cards) **cannot capture** 802.11ax packets.
    *   You must use an adapter that **supports 802.11ax**, paired with a Linux driver that supports Monitor Mode.
2.  **Experimental Equipment:**
    *   **AP (Beamformer):** A router supporting Wi-Fi 6 (802.11ax) (e.g., TP-LINK BE7200).
    *   **STA (Beamformee):** A mobile phone (Android is highly recommended, avoid Apple if possible) or a laptop supporting Wi-Fi 6.
    *   **Control Terminal:** A computer with Kali Linux installed (used to run the sniffer tool and iperf server).

---

## I. Experimental Topology & Connection 🌐

To maximize the induction of Beamforming, we need to establish a high-throughput **Downlink Traffic** stream between the AP and the STA. Because only when the AP needs to send a massive amount of data to the STA will it be motivated to initiate Channel Sounding to optimize the downlink.

### Connection Steps:

1.  **Configure AP:**
    *   Log into the router's backend (TP-LINK's admin URL is usually `tplogin.cn`).
    *   **Disable "Smart Connect" (Dual-band integration)** and turn off the 2.4GHz band.
    *   **Fix the Channel:** It is recommended to choose a 5GHz channel with less interference, such as **36, 44, 149, or 153**. Do not set it to "Auto". Setting it uniformly to **153** is recommended.
    *   **Fix the Bandwidth:** Set to **80MHz** (most commonly used for experiments, easier to analyze).
    *   Ensure **"MU-MIMO"** and **"OFDMA"** options are enabled.
2.  **Connect STA:**
    *   Connect the Wi-Fi 6 phone/laptop to the AP's 5GHz band.
    *   Confirm in the management interface that the device is connected via Wi-Fi 6 mode, and **record the STA's MAC address**.
3.  **Connect the Sniffer (Kali):**
    *   Plug the USB Wi-Fi adapter into the computer and connect it to the Kali virtual machine.
    *   Ensure Kali is also connected to the same LAN via a wired connection or another adapter (to run the iperf3 server).

---

## II. Traffic Induction (Triggering BFM) 🚀

In the Wi-Fi 6 (802.11ax) protocol, Beamforming Feedback is an **"on-demand"** underlying mechanism. Only when the router (AP) needs to send massive data to the target device (STA) and faces a complex channel environment will it initiate Channel Sounding to optimize the downlink.

Therefore, we need to artificially create extreme **downlink network pressure** to trigger the generation of BFM matrices.

### Plan A: Use iperf3 for Continuous High-Frequency Flooding (Highly Recommended ⭐)

`iperf3` can generate extreme throughput data streams and is the best tool for triggering BFM at high frequencies. To meet the experimental needs for both Single-User (SU-MIMO) and Multi-User (MU-MIMO), please follow these steps:

#### 1. Start Server
On the PC or Kali VM connected to the same LAN via Ethernet, open a terminal and start the iperf3 server, specifying listening port `5201`:
```bash
iperf3 -s -p 5201
```
*(Note: If you need to test dual-device concurrency, it is recommended to open two independent server instances listening on different ports, e.g., open another terminal and run `iperf3 -s -p 5202`)*

#### 2. Start Client for Single Device Communication (Trigger SU-MIMO)
On your Wi-Fi 6 receiver device (e.g., phone running Termux/Magic iPerf, or laptop terminal), execute the following "unlimited downlink flooding" command:
```bash
iperf3 -c <Target_Host_IP> -p 5201 -u -t 0 -b 0 -R
```

**🔑 Core Parameters Breakdown:**
*   `-c <Target_Host_IP>`: Run in client mode and connect to the Server IP you just started.
*   `-p 5201`: Specify communication port as 5201.
*   `-u`: Use **UDP** protocol. UDP has no TCP congestion control and can squeeze bandwidth to the absolute limit, making it perfect for triggering beamforming.
*   `-t 0`: Set transmission time to **infinite (continuous running)** until manually stopped with `Ctrl+C`.
*   `-b 0`: Remove bandwidth limits (force AP to transmit at maximum capacity).
*   `-R`: **Reverse mode**. This step is **crucial**! It reverses the default uplink traffic into **downlink traffic** (Server sends to Client), forcing the AP to actively compute beamforming, thereby inducing the STA to reply with BFM inclusion matrices.

#### 3. Advanced: Dual-Device Concurrent Communication (Trigger MU-MIMO)
If you need to collect overlapped MU-MIMO BFM data in a complex environment, you need two Wi-Fi 6 devices requesting heavy traffic from the AP simultaneously.
1. **Server** runs two ports simultaneously: `iperf3 -s -p 5201` and `iperf3 -s -p 5202`.
2. **Device A (STA 1)** connects to port 5201:
   ```bash
   iperf3 -c <Target_Host_IP> -p 5201 -u -t 0 -b 0 -R
   ```
3. **Device B (STA 2)** simultaneously connects to port 5202:
   ```bash
   iperf3 -c <Target_Host_IP> -p 5202 -u -t 0 -b 0 -R
   ```
At this point, to satisfy the high-bandwidth downlink demands of both devices simultaneously, the router (AP) will trigger high-frequency MU-MIMO channel sounding, and you will capture massive concurrent BFM data on the sniffer side.

---

### Plan B: Use Ping for Keep-Alive and Basic Testing

*⚠️ Note: Ping sends extremely small ICMP packets. Routers usually send these using omnidirectional antennas and rarely bother calculating complex beamforming matrices. This method is mainly used for initial connectivity verification or preventing device sleep.*

**Operation Steps:**

1. **Get Target IP:**
   * Log into the router's admin backend (e.g., TP-LINK default is `tplogin.cn`).
   * Go to "Device Management" or "DHCP Client List".
   * Find your experimental target device (Wi-Fi 6 phone or laptop), view and record its currently assigned LAN IP address (e.g., `192.168.1.105`).

2. **Execute Continuous Ping Test:**
   * On another Windows computer within the LAN, press `Win + X` to open **Windows PowerShell** (or CMD).
   * Enter the following command to continuously send packets to the target host (the `-t` parameter means continuous sending until `Ctrl+C` is pressed):
   ```powershell
   ping <Target_Host_IP> -t
   # Example: ping 192.168.1.105 -t
   ```
3. **Observe Results:** As long as the terminal continuously returns `Reply from 192.168.1.xxx: bytes=32 time=...`, it means the wireless link connectivity is normal. You can switch back to `iperf3` at any time to start formal high-frequency induction sniffing.

---

## III. Kali Linux Adapter Configuration & Monitoring 💻

Now let's configure the sniffing environment.

1.  **Open terminal and check adapter:**
    ```bash
    sudo ifconfig
    # Assume the adapter name is wlan0
    ```
2.  **Set Monitor Mode & Channel:**
    *Note: It is recommended to use the `iw` command, as `iwconfig` has poor support for 802.11ax.*

    ```bash
    # 1. Bring the interface down
    sudo ip link set dev wlan0 down

    # 2. Set to Monitor mode
    sudo iw wlan0 set type monitor

    # 3. Bring the interface up
    sudo ip link set dev wlan0 up

    # 4. Set channel and bandwidth (Crucial step)
    # Syntax: sudo iw dev <device> set channel <channel> <bandwidth_type>
    sudo iw dev wlan0 set channel 153 80MHz
    ```
    *Verify settings: Type `iw dev` to check if the current channel and mode are active.*

---

## IV. Capturing BFM Frames with Tshark 🎣

Start capturing before the iperf3 traffic transmission begins.

**Command Optimization:** To prevent the capture file from becoming too large, we usually only focus on "Action Frames" because BFM is contained within them.

```bash
# Capture and save to file
sudo tshark -i wlan0 -w /root/he_bfm_capture.pcap

# Or use real-time line-buffering mode
sudo tshark -l -i wlan0 -w /root/he_bfm_capture.pcap
```

### Execution Sequence:
1. Phone is ready to run the `iperf3 ... -R` traffic command.
2. Kali runs the `tshark` command.
3. Phone starts generating traffic.
4. After about 60 seconds, press `Ctrl+C` in the Kali terminal to stop capturing.

---

## V. Data Filtering & Analysis (Wireshark) 🔍

This is the most critical step: extracting Wi-Fi 6 Beamforming reports from the massive data.

1.  **Open the file:**
    ```bash
    wireshark /root/he_bfm_capture.pcap
    ```
2.  **Apply Display Filter:**
    *   **Core Filter (for HE-MIMO BFM):**
        `wlan.he.action.he_mimo_control`
    *   **Combined Filter (to see the complete interaction, including NDPA announcement):**
        `(wlan.fc.type_subtype == 21) || (wlan.he.action.he_mimo_control)`

3.  **Identify the correct BFM frame:**
    In the filtered results, look for frames sent by the **STA (Phone) to the AP (Router)**:
    *   **Protocol Column:** Usually displays `802.11` or `HE Action`.
    *   **Info Column:** Will show `Action frame, Category: HE`.
    *   **Expand Frame Structure:**
        1. Expand `IEEE 802.11 Wireless Management`
        2. Look for `HE Action code: HE Compressed Beamforming And CQI (0)`
        3. Expand the `HE MIMO Control` field.
        4. Expand `Feedback Matrices`, which is the targeted **BFM matrix information**.

4.  **Interpret HE MIMO Control Data:**
    *   **Nc Index / Nr Index:** Antenna stream configuration.
    *   **Channel Width:** (0=20, 1=40, 2=80MHz...).
    *   **Codebook Size:** Feedback precision (e.g., 4/2 bit or 7/5 bit).
    *   **Beamforming Report:** The long string of hexadecimal data following this is the compressed $\phi$ and $\psi$ angle matrices.

---

## VI. Troubleshooting ❓

1.  **Cannot capture packets / All packets are gibberish (FCS Error):**
    *   **Distance Issue:** The position of the sniffer is extremely critical! AP signals are strong and easy to catch, but **BFM is sent by the STA**. If your adapter is too far from the phone, you won't receive the feedback frames.
    *   **Suggestion:** Place the sniffing adapter **close to the phone (within 1 meter)**.

2.  **Cannot see HE Action / BFM payload:**
    *   **Encryption Issue:** In WPA2/WPA3 networks, some Action Frames are encrypted (Protected Action Frame).
    *   **Solution:**
        *   **Method 1 (Recommended):** Set the router to an **OWE (Enhanced Open)** or completely **Open (no password)** network for the experiment.
        *   **Method 2:** If you must use WPA, you need to input the Wi-Fi password in Wireshark's protocol settings and completely capture the **4-way handshake** when the phone connects.

3.  **tshark reports "Operation not supported":**
    *   This usually means your adapter driver does not support Monitor mode or does not support 80MHz bandwidth settings. Please verify your adapter chipset model.

---

## Summary: BFM Capture & Filter Quick Reference 📝

| Step | Key Command / Action |
| :--- | :--- |
| **Set Adapter** | `sudo iw dev wlan0 set channel 153 80MHz` |
| **Trigger Traffic** | `iperf3 -c <Server_IP> -R` (Downlink is key) |
| **VHT Filter** | `wlan.vht.mimo_control` (For Wi-Fi 5) |
| **HE Filter** | `wlan.he.action.he_mimo_control` (For Wi-Fi 6) |
| **Complete Interaction** | `wlan.fc.type_subtype == 21` |
