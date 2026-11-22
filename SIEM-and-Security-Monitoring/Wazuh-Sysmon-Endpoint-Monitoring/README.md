# Endpoint Monitoring and SSH Brute-Force Investigation Using Sysmon and Wazuh

**Project Type:** Hands-on SOC Lab  
**Focus Areas:** Endpoint Monitoring, SIEM, Log Analysis, Authentication Monitoring, Network Security and Incident Investigation

**Technologies & Tools:**
- Wazuh SIEM
- Sysmon (System Monitor)
- Kali Linux
- Windows Server 2025
- OpenSSH
- Hydra
- PowerShell
- Windows Event Viewer
- Wireshark / Network Utilities

---

## 1. Project Overview

This project involved deploying an endpoint monitoring environment using Sysmon and Wazuh, followed by a controlled SSH brute-force attack simulation to investigate how Windows Server records authentication activity and how security events are collected and presented in a centralized SIEM platform.

The objective was to understand how endpoint telemetry can support security monitoring, how authentication attempts are recorded, and how a SOC analyst can investigate suspicious login activity.

I used Kali Linux as the testing workstation and Windows Server 2025 as the monitored endpoint. Sysmon was deployed to provide detailed system activity logs, while the Wazuh agent was configured to forward relevant Windows event data to the Wazuh manager.

During the exercise, I encountered network connectivity issues, investigated Windows Firewall configurations, simulated SSH password-guessing activity using Hydra, and examined the resulting authentication events.

One of the most interesting parts of the investigation was identifying a discrepancy between the source IP information available in OpenSSH operational logs and the information displayed in Wazuh authentication alerts.

This project helped me understand that generating security alerts is only one part of monitoring. Being able to verify event collection, understand the original event data, and identify missing investigation context is equally important.

## 2. Project Objectives

The main objectives were to:

1. Deploy Sysmon on Windows Server 2025 for detailed endpoint activity monitoring.
2. Configure Wazuh to collect and centralize Windows endpoint logs.
3. Establish SSH connectivity between Kali Linux and Windows Server.
4. Investigate and resolve network connectivity issues affecting SSH access.
5. Simulate controlled SSH brute-force activity using Hydra.
6. Examine successful and unsuccessful authentication attempts in Windows logs.
7. Configure Wazuh to monitor the OpenSSH operational event channel.
8. Investigate how authentication events appear in Wazuh.
9. Understand the difference between an endpoint agent IP and a remote attacker's IP.
10. Identify potential gaps in event collection and investigation visibility.

## 3. Lab Environment and Architecture

The exercise was conducted in a virtualized environment.

| Component | Role | Operating System / Tool |
|---|---|---|
| Workstation | Attack simulation and testing | Kali Linux |
| Target endpoint | Monitored Windows system | Windows Server 2025 |
| Endpoint monitoring | System activity collection | Sysmon |
| SIEM platform | Centralized security monitoring | Wazuh |
| Authentication service | Remote access | OpenSSH Server |
| Attack simulation | Password-guessing activity | Hydra |
| Investigation | Event and log analysis | PowerShell, Event Viewer |

### Network Configuration

The following addresses were observed during the investigation:

| Machine | IP Address |
|---|---|
| Kali Linux | `192.168.56.101` |
| Windows Server 2025 | `192.168.56.102` |
| Windows agent IP displayed in Wazuh | `10.0.2.15` |
| Wazuh manager | `192.168.109.128` |

The addresses represent the different network interfaces and environments observed during the lab. The Wazuh agent IP was not the remote source address of the SSH attempts.

## 4. Phase One: Deploying Sysmon for Endpoint Monitoring

### 4.1 Downloading and Installing Sysmon

I started by creating a dedicated directory for Sysmon on Windows Server.

```powershell
mkdir C:\Sysmon
cd C:\Sysmon
```

I downloaded Sysmon from Microsoft's Sysinternals download source:

```powershell
Invoke-WebRequest -Uri "https://download.sysinternals.com/files/Sysmon.zip" -OutFile "Sysmon.zip"
```

I then extracted the downloaded archive:

```powershell
Expand-Archive Sysmon.zip -DestinationPath C:\Sysmon -Force
```

### 4.2 Downloading a Sysmon Configuration

Rather than relying entirely on the default Sysmon configuration, I used the SwiftOnSecurity Sysmon configuration as a starting point.

```powershell
$url = "https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml"

Invoke-WebRequest -Uri $url -OutFile "sysmonconfig.xml"
```

This provided a structured configuration for collecting relevant system activity.

### 4.3 Installing Sysmon with the Configuration

I installed Sysmon using the extracted executable:

```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

The `-accepteula` option accepted the Sysinternals license agreement, while `-i` installed Sysmon using the specified configuration.

I verified the service:

```powershell
Get-Service Sysmon64
```

I also checked the active configuration:

```powershell
C:\Sysmon\Sysmon64.exe -c
```

Finally, I opened Windows Event Viewer to confirm that Sysmon was generating events.

This helped establish that Sysmon was installed and collecting endpoint telemetry.

## 5. Phase Two: Forwarding Sysmon Events to Wazuh

After confirming that Sysmon was generating events, I moved on to configuring the Wazuh agent to collect the relevant Sysmon event channel.

I opened the agent configuration file:

```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

I added the following configuration:

```xml
<localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
</localfile>
```

This instructed the Wazuh agent to monitor the Sysmon operational event channel.

After saving the configuration, I restarted the Wazuh agent:

```powershell
Restart-Service -Name WazuhSvc
```

I checked the service status:

```powershell
Get-Service WazuhSvc
```

### 5.1 Generating Test Activity

To generate different types of endpoint activity, I executed several commands on Windows Server:

```powershell
whoami
ipconfig
net user
Invoke-WebRequest -Uri "https://example.com" -UseBasicParsing
```

These commands generated activity involving:

- User and identity information.
- Network configuration.
- Local account enumeration.
- External website requests and name resolution.

The intention was to generate observable system activity and examine the resulting telemetry.

I then checked the Wazuh dashboard to investigate whether the relevant endpoint events were being collected.

## 6. Phase Three: Preparing Windows Server for SSH Attack Simulation

After setting up endpoint monitoring, I moved on to generating authentication activity that could be investigated through Windows logs and Wazuh.

### 6.1 Enabling Windows Auditing

I enabled the following auditing policy:

```powershell
auditpol /set /subcategory:"Other Object Access Events" /success:enable
```

This was part of the initial auditing configuration for the lab.

### 6.2 Checking SSH Availability

I checked the SSH client version:

```powershell
ssh -V
```

I identified the Windows Server network addresses:

```powershell
ipconfig
```

I then started the OpenSSH Server service:

```powershell
NET START sshd
```

And confirmed its status:

```powershell
Get-Service sshd
```

### 6.3 Investigating the Initial Connection Failure

I attempted to establish an SSH connection from Kali Linux, but the connection failed.

Instead of assuming the SSH service was unavailable, I investigated the possible causes.

First, I checked whether TCP port 22 was listening:

```powershell
netstat -ano | findstr :22
```

The output confirmed that the SSH service was listening on port 22.

I then inspected the Windows Firewall rule:

```powershell
Get-NetFirewallRule -Name "OpenSSH-Server-In-TCP"
```

I discovered that the inbound SSH rule allowed connections under the Private network profile.

I also attempted to ping Windows Server from Kali, but the request failed.

To investigate the TCP connection directly, I used:

```bash
nc -zv 192.168.56.102 22
```

The connection timed out.

### 6.4 Identifying the Actual Problem

At this stage, I realized that Windows Server had two network adapters, and both were classified as Public.

The problem was that the SSH inbound firewall rule was configured to allow connections through the Private profile, while the active network adapters were identified as Public.

This meant that even though the SSH service was listening, the firewall configuration was preventing the expected connection.

I inspected the network configuration and identified `Ethernet 2` as the adapter carrying the Kali-facing IP address.

I changed its network category using:

```powershell
Set-NetConnectionProfile -InterfaceAlias "Ethernet 2" -NetworkCategory Private
```

I then checked the network profiles again to verify the change.

### 6.5 Successfully Establishing SSH Connectivity

After changing the network profile, I returned to Kali Linux and tested the connection again:

```bash
nc -nvz -w 5 192.168.56.102 22
```

This time, the connection succeeded.

I then established a remote SSH session:

```bash
ssh Administrator@192.168.56.102
```

After successfully authenticating, I executed commands remotely:

```powershell
whoami
```

```powershell
net start
```

This confirmed that remote access to Windows Server was working.

**Key observation:** A service can be running and listening on the expected port while network security controls still prevent remote access. Verifying the service alone is not enough; firewall rules and network profiles must also be investigated.

## 7. Phase Four: Simulating SSH Brute-Force Activity

With SSH connectivity established, I moved on to generating controlled authentication activity.

I used Hydra from Kali Linux to simulate SSH password-guessing attempts against my Windows Server virtual machine.

First, I created a password list:

```bash
nano password.txt
```

I added a few test passwords and saved the file.

I then executed the following command:

```bash
hydra -l Administrator -P password.txt -t 4 ssh://192.168.56.102
```

I performed two controlled SSH brute-force tests against my own lab environment.

The objective was to generate repeated failed authentication attempts and investigate how Windows Server and Wazuh recorded them.

This exercise demonstrated how repeated login attempts can be generated against an SSH service and why authentication monitoring is important.

## 8. Phase Five: Investigating Windows Authentication Events

After generating the SSH authentication activity, I moved to Windows Server to verify whether the attempts were recorded.

### 8.1 Investigating OpenSSH Operational Logs

I used PowerShell to inspect the OpenSSH operational event channel:

```powershell
Get-WinEvent -LogName "OpenSSH/Operational" -MaxEvents 30 |
Select-Object TimeCreated, Id, LevelDisplayName, Message |
Format-List
```

During the investigation, I found OpenSSH events containing messages such as:

**Failed authentication:**

```text
sshd: Failed password for administrator from 192.168.56.101
```

**Successful authentication:**

```text
sshd: Accepted password for administrator from 192.168.56.101
```

These events were particularly useful because they explicitly identified the remote IP address associated with the authentication attempt.

The logs demonstrated that OpenSSH could record both successful and unsuccessful authentication activity.

### 8.2 Investigating Windows Security Events

I also examined the Windows Security event log.

To inspect failed logons:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4625
} -MaxEvents 20 |
Select-Object TimeCreated, Id, Message |
Format-List
```

To inspect successful logons:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4624
} -MaxEvents 20 |
Select-Object TimeCreated, Id, Message |
Format-List
```

The relevant event IDs were:

| Event ID | Meaning |
|---|---|
| 4624 | Successful account logon |
| 4625 | Failed account logon |

I found SSH-related authentication events associated with:

```text
C:\Windows\System32\OpenSSH\sshd.exe
```

However, I noticed an important limitation in the examples I investigated: the source network address and source port fields appeared as a dash (`-`).

This meant that although the Security event log recorded authentication activity, the particular events I examined did not provide the same source IP information available in the OpenSSH operational logs.

This highlighted the importance of investigating multiple event sources instead of relying on a single Windows log.

## 9. Phase Six: Configuring Wazuh to Monitor OpenSSH Events

After investigating the Windows logs, I wanted to determine whether Wazuh could collect the OpenSSH operational events directly.

I opened the Wazuh agent configuration:

```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

I added the following configuration:

```xml
<localfile>
    <location>OpenSSH/Operational</location>
    <log_format>eventchannel</log_format>
</localfile>
```

This instructed the Wazuh agent to monitor the OpenSSH operational event channel.

I saved the configuration and restarted the agent:

```powershell
Restart-Service -Name WazuhSvc
```

I checked its status:

```powershell
Get-Service WazuhSvc
```

### 9.1 Verifying Event Channel Initialization

I examined the Wazuh agent log and found confirmation that the agent was analysing the OpenSSH event channel:

```text
Analyzing event log: 'OpenSSH/Operational'.
```

I also found evidence of a connection to the Wazuh manager:

```text
192.168.109.128:1514/tcp
```

This confirmed that the agent had initialized monitoring for the OpenSSH channel and had a connection to the manager.

However, I did not treat this as proof that the individual OpenSSH events had successfully reached the manager and become searchable.

That distinction became an important part of the investigation.

## 10. Phase Seven: Investigating the Source IP Discrepancy

One of the most interesting findings was the difference between the IP address displayed in Wazuh and the actual remote source IP recorded by OpenSSH.

The Wazuh alert contained:

```text
agent.name: Kali-winserver
agent.id: 002
agent.ip: 10.0.2.15
```

Initially, the IP address could be mistaken for the source of the SSH attack.

However, `agent.ip` identifies the IP associated with the monitored Wazuh agent. It does not necessarily identify the remote machine initiating the connection.

The lab network information showed:

| Machine | IP Address |
|---|---|
| Kali workstation | `192.168.56.101` |
| Windows Server | `192.168.56.102` |
| Windows agent IP displayed in Wazuh | `10.0.2.15` |

The OpenSSH operational log explicitly identified:

```text
192.168.56.101
```

as the remote source address.

However, the Windows Security authentication events I examined did not expose that same address in their source-address field.

### 10.1 Investigating Missing OpenSSH Entries in Wazuh

I searched Wazuh Threat Hunting using:

```text
Failed password
```

However, I could not find the expected OpenSSH entries.

This raised an important question:

Although the Wazuh agent had initialized monitoring of `OpenSSH/Operational`, were the actual OpenSSH events reaching the Wazuh manager and being stored or indexed?

I had evidence of the agent monitoring the event channel, but I had not yet established end-to-end event delivery and searchability.

This remained an unresolved investigation point.

## 11. Key Findings and Observations

The investigation produced several useful findings.

### Endpoint Monitoring
- Successfully installed Sysmon on Windows Server 2025.
- Applied the SwiftOnSecurity Sysmon configuration.
- Verified the Sysmon service and active configuration.
- Configured Wazuh to monitor the Sysmon operational event channel.
- Generated test system activity to investigate endpoint telemetry.

### Network and SSH Troubleshooting
- Identified that TCP port 22 was listening.
- Discovered that Windows Firewall allowed SSH traffic under the Private profile.
- Identified that both Windows network adapters were initially classified as Public.
- Corrected the network profile of the Kali-facing adapter.
- Successfully established SSH connectivity from Kali Linux.

### Attack Simulation and Authentication Investigation
- Successfully generated controlled SSH authentication activity using Hydra.
- Found successful and unsuccessful authentication messages in OpenSSH operational logs.
- Identified Windows Security event IDs 4624 and 4625.
- Observed differences in the source IP information available from different event sources.

### SIEM Investigation
- Configured Wazuh to monitor the OpenSSH operational event channel.
- Confirmed that the agent initialized monitoring of the channel.
- Confirmed the Wazuh manager connection.
- Identified that the agent IP and remote attacker's IP represent different information.
- Could not yet verify that the individual OpenSSH events were searchable in Wazuh.

## 12. Challenges Encountered

| Challenge | Investigation / Action |
|---|---|
| Initial SSH connection failure | Checked the SSH service, listening port, firewall rule and TCP connectivity. |
| Firewall blocking SSH traffic | Investigated Windows network profiles and identified the mismatch. |
| Windows adapters classified as Public | Changed the Kali-facing adapter to Private. |
| Missing source IP in Security events | Compared Windows Security events with OpenSSH operational logs. |
| OpenSSH entries missing from Wazuh Threat Hunting | Investigated the difference between event channel initialization and confirmed event delivery. |
| Confusion around the Wazuh agent IP | Compared agent metadata with the source IP in the original OpenSSH event. |

## 13. Lessons Learned

This project helped me understand several important concepts in endpoint monitoring and security operations.

**1. A running service does not guarantee network accessibility.**

The SSH service was running and listening on port 22, but the Windows Firewall profile prevented the expected connection. I learned to investigate the complete network path instead of focusing on the service alone.

**2. Different event sources provide different investigation details.**

OpenSSH operational logs provided the remote source IP in the examples I examined, while the Windows Security events did not expose the same information.

**3. SIEM alerts must be interpreted carefully.**

The Wazuh `agent.ip` field represented the monitored endpoint's agent address, not necessarily the remote source of suspicious activity.

**4. Event collection must be verified end to end.**

Configuring an agent to monitor an event channel does not automatically prove that all events are being received, processed, indexed and made searchable.

**5. Troubleshooting is part of security investigation.**

The initial SSH connectivity issue required examining network adapters, firewall rules and connection tests. This reinforced the importance of systematic troubleshooting when investigating security problems.

**6. Multiple sources of evidence improve investigations.**

Comparing OpenSSH operational logs, Windows Security events and Wazuh alerts provided a clearer picture than relying on a single source.

## 14. Project Outcome

I successfully deployed Sysmon on Windows Server 2025, configured Wazuh to monitor endpoint and OpenSSH event channels, established remote SSH connectivity from Kali Linux, and generated controlled authentication activity using Hydra.

I also investigated the resulting Windows authentication events and identified differences between the information provided by OpenSSH operational logs and Windows Security events.

The investigation confirmed the importance of distinguishing between endpoint metadata and remote source information.

However, end-to-end delivery and searchability of the OpenSSH events in Wazuh remained unverified at the end of this exercise.

This project strengthened my understanding of endpoint telemetry, SIEM monitoring, Windows authentication logging, network troubleshooting and the investigation process used in SOC environments.

## 15. Future Improvements

The next steps I would take to improve this lab include:

- Investigating why OpenSSH operational events were not appearing in Wazuh Threat Hunting.
- Verifying event delivery at the Wazuh manager and checking the relevant indices.
- Reviewing Wazuh decoding and detection rules for OpenSSH authentication events.
- Developing custom detection rules for repeated failed SSH authentication attempts.
- Testing whether the remote source IP can be reliably extracted and included in generated alerts.
- Investigating how Sysmon process and network events can provide additional context during authentication investigations.
- Testing alert correlation between endpoint telemetry and authentication logs.
- Developing a more complete incident investigation workflow.

## 16. Evidence Gallery

Screenshots supporting this project will be added to this section.

Each screenshot will include a short description explaining what it demonstrates and how it relates to the investigation.



**Disclaimer:** This project was conducted in an isolated, controlled virtual lab using systems under my control. The attack simulation was performed for educational and defensive security monitoring purposes.

**Project Status:** Implementation and investigation completed; further verification of OpenSSH event delivery and custom detection remains a planned improvement.

---

*Learning through practice. Investigating through evidence. Building towards cybersecurity proficiency.*
