# 🦈 Wireshark Network Traffic Analysis & Packet Inspection Lab

A deep packet inspection (DPI) and network protocol analysis lab conducted within an isolated, host-only virtual network environment. This project documents the reproduction, capture process, and packet-level dissection across 7 core networking and security modules using **Wireshark**.

---

## 📌 Lab Topology & Environment Setup

* **Kali Linux (Attacker / Sensor Node):** `192.168.137.135` (MAC: `00:0c:29:0e:29:66`)
* **Windows 10 (Target Host):** `192.168.137.134` (MAC: `00:0c:29:16:77:30`)
* **Virtual Gateway / Switch:** `192.168.137.1`
* **Network Interface:** `eth1` (VMware Host-Only segment)

```text
+-----------------------+              Host-Only Virtual Network              +-------------------------+
|      Kali Linux       |                     (VMnet1)                        |       Windows 10        |
|  [Attacker / Sensor]  | <=================================================> |     [Target Server]     |
|   IP: 192.168.x.10    |                                                     |    IP: 192.168.x.20     |
+-----------------------+                                                     +-------------------------+
---
```
## 🔬 Lab Modules & Reproduction Guide

### 1. Packet Capture Basics
* **Objective:** Verify interface binding to the Host-Only adapter (`eth1`), identify frame encapsulation, and validate traffic distribution across Wireshark's three primary panes.
* **How Traffic Was Generated:**
  1. Opened Wireshark on Kali Linux and selected interface `eth1`.
  2. Booted the Windows 10 VM to generate initial operating system discovery and ARP broadcasts across the virtual switch.
* **Display Filter:** `frame`
* **Dissection & Observations:**
  * **Packet List (Top):** Indexed real-time inter-VM packets, displaying frame numbers, timestamps, Layer 3/4 addressing, and protocol classifications.
  * **Packet Details (Middle):** Parsed encapsulated headers across the OSI stack: Frame $\rightarrow$ Ethernet II $\rightarrow$ IPv4 $\rightarrow$ Upper Layer Protocols.
  * **Packet Bytes (Bottom):** Displayed raw binary payload data synchronized in hexadecimal offsets and ASCII translations.

<img width="1438" height="708" alt="Screenshot 2026-09-22 183827" src="https://github.com/user-attachments/assets/dfc5a1bd-e3f6-4798-91bf-ee84349ec1ee" />


---

### 2. ICMP (Internet Control Message Protocol)
* **Objective:** Dissect Layer 2 hardware address resolution and verify bidirectional Layer 3 reachability between Kali and Windows 10.
* **How Traffic Was Generated:**
  * Executed a bidirectional ping test between both virtual machines:
    * From Kali Terminal: `ping -c 4 192.168.137.134`
    * From Windows CMD: `ping 192.168.137.135`
* **Display Filter:** `icmp`
* **Dissection & Observations:**
  * **ARP Resolution:** Prior to initial ping delivery, an ARP broadcast resolved the physical MAC addresses between both endpoints.
  * **Type 8 (Echo Request):** Initiated by Windows with a standard Windows base `TTL = 128`.
  * **Type 0 (Echo Reply):** Returned by Kali Linux with a standard Linux base `TTL = 64`.

<img width="1441" height="538" alt="Screenshot 2026-09-22 183904" src="https://github.com/user-attachments/assets/51f2635e-7044-4e01-a85b-31d8e9853780" />


---

### 3. DNS (Domain Name System)
* **Objective:** Trace client hostname resolution queries and examine DNS packet structures over UDP port 53.
* **How Traffic Was Generated:**
  * Monitored automatic background operating system telemetry queries initiated by Windows 10 attempting to contact Microsoft endpoints upon network connection.
* **Display Filter:** `dns`
* **Dissection & Observations:**
  * **Standard Query (Flags: 0x0100):** Observed Type A query frames directed to the virtual DNS resolver at `192.168.137.1`.
  * **Network Behavior:** Queries displayed retransmissions due to the non-routed, internet-isolated nature of the Host-Only network segment.

<img width="1438" height="699" alt="Screenshot 2026-09-22 183933" src="https://github.com/user-attachments/assets/eab63a38-4ce1-4bfa-8a34-97c6385cea98" />


---

### 4. TCP 3-Way Handshake
* **Objective:** Capture stateful transport-layer session initialization (`SYN` $\rightarrow$ `SYN-ACK` $\rightarrow$ `ACK`) on port 80.
* **How Traffic Was Generated:**
  1. On Kali Linux, allowed incoming connections and started an HTTP server:
     ```bash
     sudo iptables -F
     sudo python3 -m http.server 80 --bind 192.168.137.135
     ```
  2. On Windows 10, initiated an HTTP web request via browser / curl:
     ```cmd
     curl [http://192.168.137.135](http://192.168.137.135)
     ```
* **Display Filter:** `tcp.port == 80`
* **Dissection & Observations:**
  * **Step 1 (`SYN`):** Windows (`192.168.137.134:53193`) initiates with `Seq=0`, MSS (`1460`), and window scaling parameters.
  * **Step 2 (`SYN, ACK`):** Kali server acknowledges client sequence (`Ack=1`) and sends its synchronization sequence (`Seq=0`).
  * **Step 3 (`ACK`):** Windows returns the final acknowledgment (`Ack=1`), moving the socket into an `ESTABLISHED` state.

<img width="1443" height="727" alt="Screenshot 2026-09-22 191259" src="https://github.com/user-attachments/assets/1bd09e86-efed-49cd-9467-cedd721884a2" />


---

### 5. HTTP vs. HTTPS (Cleartext Payload Inspection)
* **Objective:** Demonstrate the vulnerability of unencrypted HTTP transactions to cleartext inspection.
* **How Traffic Was Generated:**
  * Captured the application layer payload resulting from the browser visit to `http://192.168.137.135`.
* **Display Filter:** `http`
* **Dissection & Observations:**
  * **Stream Reassembly:** Reconstructed via **Follow $\rightarrow$ HTTP Stream** to extract the raw conversation.
  * **Cleartext Exposure:** Request headers (`GET / HTTP/1.1`, `User-Agent: Mozilla/5.0... Edg/92.0`), server banners (`SimpleHTTP/0.6 Python/3.14.7`), and the full HTML document markup were fully readable over the wire.

<img width="1444" height="714" alt="Screenshot 2026-09-22 191156" src="https://github.com/user-attachments/assets/8d80128e-0eb8-46b3-b8a5-2b3914e5523d" />

<img width="949" height="750" alt="Screenshot 2026-09-22 191438" src="https://github.com/user-attachments/assets/530c7a6e-2373-499e-b6a0-1ec97a437a3d" />


---

### 6. FTP Credential Sniffing (Plaintext Authentication)
* **Objective:** Intercept and expose plaintext authentication credentials transmitted over legacy transfer protocols.
* **How Traffic Was Generated:**
  1. On Kali Linux, installed and started an FTP service on port 21:
     ```bash
     sudo apt install -y python3-pyftpdlib
     sudo python3 -m pyftpdlib -p 21
     ```
  2. On Windows 10 CMD, initiated an FTP authentication session:
     ```cmd
     ftp 192.168.137.135
     ```
     * Entered Username: `admin`
     * Entered Password: `SuperSecretPassword123`
* **Display Filter:** `tcp.stream eq 1` (or `ftp`)
* **Dissection & Observations:**
  * **Stream Reassembly:** Inspected via **Follow $\rightarrow$ TCP Stream**.
  * **Credential Leakage:** Extracted unencrypted `USER admin` and `PASS SuperSecretPassword123` commands transmitted directly across Layer 7.

<img width="1449" height="259" alt="Screenshot 2026-09-22 191852" src="https://github.com/user-attachments/assets/327952b9-9c78-4557-be72-90e63020dd1e" />
<img width="1450" height="334" alt="Screenshot 2026-09-22 193031" src="https://github.com/user-attachments/assets/c86ee663-62cc-4177-8092-80a0ba359dc5" />
<img width="751" height="348" alt="Screenshot 2026-09-22 193042" src="https://github.com/user-attachments/assets/05561bbe-2c0c-4ba2-aa6f-e918436f8c27" />


---

### 7. Attack Analysis: Port Scan & Reconnaissance Detection
* **Objective:** Identify, isolate, and fingerprint active TCP stealth reconnaissance scans generated from Kali targeting the Windows node.
* **How Traffic Was Generated:**
  * Executed a TCP SYN stealth scan from Kali against the target Windows VM:
    ```bash
    sudo nmap -sS -p 21,80,135,445,3389 192.168.137.134
    ```
* **Display Filter:** `tcp.flags.syn == 1 || tcp.flags.reset == 1`
* **Dissection & Observations:**
  * **SYN Stealth Scan Pattern (`-sS`):** Rapid half-open `[SYN]` packets sent across ports 21, 3389, 80, 445, and 135 within milliseconds.
  * **Closed Port Behavior:** Windows immediately sent `[RST, ACK]` packets to terminate requests on closed ports (21, 80, 3389).
  * **Open Port Behavior:** Windows responded with `[SYN, ACK]` on listening services (135, 445), which Kali immediately severed with an `[RST]` packet to avoid completing the full handshake.

<img width="524" height="289" alt="Screenshot 2026-09-22 193511" src="https://github.com/user-attachments/assets/00e78666-7791-4cff-9989-9659c2817668" />
<img width="1455" height="370" alt="Screenshot 2026-09-22 193531" src="https://github.com/user-attachments/assets/4d9a3c3f-cb84-4d79-aad6-ba462e825e61" />

---

## 🎯 Key Takeaways & Defensive Recommendations

1. **Enforce Cryptographic Protocols:** Unencrypted protocols such as standard HTTP and FTP expose critical credentials and payloads to trivial packet sniffing. Secure equivalents (HTTPS, SFTP, SSH) must be enforced.
2. **TCP State Machine Diagnostics:** Monitoring control flags (`SYN`, `ACK`, `RST`, `FIN`) is critical for identifying session terminations, connection resets, and firewall drops.
3. **Reconnaissance Detection:** Automated port scanners generate distinct traffic signatures—namely rapid bursts of half-open SYN packets across multiple destination ports—which should be leveraged to trigger IDS/IPS blocking rules.

---

