# Applied Cybersecurity Labs

A collection of hands-on projects from coursework spanning digital forensics 
and incident response (DFIR), offensive security, and machine learning 
applied to network security.

## Projects

| Project | Focus | Summary |
|---|---|---|
| [Defense Evasion Investigation](./Defense_Evasion_Writeup.pdf) | [Writeup](Machine Defense_Evasion_Writeup.pdf) | DFIR / Blue Team | Investigated a post-breach host where security logs and Defender alerts were missing. Traced the attacker's full defense-evasion chain — LSA protection tampering, Defender config changes, an AMSI bypass, Safe Mode reconfiguration, and PowerShell logging suppression — using Windows event logs, PowerShell logs, and Sysmon data. |
| [Machine Exploitation Walkthrough](./Machine_Defense_Evasion_Writeup.pdf) | Offensive Security | End-to-end compromise of a Windows target: Nmap service enumeration, unauthenticated SMB share discovery, EVTX log parsing to recover plaintext credentials, credential verification with CrackMapExec, and an RDP session to retrieve the user flag. |
| [Network Intrusion Detection](./Network_Intrusion_Detection_Write-Up.pdf) ([notebook](./network_intrusion_detection.ipynb)) | Security + Machine Learning | Built a two-stage intrusion detection pipeline on 3.1M rows of network traffic: a Random Forest binary classifier (Benign vs. Attack, near-perfect separation) and a PyTorch DNN multi-class classifier across 6 attack types, with class-weighted loss to handle severe class imbalance. Includes a full preprocessing audit documenting a rare attack class that was inadvertently dropped during data cleaning. |

## Skills Demonstrated
- Windows forensics and incident response (Sysmon, Windows Event Logs, PowerShell logging)
- Defense evasion technique analysis (LSA tampering, AMSI bypass, Defender manipulation)
- Network enumeration and exploitation (Nmap, SMB, CrackMapExec, RDP)
- EVTX log parsing and credential recovery
- Applied machine learning for security (scikit-learn, PyTorch)
- Data preprocessing at scale, class imbalance handling, and model evaluation
- Critical review of AI-generated code (catching and correcting a double-softmax bug)

## Stack
Python · PyTorch · scikit-learn · pandas · Nmap · Impacket/CrackMapExec · 
Sysmon · PowerShell · Windows Event Log (EVTX)
