# 🦈 Wireshark Network Traffic Analysis & Packet Inspection Lab

A hands-on network security project focused on deep packet inspection (DPI), protocol analysis, and traffic troubleshooting using **Wireshark**.

---

## 📌 Project Overview
The objective of this project is to analyze network conversations across the OSI model layers, understand unencrypted vs. encrypted protocol behaviors, dissect TCP connection lifecycles, and use advanced display filters to isolate anomalous or targeted network traffic.

---

## 🛠️ Lab Environment & Tools
* **Tool:** Wireshark
* **OS:** Kali Linux / Windows 11
* **Protocols Analyzed:** ARP, ICMP, DNS, TCP, HTTP, TLS/HTTPS
* **Utilities Used:** `ping`, `curl`, `dig`, `nmap`

---

## 📂 Repository Structure
* `/captures`: Exported packet capture files (`.pcapng`) for each lab exercise.
* `/screenshots`: Annotations showing filter syntax, stream reconstruction, and OSI layer breakdowns.
* `/filters`: Wireshark display and capture filter cheat sheets.

---

## 🔬 Key Lab Exercises

### Lab 1: Interface Inspection & ICMP Analysis
* Captured ICMP Echo Request and Echo Reply packets.
* Inspected Layer 2 (Ethernet MAC headers), Layer 3 (IP Source/Destination, TTL), and Layer 4 packet data.
* **Filter Used:** `icmp && ip.addr == <target_ip>`



### Lab 2: DNS Resolution & Query Dissection
* Traced domain name queries from local resolver to authoritative/recursive DNS servers.
* Analyzed DNS query flags, transaction IDs, and response records (A, AAAA, CNAME).
* **Filter Used:** `dns.flags.response == 0` (queries) and `dns.flags.response == 1` (replies)

### Lab 3: TCP 3-Way Handshake & Teardown
* Tracked the `SYN` $\rightarrow$ `SYN-ACK` $\rightarrow$ `ACK` connection sequence.
* Monitored Relative vs. Absolute Sequence (SEQ) and Acknowledgment (ACK) numbers.
* Inspected graceful termination (`FIN`) versus abrupt termination (`RST`).
* **Filter Used:** `tcp.flags.syn == 1 && tcp.flags.ack == 0`

### Lab 4: Insecure Protocols & Plaintext Credential Exposure
* Captured an unencrypted HTTP POST request and FTP/Telnet authentication.
* Reassembled the communication session using **Follow $\rightarrow$ TCP Stream** to recover credentials in cleartext.
* Compared payload visibility against an encrypted TLS/HTTPS session.
* **Filter Used:** `http.request.method == "POST"`

---

## 🔍 Wireshark Filter Quick Reference

| Task | Display Filter |
| :--- | :--- |
| Filter by Host IP | `ip.addr == 192.168.1.1` |
| Filter by Subnet | `ip.addr == 192.168.1.0/24` |
| Filter by Port | `tcp.port == 80 || udp.port == 53` |
| Show Insecure HTTP Logins | `http.request.method == "POST"` |
| Show TCP SYN Packets Only | `tcp.flags.syn == 1 && tcp.flags.ack == 0` |
| Detect TCP Reset Packets | `tcp.flags.reset == 1` |
| Exclude ARP & Broadcast Traffic | `!arp && !dhcp` |

---

## 🎯 Key Takeaways & Security Insights
1. **Cleartext Vulnerability:** Protocols like HTTP, FTP, and Telnet transmit credentials directly across the wire, exposing them to ARP spoofing and packet sniffing.
2. **TCP State Awareness:** Analyzing sequence and acknowledgment numbers is critical to identifying dropped packets, network retransmissions, and out-of-order packets.
3. **Filter Optimization:** Using targeted display filters reduces noise from thousands of background background packets to isolate relevant traffic in seconds.

---

