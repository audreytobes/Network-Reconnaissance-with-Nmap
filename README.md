# Network Reconnaissance with Nmap

## Objectives

- Perform live host discovery to identify active targets on a network
- Conduct basic and advanced port scans to enumerate open services
- Detect service versions and operating systems running on target hosts
- Use Nmap Scripting Engine (NSE) to gather additional target information
- Understand how reconnaissance techniques are used in both offensive and defensive contexts
- Save and interpret scan output for documentation and reporting

### Skills Learned

- Host discovery using ICMP, ARP, and TCP/UDP probes
- TCP SYN, TCP connect, UDP, and NULL/FIN/Xmas scan techniques
- Service and version detection with `-sV`
- OS fingerprinting with `-O`
- NSE script usage for automated enumeration
- Scan output saving and reporting (`-oN`, `-oX`)
- Interpreting Nmap results from a SOC analyst perspective

### Tools Used

- Nmap
- TryHackMe AttackBox / Kali Linux
- TryHackMe target machines (Jr Penetration Tester path)

---

# Section 1 — Live Host Discovery

## Overview

Before scanning ports, a SOC analyst or penetration tester must first identify which hosts are active on the network. This section covers the different probes Nmap uses to determine whether a target is alive.

## Steps

### Step 1 — ARP Scan (Local Network)

ARP scans are the most reliable method for discovering hosts on a local subnet. Nmap sends ARP requests and listens for replies — no firewall can block ARP at the local network level. Discovered one other host on this subnet: 10.67.121.176.

```bash
nmap -PR -sn 10.67.121.0/24
```

<img width="2325" height="688" alt="Screenshot 2026-05-08 141952" src="https://github.com/user-attachments/assets/e1c3c43e-c42d-446a-a026-072248f55f94" />

*Ref 1: ARP scan output showing discovered hosts on the subnet.*

### Step 2 — ICMP Echo Scan

Sends a standard ping to check if the host is alive. Many hosts or firewalls block ICMP, making this less reliable than ARP on external networks. Discovered that the host found using ARP was no longer up or blocking the ICMP echo request, but another host was discovered on the subnet: 10.67.121.95.  

```bash
nmap -PE -sn 10.67.121.176
nmap -PE -sn 10.67.121.0/24
```

<img width="2322" height="537" alt="Screenshot 2026-05-08 142914" src="https://github.com/user-attachments/assets/def20713-81d7-4450-9633-43cbfbdb5568" />

*Ref 2: ICMP echo scan output.*

### Step 3 — TCP SYN Ping

Sends a SYN packet to the common ports (default 80). If the host responds with SYN/ACK or RST, it is considered alive. Useful when ICMP is blocked. I had to reboot my VM at this step, so the subnet changed.

```bash
nmap -PS80,443 -sn 10.65.70.0/24
```

<img width="2340" height="343" alt="Screenshot 2026-05-08 144356" src="https://github.com/user-attachments/assets/5c5260f4-b26e-43d5-89ad-d230bccf6eb5" />

*Ref 3: TCP SYN ping output showing host response.*

### Step 4 — UDP Ping

Sends a UDP packet to check for a response. Less reliable but useful for hosts that block TCP probes.

```bash
nmap -PU -sn 10.65.70.176
```

<img width="2335" height="224" alt="Screenshot 2026-05-08 144655" src="https://github.com/user-attachments/assets/9fba3f45-09db-4758-af4d-b485da167dfd" />

*Ref 4: UDP ping scan output.*

## Findings

### ARP vs. ICMP Reliability
ARP scans are the most accurate for local network discovery because ARP operates below the IP layer and cannot be filtered by host-based firewalls. ICMP scans are frequently blocked, making them unreliable for external targets.

### Host Discovery Without Port Scanning
The `-sn` flag tells Nmap to skip port scanning entirely and only confirm whether hosts are alive — useful for quickly mapping a network without generating excessive traffic.

---

# Section 2 — Basic Port Scans

## Overview

Once active hosts are identified, the next step is determining which ports are open and what services may be running. This section covers the fundamental scan types used to enumerate ports.

## Steps

### Step 1 — Default TCP Scan

Scans the 1000 most common ports using a TCP SYN scan. Requires root/sudo privileges.

```bash
nmap 10.65.152.190
```

<img width="2334" height="733" alt="Screenshot 2026-05-08 154641" src="https://github.com/user-attachments/assets/4bbba318-aa36-4499-907a-b4ee8d16bb8a" />

*Ref 5: Default scan output showing open ports and detected services.*

### Step 2 — TCP Connect Scan

Completes the full three-way handshake. Slower and more detectable than SYN scan but does not require elevated privileges.

```bash
nmap -sT 10.65.152.190
```

<img width="2346" height="628" alt="Screenshot 2026-05-08 154734" src="https://github.com/user-attachments/assets/858c8679-b1d4-44b1-81e1-7940e886ed7c" />

*Ref 6: TCP connect scan output.*

### Step 3 — UDP Scan

Scans for open UDP ports. Slower than TCP scans because UDP is connectionless — Nmap must wait for a response or timeout. At this point, I had to restart the VM so the target address changed. 

```bash
nmap -sU -F -v 10.66.142.113
```

<img width="2324" height="997" alt="Screenshot 2026-05-08 160848" src="https://github.com/user-attachments/assets/47945451-ed20-42d0-80e9-0178f3b01cb1" />

*Ref 7: UDP scan output showing open UDP ports.*

### Step 4 — Scan Specific Ports & Ranges

```bash
# Scan a single port
nmap -p 22 10.66.142.113

# Scan a range
nmap -p 1-1000 10.66.142.113

# Scan all 65535 ports
nmap -p- 10.66.142.113
```

<img width="2391" height="1555" alt="Screenshot 2026-05-08 161113" src="https://github.com/user-attachments/assets/d9f304a8-5331-4294-8ff7-30bfb6118e58" />

*Ref 8: Targeted port scan output comparing results across different ranges.*

## Findings

### TCP SYN vs. TCP Connect
SYN scans (`-sS`) are faster and stealthier because the connection is never fully established, reducing the chance of being logged by the target application. TCP connect scans (`-sT`) complete the handshake and are more likely to appear in application logs.

### Open, Closed, and Filtered Ports
Nmap reports six states: **open** (service listening), **closed** (no service, but accessible), **filtered** (firewall/blocked), **unfiltered** (accessible but state unknown), **open|filtered**, and **closed|filtered**. Filtered ports are significant from a SOC perspective — they indicate a security control is in place.

---

# Section 3 — Advanced Port Scans

## Overview

Advanced scan types allow for stealthier enumeration and can help identify firewall rules and filtering behavior. This section covers NULL, FIN, Xmas, and decoy scans.

## Steps

### Step 1 — NULL Scan

Sends a TCP packet with no flags set. On open ports or ports blocked due to firewall rules, no response is returned. On closed ports, an RST is returned. Can bypass some stateless firewalls.

```bash
nmap -sN 10.67.186.242
```

<img width="2325" height="598" alt="Screenshot 2026-05-08 163314" src="https://github.com/user-attachments/assets/e144f4b5-4c34-4d73-b34f-8866b312025c" />

*Ref 9: NULL scan output.*

### Step 2 — FIN Scan

Sends a TCP FIN packet. Similar behavior to NULL scan — useful for bypassing certain packet filters.

```bash
nmap -sF 10.67.186.242
```

<img width="2338" height="597" alt="Screenshot 2026-05-08 163254" src="https://github.com/user-attachments/assets/4c3c20fd-1990-48fa-a720-d6b90ec5b413" />

*Ref 10: FIN scan output.*

### Step 3 — Xmas Scan

Sets the FIN, PSH, and URG flags simultaneously. Named "Xmas" because the flags light up like a Christmas tree in a packet analyzer.

```bash
nmap -sX 10.67.186.242
```

<img width="2333" height="601" alt="Screenshot 2026-05-08 163518" src="https://github.com/user-attachments/assets/b938d777-fb8a-4760-a5a2-3307f8153cdb" />

*Ref 11: Xmas scan output.*

### Step 4 — Decoy Scan

Spoofs additional source IPs to obscure which host is actually performing the scan. Makes it harder for a defender to identify the real attacker.

```bash
nmap -D RND:5 10.64.191.192
```

<img width="2330" height="447" alt="Screenshot 2026-05-08 215841" src="https://github.com/user-attachments/assets/e7f8c587-a57d-4f82-acd4-bbd05565c965" />

*Ref 12: Decoy scan output showing spoofed source addresses.*

## Findings

### Stealth Scan Techniques
NULL, FIN, and Xmas scans exploit a quirk in the TCP RFC; compliant systems should respond with RST on closed ports and nothing on open ports. However, Windows systems do not follow this behavior and will return RST regardless, making these scans less reliable against Windows targets.

### Defensive Relevance
From a SOC perspective, unusual combinations of flags (no flags, FIN only, FIN+PSH+URG) in network traffic are strong indicators of reconnaissance activity. A properly tuned IDS would alert on these patterns.

---

# Section 4 — Post Port Scans

## Overview

After identifying open ports, the final step is fingerprinting exactly what is running on each service. This section covers service/version detection, OS fingerprinting, and Nmap Scripting Engine (NSE) usage.

## Steps

### Step 1 — Service & Version Detection

Probes each open port to identify the service name and software version. Version information is critical for identifying known vulnerabilities.

```bash
nmap -sV 10.64.178.218
```

<img width="2340" height="631" alt="Screenshot 2026-05-08 222120" src="https://github.com/user-attachments/assets/fbfc394e-d7c3-42c1-8056-96c9c81a42bf" />

*Ref 13: Service version detection output showing service names and versions.*

### Step 2 — OS Detection

Analyzes TCP/IP stack behavior to guess the target's operating system and version. The no exact match is to be expected with TryHackMe's VMs. 

```bash
nmap -O 10.64.178.218
```

<img width="2327" height="1050" alt="Screenshot 2026-05-08 222526" src="https://github.com/user-attachments/assets/0a72b2a4-1cb5-41aa-9865-1e949478696b" />

*Ref 14: OS detection output showing OS guess and confidence level.*

### Step 3 — Default NSE Scripts

Runs Nmap's default script set, which performs additional enumeration, including checking for common vulnerabilities, gathering banners, and identifying misconfigurations.

```bash
nmap -sC 10.64.179.194
```

<img width="1988" height="1300" alt="Screenshot 2026-05-08 223902" src="https://github.com/user-attachments/assets/268b09aa-ddb5-4d1b-809a-9eebd839ef83" />

*Ref 15: NSE default script output.*

### Step 4 — Aggressive Scan

Combines OS detection, version detection, script scanning, and traceroute in a single command. Most comprehensive scan — also most detectable.

```bash
nmap -A 10.64.179.194
```

<img width="2496" height="1203" alt="Screenshot 2026-05-08 225535" src="https://github.com/user-attachments/assets/7fc21401-0da8-4caf-b9eb-9399833c1338" />
<img width="2348" height="1024" alt="Screenshot 2026-05-08 225551" src="https://github.com/user-attachments/assets/1f753d7d-a5fe-4696-af65-31890b994dec" />

*Ref 16: Aggressive scan output showing combined results.*

### Step 5 — Save Output

Saving scan results is standard practice for documentation, reporting, and later analysis.

```bash
# Save as plain text
nmap -A 10.64.164.212 -oN scan_results.txt

# Save as XML (for use with other tools)
nmap -A 10.64.164.212 -oX scan_results.xml
```

<img width="2377" height="1116" alt="Screenshot 2026-05-08 230259" src="https://github.com/user-attachments/assets/86d583c0-7497-4484-a344-0360c0dee671" />
<img width="2868" height="1307" alt="Screenshot 2026-05-08 230241" src="https://github.com/user-attachments/assets/4b7f5c73-8d5c-4297-b9df-441348b831ed" />

*Ref 17: Saved scan output file contents.*

## Findings

### Version Detection and Vulnerability Research
Service version numbers allow analysts and attackers alike to cross-reference against CVE databases. A service running an outdated version with a known exploit is an immediate finding. This is a core part of both penetration testing and SOC vulnerability management workflows.

### NSE Scripts
The Nmap Scripting Engine extends Nmap's capabilities beyond simple port scanning into active enumeration. Scripts can detect vulnerabilities, enumerate shares, brute force credentials, and more. Understanding what NSE scripts do is important for SOC analysts because the traffic they generate has recognizable patterns that should trigger alerts.

### Output Formats
Saving output in multiple formats is a professional habit — plain text for readability, XML for ingestion into other tools like vulnerability scanners or SIEMs.

---

## Security Relevance

Nmap is one of the most widely used tools in both offensive security and network defense. Penetration testers use it for initial reconnaissance; SOC analysts use it for asset inventory, identifying unexpected open ports, and validating firewall rules. Understanding how Nmap works and what its scans look like in network traffic directly improves an analyst's ability to detect and interpret reconnaissance activity against their own environment.

## Conclusion

This project covered the full Nmap reconnaissance workflow across four TryHackMe rooms: live host discovery, basic port scanning, advanced and stealthy scan techniques, and post-scan enumeration with service detection and NSE scripts. Each section builds on the last, progressing from "is a host alive?" to "what exact software version is running and what does that mean?" This structure mirrors the real-world workflow used in both penetration testing engagements and SOC investigations.
