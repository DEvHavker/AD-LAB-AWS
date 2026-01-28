# AD-LAB-AWS
Cybersecurity project simulating brute force password spray attack on AD environment

## Commands

# 1. Reconnaissance & Enumeration
Before launching the attack, these commands were used to identify the target and understand its available services.
	•	Nmap Scan: Used to discover open ports and services on the target Domain Controller.
	◦	sudo nmap -p- -sV -T5 10.0.1.100
	◦	Significance: This identified that Port 445 (SMB) was open, which is essential for the password spray.
	•	Enum4linux-ng: Used for deep enumeration of the Windows environment.
	◦	enum4linux-ng -A <ip-address>
	◦	Significance: This helped extract information such as domain SID, password policies, and user lists from the AD server.
# 2. Password Spraying & Exploitation
These commands were the core of the "Red Team" phase, used to identify valid credentials for the lab.local domain.
	•	Kerbrute Password Spray: Used to test a single password against a list of users via the Kerberos protocol.
	◦	./kerbrute passwordspray -d lab.local --dc <ip-address> users.txt 'Password123!'
	◦	Significance: This allowed you to check many accounts at once without triggering a lockout on a single account.
	•	NetExec (nxc) / CrackMapExec: Used to verify credentials and check for administrative privileges.
	◦	nxc smb 10.0.1.65 -u users.txt -p passwords.txt
	◦	Significance: This confirmed the successful login for labuser and indicated the [+] success status in your terminal.
# 3. VNC Server Management
Because you were connecting to Kali via a JumpBox, you used these commands to manage the graphical interface.
	•	Start VNC Server:
	◦	vncserver -localhost no :1
	•	Kill VNC Session (Reset):
	◦	vncserver -kill :1
	•	Clear Lock Files:
	◦	rm -f /tmp/.X1-lock
	◦	rm -f /tmp/.X11-unix/X1
# 4. Splunk Detection (SPL Queries)
While not terminal commands, these Search Processing Language (SPL) queries were used within the Splunk Web UI to detect the attack:
	•	Find Failed Logins:
	◦	index=main EventCode=4625 | stats count by TargetUserName, Source_Network_Address
	•	Find Successful Breach:
	◦	index=main EventCode=4624 Logon_Type=3

## SETUP

# Step 1: Configure the Network (VPC & Security Groups)
Before launching instances, you must build the environment they will live in.
	1	Create a VPC: Use a dedicated IPv4 CIDR block (e.g., 10.0.1.0/24).
	2	Define Security Groups: Create three distinct groups to control traffic:
	◦	Attacker (Kali): Allow VNC (5901) and SSH (22) from your JumpBox.
	◦	Target (Windows): Allow SMB (445) and RDP (3389) from the VPC, and WinRM if needed.
	◦	SIEM (Splunk): Allow Web UI (8000) and Log Indexing (9997) from the VPC.

# Step 2: Provision EC2 Instances
Launch three EC2 instances with the following configurations:
	1	Kali Linux (Attacker): Choose a t3.medium AMI from the AWS Marketplace.
	2	Windows Server (Target): Use Windows Server 2022 Base. Ensure it has at least 4GB of RAM.
	3	Ubuntu (Splunk): A t3.large instance is recommended to handle Splunk’s indexing load.

# Step 3: Promote Windows Server to Domain Controller
	1	Log in to the Windows instance via RDP.
	2	Open Server Manager and select Add Roles and Features.
	3	Install Active Directory Domain Services.
	4	Once installed, click the flag icon and select Promote this server to a domain controller.
	5	Create a new forest (e.g., lab.local) and create target user accounts (e.g., labuser) for your attack simulation.

# Step 4: Configure the Telemetry Pipeline (Splunk)
	1	Install Splunk Enterprise: On your Ubuntu instance, download and install the Splunk .deb package. Enable port 9997 for receiving data.
	2	Install Splunk Universal Forwarder: On the Windows Domain Controller, install the Forwarder.
	3	Configure Inputs: Point the Forwarder to your Splunk Ubuntu IP on port 9997.
	4	Audit Policy: On the Windows Server, use Group Policy (GPO) to enable auditing for "Logon" events to ensure Event IDs 4624 and 4625 are being generated.

# Step 5: Execute and Detect
	1	Connect to your Kali instance and run the Password Spray using nxc: nxc smb 10.0.1.65 -u users.txt -p passwords.txt
	2	Switch to your Splunk Web UI and run a search to see the results: index=main (EventCode=4624 OR EventCode=4625)
