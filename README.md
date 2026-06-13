# Packet Capture on Kali Linux: Daemonlogger + Wireshark Analysis

A hands-on walkthrough of full packet capture on Kali Linux using **Daemonlogger** — generating real mixed traffic with `ping`, `curl`, `iperf3`, and a browser, then analyzing the `.pcap` file in **Wireshark** to extract HTTP/HTTPS/TCP counts, top conversations, protocol hierarchy, and IO graphs.

📖 Full article on Dev.to: [Packet Capture on Kali Linux: Daemonlogger Setup, Traffic Generation & Wireshark Analysis](https://dev.to/almahmudkhalif/lab-task-10-packet-capture-on-kali-linux-daemonlogger-setup-traffic-generation-wireshark-o7n)

---

## Environment

- **OS:** Kali Linux 2026.1 (Oracle VirtualBox)
- **Network Interface:** `eth0` — IP `10.0.2.15`
- **Tools:** `daemonlogger`, `iperf3`, `curl`, `ping`, Firefox, `wireshark`

---

## Step 1 — Install Daemonlogger

```bash
sudo apt-get install daemonlogger -y
daemonlogger --help
```

![Daemonlogger Setup](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/5atnyfadb4bmey30myi7.png)

| Flag | Purpose |
|------|---------|
| `-i <intf>` | Capture from this network interface |
| `-l <path>` | Write log files to this directory |
| `-n <name>` | Set the output filename prefix |
| `-d` | Daemonize (run in background) |

---

## Step 2 — Identify Your Network Interface

```bash
ip link show
ifconfig
```

![Network Interface](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jgacc746czvofmo53i7y.png)

---

## Step 3 — Create Log Directory and Start Capture

```bash
sudo mkdir /var/log/daemonlogger
sudo daemonlogger -i eth0 -l /var/log/daemonlogger/ -n capture
```

![Start Packet Capture](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1kgspvog9trsamyt2q12.png)

---

## Step 4 — Generate Traffic

### ICMP — Ping + HTTP — curl

```bash
ping google.com -c 50
curl http://testph.vulnweb.com
```

![Ping and Curl](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/w1ebdtoi0im9m1pz73lx.png)

### TCP Throughput — iperf3

```bash
# Terminal 1 — server
iperf3 -s

# Terminal 2 — client
iperf3 -c 127.0.0.1 -t 30
```

![iPerf3 TCP Traffic](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ofrk3ipwtz2m9ywcx33o.png)

### Website Visits — Browser

Visited `http://neverssl.com` to generate plain unencrypted HTTP traffic.

![Website Visit NeverSSL](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ntnr7j6a9slwuh36k9g6.png)

---

## Step 5 — Stop Capture and Check the File

```bash
sudo pkill daemonlogger
ls -lh /var/log/daemonlogger/
```

![Stop Capture](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4q9xutcee5o3mo9qqd2e.png)

![PCAP File Listed](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/uot51859eyi6cax47lyx.png)

**Result: 13,439 packets — 9.6 MB — zero drops.**

---

## Step 6 — Install Wireshark and Open the File

```bash
sudo apt-get install wireshark -y
wireshark /var/log/daemonlogger/capture*.pcap
```

![Wireshark Open PCAP](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/m08ef7ub6wndlzvj76vc.png)

---

## Step 7 — Wireshark Analysis

### HTTP Traffic — filter: `http`

![HTTP Traffic](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/r6844zutx3tk6wrtn4jz.png)

**HTTP packet count: 25** (0.2%)

---

### HTTPS Traffic — filter: `ssl || tls`

![HTTPS Traffic](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/oseyp33rxjt45bh1ihw4.png)

**HTTPS packet count: 1,833** (13.6%)

---

### TCP Traffic — filter: `tcp`

![TCP Traffic](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/de22g2u3zr7eav5sircj.png)

**TCP packet count: 11,739** (87.4%)

---

### Top Conversations — Statistics → Conversations → TCP tab

![Top Conversations](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/z79bihol0cu032ubsr5d.png)

| Source IP | Destination IP | Port |
|-----------|---------------|------|
| 10.0.2.15 | 34.107.243.93 | 443 |
| 10.0.2.15 | 34.223.124.45 | 80 |
| 10.0.2.15 | 34.223.124.45 | 443 |
| 10.0.2.15 | 44.228.249.3 | 443 |

---

### Top Protocols — Statistics → Protocol Hierarchy

![Top Protocols](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8b8ic6thrufou7r0li1u.png)

| Protocol | % of Packets |
|----------|-------------|
| TCP | 100% |
| HTTP | 8.7% |
| OCSP | 8.7% |
| ICMP | present in unfiltered view |

---

### IO Graph — Statistics → I/O Graph

![IO Graph](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3wwv5ggvmbou4fhuabff.png)

- **Spike at ~250s** — iperf3 burst
- **Quiet baseline** — background HTTPS browsing
- **Late spike ~1000s** — NeverSSL and website visits

---

### Capturing HTTP Credentials

```
http.request.method == "POST"
```

Right-click → **Follow → HTTP Stream** to read plain-text form submissions.

---

## Results Summary

| Metric | Value |
|--------|-------|
| Total packets captured | 13,439 |
| Capture file size | 9.6 MB |
| HTTP packets | 25 (0.2%) |
| HTTPS / TLS packets | 1,833 (13.6%) |
| TCP packets | 11,739 (87.4%) |
| Packets dropped by kernel | 0 |
| Top source IP | 10.0.2.15 |
| Top destination ports | 443, 80 |

---

## Common Mistakes

| Mistake | What happens | Fix |
|---------|-------------|-----|
| Running daemonlogger without `sudo` | Permission denied | Always prefix with `sudo` |
| Skipping `mkdir` for log directory | Daemonlogger exits immediately | Create the directory first |
| Wrong interface name | No packets captured | Verify with `ip link show` first |
| Opening `.pcap` before stopping capture | Truncated/incomplete data | `sudo pkill daemonlogger` first |
| Using only `ssl` filter | Misses TLSv1.3 packets | Use `ssl \|\| tls` |
| Expecting HTTPS credentials | Empty results | Only works on plain HTTP |

---

## 🌐 Connect With Me

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?logo=linkedin)](https://www.linkedin.com/in/almahmudkhalif/)
[![Dev.to](https://img.shields.io/badge/Dev.to-Articles-black?logo=devdotto)](https://dev.to/almahmudkhalif/)
