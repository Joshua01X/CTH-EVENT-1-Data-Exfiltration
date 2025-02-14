# Threat Hunting Investigation: Exposed Devices on the Internet

## Introduction/Objectives
As part of an advanced threat-hunting initiative, this project investigates potential security incidents using Microsoft Defender for Endpoint (MDE) within a virtual machine hosted on Microsoft Azure. The primary focus of this investigation is to analyze compromised endpoints, detect Indicators of Compromise (IoCs), and uncover unauthorized activities related to devices exposed to the internet. 

The investigation follows a structured approach using Kusto Query Language (KQL) to analyze event logs, correlate suspicious activities, and identify tactics employed by threat actors. This research-driven approach provides valuable insights into adversary techniques and strengthens an organization's defensive capabilities.

## Components, Tools, and Technologies Employed
- **Cloud Environment:** Microsoft Azure (VM-hosted threat-hunting lab)  
- **Threat Detection Platform:** Microsoft Defender for Endpoint (MDE)  
- **Query Language:** Kusto Query Language (KQL) for log analysis  

## Disclaimer
This project is conducted within a shared learning environment hosted under the same Microsoft Azure subscription. Consequently, certain private IP addresses showing failed logon attempts may appear for testing purposes. The bad actors referred to in this project are remote IPs originating from external locations, particularly accounts that are not subscribed under the company’s Azure environment.

## Scenario
A high-ranking executive, Bryce Montgomery, is under investigation for potentially exfiltrating sensitive corporate data. The Risk Department suspects unauthorized data access, and the Security Operations team has been tasked with uncovering any suspicious activities linked to his workstation and other potential machines used. The investigation involves tracing file interactions, identifying lateral movements, and detecting data exfiltration attempts.

## High-Level IoC Discovery Plan
The investigation follows a structured approach:
1. **Identifying initial file interactions** on Bryce Montgomery’s workstation.
2. **Checking for alternative access points**, including shared workstations and remote logins.
3. **Tracking renamed files** to verify if they were moved or modified across different systems.
4. **Investigating the use of steganography** to hide sensitive data within image files.
5. **Tracing potential compression and exfiltration methods**, such as the use of archiving tools.
6. **Finding conclusive evidence** linking Bryce Montgomery to the data theft.

## Steps Taken

### Phase 1: Initial File Interaction
#### Summary
The investigation starts with monitoring Bryce Montgomery’s workstation for any file interactions. Since he holds administrative privileges, his activity must be carefully analyzed. A KQL query is used to identify the first corporate file accessed during the investigation.

#### KQL Query Used
```kql
DeviceFileEvents
| where DeviceName == "corp-ny-it-0334"
| where InitiatingProcessAccountName == "bmontgomery"
```
![image](https://github.com/user-attachments/assets/0b94394a-21ac-46e8-9a2b-0117677263a8)


### Phase 2: Identifying Alternate Workstations
#### Summary
Given that no significant findings were observed on the primary workstation, the possibility arises that Bryce used a different system, possibly through shared workstations with generic user profiles.

#### KQL Query Used
```kql
DeviceLogonEvents
| where InitiatingProcessRemoteSessionDeviceName == "Guacamole RDP"
```
![image](https://github.com/user-attachments/assets/6674ce82-fc78-4810-8e0b-fd0f64c77ed2)


### Phase 3: File Renaming Analysis
#### Summary
Suspicious files were identified on multiple machines, suggesting that file renaming might have been used to obfuscate sensitive information. This phase aims to correlate renamed files across different systems.

#### KQL Queries Used
```kql
DeviceFileEvents
| where DeviceName == "lobby-fl2-ae5fc"
| where PreviousFileName in ("Q1-2025-ResearchAndDevelopment.pdf", "Q2-2025-HumanTrials.pdf", "Q3-2025-AnimalTrials-SiberianTigers.pdf")
| project Timestamp, DeviceName, PreviousFileName, FileName, FolderPath, InitiatingProcessFileName
| order by Timestamp asc
```

```kql
DeviceProcessEvents
| where DeviceName == "lobby-fl2-ae5fc"
| where ProcessCommandLine has_any ("bryce-homework-fall-2024.pdf", "Amazon-Order-123456789-Invoice.pdf", "temp___2bbf98cf.pdf")
| project Timestamp, DeviceName, InitiatingProcessFileName, ProcessCommandLine
| order by Timestamp asc
```
![image](https://github.com/user-attachments/assets/853d17bf-655f-4ea6-8c05-23cfcdd3e9ed)


### Phase 4: Steganography Analysis
#### Summary
The investigation uncovers the use of **steghide.exe**, a tool commonly employed for steganography. This phase seeks to identify the output files generated by the execution of steghide.

#### KQL Query Used
```kql
DeviceFileEvents
| where DeviceName == "lobby-fl2-ae5fc"
| where InitiatingProcessFileName == "steghide.exe"
| where ActionType == "FileCreated"
| project Timestamp, DeviceName, FileName, FolderPath, InitiatingProcessFileName, ActionType
| order by Timestamp asc
```
![image](https://github.com/user-attachments/assets/96be3aec-7a1d-40e9-8f83-19fa2007bd7b)


### Phase 5: Tracking Process Interactions
#### Summary
Stego images were recovered from Bryce's machine. The objective now is to track any other processes that interacted with these images to further determine data exfiltration methods.

#### KQL Query Used
```kql
DeviceProcessEvents
| where DeviceName == "lobby-fl2-ae5fc"
| where ProcessCommandLine has_any ("bryce-and-kid.bmp", "bryce-fishing.bmp", "suzie-and-bob.bmp")
```
![image](https://github.com/user-attachments/assets/431ad9cf-4799-47a1-bf97-100b43ae96a5)


### Phase 6: Identifying Archive Creation
#### Summary
Further analysis reveals that a **7z.exe** command was used to compress the files, suggesting that they were prepared for exfiltration. The objective is to locate the final storage location of the archive.

#### KQL Queries Used
```kql
DeviceFileEvents
| where DeviceName == "lobby-fl2-ae5fc"
| where FileName endswith ".zip"
| where InitiatingProcessFileName == "7z.exe"
| order by Timestamp asc
```

```kql
DeviceFileEvents
| where DeviceName == "lobby-fl2-ae5fc"
| where FileName endswith ".zip"
| where ActionType in ("FileCreated", "FileModified", "FileMoved", "FileRenamed", "FileCopied")
| where InitiatingProcessAccountName == "lobbyuser"
| where InitiatingProcessFileName has_any ("7z.exe", "cmd.exe", "powershell.exe", "msedge.exe", "explorer.exe", "winrar.exe", "winzip.exe", "zip")
| order by Timestamp asc
```
![image](https://github.com/user-attachments/assets/732ce97d-6c04-49c2-a703-07a9d8cde15b)


### Phase 7: Conclusive Evidence
#### Summary
Final analysis focuses on establishing conclusive proof that Bryce Montgomery was responsible for the attempted data exfiltration.

#### KQL Query Used
```kql
DeviceFileEvents
| where DeviceName == "corp-ny-it-0334"
| where InitiatingProcessAccountName == "bmontgomery"
| where FileName endswith ".zip"
```
![image](https://github.com/user-attachments/assets/e4dc9d2c-5385-4741-a96f-cd3e8845ba65)


## Tactics, Techniques, and Procedures (TTPs) from MITRE ATT&CK Framework
- **T1078** – Valid Accounts (Shared workstations, account impersonation)
- **T1059** – Command and Scripting Interpreter (Steghide.exe usage)
- **T1027** – Obfuscated Files or Information (Steganography-based data hiding)
- **T1036** – Masquerading (Decoy file renaming, fake filenames)
- **T1021.001** – Remote Desktop Protocol (Guacamole RDP for lateral movement)
- **T1083** – File and Directory Discovery (Locating sensitive corporate files)
- **T1039** – Data from Network Shared Drive (File transfers across workstations)
- **T1560.001** – Archive via Utility (Using 7z.exe for exfiltration)

## Conclusion
This investigation successfully tracked unauthorized access, file movements, steganography attempts, and exfiltration activities linked to Bryce Montgomery. The structured approach and use of MDE with KQL provided definitive proof of insider threat activities, reinforcing the importance of proactive threat hunting within enterprise environments.

