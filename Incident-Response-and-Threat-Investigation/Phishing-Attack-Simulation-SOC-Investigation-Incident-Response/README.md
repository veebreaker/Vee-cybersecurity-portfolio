### PHISHING SIMULATION, SOC INVESTIGATION & FULL INCIDENT RESPONSE
## Cybersecurity Home Lab | Email Security | Phishing | Endpoint Monitoring | SIEM | Incident Response
<img width="1358" height="741" alt="Detecting Web Attacks_ A Practical Blue Team Walkthrough (TryHackMe) _ by Rishit Goel _ Medium" src="https://github.com/user-attachments/assets/0f674e66-224e-4b80-999f-aa8bff5803d5" />

Project overview
This project involved building and executing a phishing simulation within a home lab, followed by a full incident investigation and incident response exercise.
The project went beyond sending a phishing email. It involved preparing a Gmail sender account for SMTP delivery, troubleshooting email authentication, configuring GoPhish, developing a web-based verification flow using Flask, testing connectivity between lab components, executing the simulation, and subsequently investigating the resulting activity using a SOC analyst methodology.
The investigation was deliberately separated from the attack simulation. Despite knowing how the scenario had been constructed, I approached the investigation using post-incident evidence, beginning with an employee report rather than relying on knowledge of the attack.
The goal was to practice the technical execution of a phishing scenario while developing the analytical skills required to investigate, correlate, assess, contain, eradicate, and recover from a suspected security incident.
### PART I — PREPARATION AND LAB DEVELOPMENT
## Objectives
The objectives of this project were to: 
1. Develop a working phishing simulation environment.
2. Configure an email sender and troubleshoot SMTP authentication.
3. Understand the technical requirements of email delivery.
4. Configure GoPhish and prepare a phishing campaign.
5. Develop a web-based verification flow using Flask.
6. Troubleshoot networking and application accessibility between virtual machines.
7. Observe user interaction and resulting endpoint telemetry.
8. Conduct a post-incident investigation independently of simulation knowledge.
9. Investigate Windows events, Sysmon telemetry, and Wazuh alerts.
10. Examine suspicious account activity.
11. Apply an incident response playbook from identification through recovery.
12. Document technical challenges, evidence limitations, and lessons learned.
### Lab environment
The project was conducted using a virtualized cybersecurity environment consisting of security-testing infrastructure, a Windows endpoint, monitoring tools, and web applications used in the simulation.
## Components
## Component and Role
1. Kali Linux: Lab security-testing environment and simulation host.
2. GoPhish: Phishing campaign management, email template, campaign tracking.
3. Gmail: SMTP sender account and subsequent security notifications
4. Swaks: SMTP connectivity and authentication troubleshooting
Flask
Web application used for the OTP stage of the simulation
Windows
Target endpoint and source of investigation evidence
Sysmon
Process creation and network telemetry
Wazuh
Endpoint log monitoring, event collection, and alerting
Windows Defender Firewall
Host-level traffic containment
Web browser
Email interaction, page navigation, and browser evidence
Architecture overview
The environment involved the following logical relationships:
GoPhish managed the phishing email campaign.
Gmail was used for email delivery.
The Windows machine received and interacted with the email.
The campaign directed browser activity to the lab-hosted verification environment.
Flask hosted the OTP-stage application.
Sysmon collected process and network activity from Windows.
Wazuh collected and monitored Windows events.
The endpoint and account activity were subsequently investigated. create a table wit this
Note: The architecture should be illustrated using the actual lab configuration and evidence. The investigation narrative must not use attacker-side findings to establish facts that were not independently available to the investigator.
### 3. Gmail sender preparation
A Gmail account was prepared to serve as the email sender for the simulation.
The preparation required configuring authenticated SMTP access rather than relying on ordinary Gmail webmail access.
Work performed
Prepared the sender Gmail account.
Generated an application password for SMTP authentication.
Configured the sender credentials for use with the SMTP service.
Tested the SMTP configuration before integrating it into GoPhish.
Troubleshot authentication failures.
App password
The Gmail app password was used in place of the regular Google account password when authenticating the SMTP client.
### 4. SMTP troubleshooting with Swaks
Before relying on the email campaign, SMTP authentication was tested and troubleshooting was performed.
Swaks was used to inspect the SMTP interaction and identify why the Gmail sender was rejecting authentication.
##  4.1 SMTP configuration
Setting
Value
SMTP server
smtp.gmail.com
Port
587
Encryption
TLS
Authentication
LOGIN
Authentication account
Sender Gmail
Credential
Gmail app password
## 4.2 Errors encountered
During troubleshooting, multiple SMTP responses were encountered.
Response
Investigation significance
5.7.0
SMTP authentication or access rejection
5.5.2
SMTP command/syntax-related failure
535 5.7.8
Gmail authentication failure / invalid credentials
The troubleshooting process involved examining the SMTP connection, authentication attempt, supplied username, password, and SMTP command formatting.
A successful TLS connection was not equivalent to successful authentication. The SMTP server could accept the encrypted connection but still reject the login credentials.
## 4.3 Troubleshooting outcome
The SMTP errors were used to identify configuration and authentication problems before proceeding with campaign preparation.
Evidence to add: Swaks terminal output showing the SMTP responses. Redact email addresses and credentials wherever necessary.
### 5. GoPhish installation and configuration
GoPhish was used as the campaign management platform.
The configuration included the components necessary for the simulation:
Sending profile
Email template
Landing page
Campaign recipient
Campaign configuration
Campaign tracking
## 5.1 Sending profile
The sender profile was configured to use the Gmail SMTP service.
The sender configuration was associated with the Gmail account prepared earlier.
The SMTP service was configured for authenticated delivery using TLS.
## 5.2 Email template
The email template was designed to imitate a Google Workspace security verification message.
Email attributes
Attribute
Value
Sender display name
Security Alert
Sender email
vee321194@gmail.com
Recipient
James vee
Subject
Urgent Action Required
Scenario
Google Workspace identity verification
Deadline
24 hours
The email incorporated personalized language and a verification call to action.
## 5.3 Campaign tracking
GoPhish template variables were used in the email to support campaign tracking and recipient-specific link generation.
The project included tracking elements associated with the campaign, which later became relevant during the email-header and message-body analysis.
### 6. Phishing email construction
The message was constructed around a Google Workspace migration and identity verification scenario.
The email communicated that the recipient needed to confirm their identity to maintain a secure and seamless account experience.
The message included:
Personalized greeting.
Reference to Google Workspace migration.
Account verification request.
A deadline.
A verification link.
Google security-themed language.
A Google Workspace Security Team sign-off.
## 6.1 Social engineering techniques
Technique
Implementation
Authority
Message presented as an account/security request
Urgency
Verification required within 24 hours
Personalization
Recipient addressed by name
Contextual relevance
Google Workspace migration referenced
Familiarity
Google identity and account security language
Call to action
User prompted to verify their identity
Trust exploitation
Familiar security terminology used to lower suspicion
These techniques were incorporated to study how a user might be persuaded to act without independently validating the request.
### 7. Landing-page development and network troubleshooting
The simulation included a Google-themed verification experience.
During development, an important networking problem occurred with the redirect address.
## 7.1 The localhost problem
The initial redirect used:
127.0.0.1:8080
This address refers to the loopback interface of the device making the request.
When the target Windows browser attempted to access 127.0.0.1, it was attempting to reach its own machine rather than the host running the Flask application.
This meant the redirect could not reach the intended Flask service through that address.
## 7.2 Troubleshooting
The issue was identified as an address-reachability problem rather than an application-page problem.
The application needed to be accessible through the address reachable from the target's network environment.
The lab-hosted address used for the OTP stage was:
192.168.56.101:8080
The browser history subsequently showed the OTP page at:
http://192.168.56.101:8080/otp
Lesson learned
localhost is relative to the device making the connection. In a multi-machine lab, a service intended to be accessed from another machine must be reachable through the appropriate interface and address.
Evidence to add: Original redirect configuration, the failed localhost navigation if captured, corrected network configuration, and successful browser access.
### 8. Flask application
Flask was used to host the OTP-stage application in the simulation.
The application was separate from the GoPhish campaign management interface and served the subsequent verification step.
Role of Flask in the simulation
The Flask application was used to:
Host the OTP-stage page.
Provide a browser-accessible endpoint.
Support the second stage of the simulated verification flow.
Record the user's interaction with that stage.
The endpoint observed in the browser was:
http://192.168.56.101:8080/otp
Application troubleshooting
The application was made accessible through the lab network address after the initial localhost redirect failed to reach the service from the target.
This demonstrated the importance of understanding:
Loopback addresses.
Host addresses.
Network reachability.
Service ports.
Browser navigation.
Application accessibility across virtual machines.
Evidence to add: Flask terminal output, application screenshot, endpoint access, and any relevant error/resolution screenshots. Do not publish captured passwords, OTPs, cookies, or active authentication tokens.
### 9. Simulation execution
After preparing the email delivery, campaign, landing page, and OTP-stage application, the simulation was executed.
The recipient received the email and interacted with the verification flow.
The simulation produced activity subsequently observed in the investigation, including:
Phishing email delivery.
Browser navigation to the campaign URL.
Navigation to the OTP page.
Windows network telemetry.
Subsequent unfamiliar account sign-in notification.
The simulation phase was then treated as separate from the investigation phase.
How to Simulate a Phishing Campaign with Defender for Office 365 – Security Occupied

### PART II — SOC INVESTIGATION
10. Investigation methodology
The investigation was conducted from the defender's perspective.
Although the simulation had been executed earlier, the investigative stage relied on evidence gathered after the incident.
The investigation began with the employee report and progressed through independent evidence collection and analysis.
Investigation sources
Employee report.
Email message and original headers.
Browser history.
Windows System event logs.
Sysmon process creation events.
Sysmon network connection events.
Wazuh alerts.
Google account security notification.
Google Account activity.
The investigation was guided by the principle that evidence should support conclusions rather than conclusions dictate the interpretation of evidence.
### 11. Incident identification
The incident began with an employee reporting a suspicious account-verification email.
The employee reported:
Receiving an account security-related email.
Clicking the link contained in the email.
Interacting with a verification page.
Entering account credentials and a verification code.
Subsequently receiving an unfamiliar sign-in notification.
These details formed the initial incident report.
They were treated as reported information pending validation.
### 12. Email investigation
The email was examined to determine its sender characteristics, message content, authentication results, and embedded URLs.
### 12.1 Email details
Field
Value
Display name
Security Alert
From
vee321194@gmail.com
To
James vee
Subject
Urgent Action Required
Date
November 25, 2026
Mail service
Gmail
TLS
Present
Content
HTML
### 12.2 Header analysis
The original message headers included:
Received
Return-Path
Message-ID
X-Mailer
Authentication-Results
Received-SPF
DKIM
ARC
The X-Mailer field identified GoPhish.
The SMTP header information showed the email had been submitted through authenticated Gmail SMTP infrastructure.
The authentication results included SPF, DKIM, and DMARC passes.
These results did not validate the claimed identity of Google Workspace Security. They established the authentication status of the sending infrastructure and domain.
### 12.3 Embedded URLs
The email contained the verification URL:
http://192.168.56.101?rid=RazrWnr
It also contained a hidden tracking URL.
The URLs were documented as evidence of the destination included in the message.
### 13. Browser investigation
Browser history was examined to establish the pages visited after the email interaction.
The history showed:
http://192.168.56.101?rid=RazrWnr
http://192.168.56.101:8080/otp
The second page displayed the title:
Google 2-Step Verification
This provided corroborating evidence of navigation through the verification flow.
Browser history did not independently confirm the content of credentials or OTP submitted, successful authentication, or the existence of an authenticated session.
### 14. Windows endpoint investigation
The endpoint investigation focused on the incident timeframe.
The Windows Security Journey — SysMon (System Monitor) | by Shlomi Boutnaru, Ph.D. | Medium
14.1 Investigation window
November 25, 2026:
1:13 PM – 1:27 PM WAT
The timeframe was selected to cover activity before and after the reported incident.
### 14.2 Sysmon Event ID 1 — Process creation
A process associated with Microsoft Edge was identified.
Field
Value
Process
msedge.exe
PID
428
User
WIN-9NJ4EETVNLO\Administrator
Event
Process creation
Command-line context
Edge network-service subprocess
Process GUID
{e61b2c74-c3a1-6b06-b901-000000000900}
The process command line indicated a network-service subprocess rather than the main browser window.
The Process GUID was used to correlate related network events.
### 14.3 Sysmon Event ID 3 — Network connections
Four network connection events were identified.
Timestamp
Source port
Destination IP
Destination port
1:21:01.905 PM
56239
192.168.56.101
8080
1:21:02.352 PM
49564
192.168.56.101
8080
1:22:30.720 PM
63152
192.168.56.101
8080
1:22:30.750 PM
61194
192.168.56.101
8080
All four events were associated with the Edge process identified by the Process GUID.
The events showed TCP connections initiated from the Windows endpoint to the destination IP and port.
They did not reveal the full URL, credential contents, OTP contents, or independently establish successful submission.
### 15. Wazuh investigation
Wazuh was used to examine alerts generated during the incident timeframe.
The only identified alert directly examined during this investigation was associated with a DNS Client resolution timeout.
Alert details
Field
Value
Rule ID
61109
Severity
Level 5
Windows Event ID
1014
Provider
Microsoft-Windows-DNS-Client
Query
17.76.217.172.in-addr.arpa.
Alert timestamp
1:19:05.966 PM WAT
Agent
Kali-winserver
Agent ID
002
Manager
cvv-wazuh
The alert reported that name resolution timed out.
Additional DNS Client events were found during the investigation.
The alert was investigated as a lead, not treated as proof of phishing activity.
### 16. Google account investigation
A genuine Google security notification reported a new sign-in on Linux.
Google Account activity showed:
Platform: Linux.
Browser: Firefox.
Location: Nigeria.
Time: Approximately 1:22 PM.
First sign-in recorded on November 25.
The notification was correlated with the employee report and the other available evidence.
The exact historical sign-in IP was not available from the account records examined.
Therefore, the source of the sign-in could not be independently attributed to a particular IP address.
### 17. Incident timeline
Time (WAT)
Event
1:18 PM
Suspicious email received
1:18:32.237 PM
Windows DNS Client event
1:19:05.966 PM
Wazuh alert generated
~1:20 PM
Employee reports suspicious interaction
1:21:01.905 PM
Edge network connection to destination
1:21:02.352 PM
Additional Edge connection
~1:22 PM
Unfamiliar Google sign-in notification
1:22:12 PM
Google security notification timestamp
1:22:30.720 PM
Additional Edge connection
1:22:30.750 PM
Additional Edge connection
1:23:19 PM
Additional DNS timeout
### 18. Evidence correlation
The investigation correlated multiple evidence sources.
Evidence
Contribution
Employee report
Initial description of the incident
Email
Sender, subject, message content, URLs
Email headers
SMTP delivery and authentication context
Browser history
Visited URLs and page sequence
Sysmon EID 1
Process identification
Sysmon EID 3
Network connection activity
Wazuh
DNS timeout alert and endpoint event visibility
Google notification
Unfamiliar Linux sign-in
Account activity
Browser/platform and approximate location
The combination of these findings strongly supported a phishing-related incident involving suspicious account activity.
However, the exact source IP of the unfamiliar sign-in, and any data exfiltration were not independently established.

### PART III — INCIDENT RESPONSE
19. Incident classification and impact
The incident was assessed as a phishing-related account security incident involving reported credential and OTP entry and subsequently observed unfamiliar account activity.
Potential impact
Exposure of account credentials.
Exposure of a verification code.
Unauthorized account access.
Potential access to email and associated services.
Potential account persistence through forwarding, recovery settings, app passwords, or connected applications.
The investigation focused on securing the account and checking for persistence or unauthorized changes.
### 20. Incident response playbook
Incident Response for Security+: Key Procedures Overview
## Phase 1 — Identification
Employee report received.
Suspicious email identified.
Unfamiliar sign-in notification reported.
Incident investigation initiated.
## Phase 2 — Detection and analysis
Email headers examined.
Email content and URLs inspected.
Browser history reviewed.
Windows event logs queried.
Sysmon process and network events examined.
Wazuh alerts reviewed.
Google account activity examined.
Timeline constructed.
Evidence correlated.
## Phase 3 — Containment
Account containment measures:
Other Google account sessions logged out.
Password changed.
Authentication strengthened.
Recovery information reviewed.
Endpoint containment measures:
Inbound firewall rule created.
Outbound firewall rule created.
Suspicious destination IP blocked.
Rules verified as enabled and configured to block traffic.
## Phase 4 — Eradication
App password removed.
Recovery settings examined.
Forwarding rules checked.
Gmail filters checked.
Delegation reviewed.
Connected applications reviewed.
Unauthorized persistence mechanisms investigated.
Sender blocked.
Suspicious email reported and removed from the inbox.
## Phase 5 — Recovery
Account security settings verified.
Authentication settings confirmed.
Endpoint examined.
Firewall rules checked.
Normal account and endpoint operation verified.
## Phase 6 — Post-incident review
Investigation limitations documented.
Missing historical attribution identified.
Telemetry gaps recorded.
Detection and logging improvements considered.
Lessons learned documented.

### PART IV — TECHNICAL CHALLENGES
## 21. Preparation troubleshooting
Problem
Investigation / resolution
SMTP authentication failures
Examined SMTP responses and sender authentication configuration
Gmail authentication errors
Troubleshot the authentication configuration
SMTP syntax/decode error
Reviewed the SMTP interaction
Localhost redirect inaccessible from target
Identified loopback address scope
Flask application not accessible using initial address
Used an address reachable through the lab network
Application connectivity
Validated access to the OTP-stage endpoint

### PART V — SKILLS DEMONSTRATED
Email security
SMTP, email headers, sender authentication, social engineering analysis
Networking
IP addressing, ports, loopback, network accessibility, connectivity troubleshooting
Application development
Flask application deployment and web service troubleshooting
Endpoint monitoring
Windows event logs, Sysmon, process and network telemetry
Incident investigation
Evidence gathering, correlation, timeline construction, limitations
Incident response
Containment, eradication, recovery, post-incident review

### PART VI — LESSONS LEARNED
## 23. Technical lessons
SMTP connectivity does not mean authentication has succeeded.
Gmail app passwords are separate from normal account passwords.
Networking issues can arise when applications use loopback addresses across virtual machines.
A web service must be reachable from the client using the appropriate interface and address.
Email authentication does not prove the legitimacy of the sender's claims.
Browser history provides navigation evidence but not necessarily proof of submitted data.
Sysmon Event IDs 1 and 3 provide useful process and network context.
Process GUID correlation is useful when linking process activity.
Wazuh alerts must be investigated and contextualized.
Account recovery and persistence checks are important after suspected account compromise.
## 24. Analyst lessons
The most important lesson was learning to distinguish observed facts, user-reported information, interpretations, and unknowns.
An investigation does not become stronger by forcing every question to have an answer.
When historical information was unavailable, I documented the limitation instead of attributing the activity without evidence.
The investigation also reinforced the importance of correlating multiple data sources rather than relying on a single alert.
PART VII — FUTURE IMPROVEMENTS
## 25. Areas for improvement
Expand Sysmon collection and configuration.
Improve Wazuh detection coverage.
Create more phishing-related detection rules.
Improve endpoint network visibility.
Centralize investigation evidence.
Improve incident timeline generation.
Create more structured incident documentation.
Expand the lab to include more endpoints and security tools.
PART VIII — PROJECT EVIDENCE
## 26. Evidence gallery checklist

### PART IX — FINAL CONCLUSION
This project provided hands-on experience with the technical and analytical aspects of phishing and incident response.
It began with preparing a controlled lab, configuring email delivery, troubleshooting SMTP authentication, building the simulation infrastructure, and validating connectivity between the applications and endpoint.
Following the simulation, I conducted a separate investigation using email evidence, browser history, Windows event logs, Sysmon telemetry, Wazuh alerts, and Google account activity.
The incident response process involved securing the account, reviewing persistence mechanisms, blocking the suspicious destination at the endpoint, verifying security settings, and restoring normal operation.
Beyond learning the tools, the project reinforced the importance of structured troubleshooting, evidence correlation, investigative discipline, and honest reporting of limitations.
It demonstrated that cybersecurity investigation is not simply about finding suspicious activity. It is about establishing what happened, understanding what the evidence supports, identifying what remains unknown, and taking appropriate action to reduce risk.
### APPENDIX — RAW TECHNICAL DETAILS RECOVERED
These are the additional technical details from the work we walked through that should be retained alongside the portfolio.
## A. Email HTML evidence
The email body included the following elements:
Personalized greeting.
Google Workspace migration context.
Identity verification request.
Urgency and 24-hour deadline.
Verification link.
Google Workspace Security Team sign-off.
Hidden tracking image.
The original URL and tracking references were:
Verification URL: http://192.168.56.101?rid=RazrWnr
Tracking URL: http://192.168.56.101/track?rid=RazrWnr
## B. Email header findings
Recovered header details:
Header
Value
From
Security Alert, vee321194@gmail.com
To
James vee, vee93569@gmail.com
Subject
Urgent Action Required
X-Mailer
GoPhish
Return-Path
vee321194@gmail.com
Content-Type
HTML
SMTP submission
Authenticated Gmail SMTP
TLS
TLS 1.3
Authentication
SPF, DKIM, DMARC pass

## C. Network and endpoint evidence
Suspicious destination observed in endpoint evidence
192.168.56.101
Port
8080/TCP
Browser history
http://192.168.56.101:8080/otp
Sysmon process GUID
{e61b2c74-c3a1-6b06-b901-000000000900}
Process
msedge.exe — PID 428
## D. Wazuh alert
Alert rule: 61109
Level: 5
Windows Event ID: 1014
DNS query: 17.76.217.172.in-addr.arpa.
Alert timestamp: 2026-11-25T13:19:05.966+0100
Agent: Kali-winserver
Agent ID: 002
Manager: cvv-wazuh
## E. Response firewall rules
Inbound
IR-Block-Kali-Inbound
Outbound
Blocking outbound connection to phishing IP address
Both rules were verified as enabled and configured to block the suspicious destination address at the Windows endpoint.
