---
title: Six - TryHackMe - IronShade Linux Forensics Challenge
date: 2025-07-21 01:00:00 +0530
categories: [Days of Security, Digital Forensics, Linux, TryHackMe]
tags: [Digital Forensics]
---
## Intro
[IronShade](https://tryhackme.com/room/ironshade) is a medium difficulty linux forensics challenge. The story goes something like this - we've setup a honeypot and have intentionally exposed a weak SSH and other ports to get attacked by an attacker. One such honeypot was compromised and now we're supposed to analyse what steps did the attacker take to compromise the machine and maintain persistence.  
Although it is a medium difficulty challenge, it didn't took as long as I expected it to take.

### Question 1
What is the Machine ID of the machine we are investigating?  
Hostnamectl is the simplest way to get this.  
![hostnamectl](/assets/ironshade/q1.png)

### Question 2
What backdoor user account was created on the server?  
For this question we can either check /etc/shadow, /etc/passwd or auth.log in case the user has been deleted. In this instance shadow file was more than enough.  
![shadow](/assets/ironshade/q2.png)

### Question 3
What is the cronjob that was set up by the attacker for persistence?  
We have to check the crontab for all users. A simple one liner can do it. This one liner just collects all the usernames from the /etc/passwd file and iterate over the user list and list the cronjobs.    
```
for user in $(cut -f1 -d: /etc/passwd); do crontab -u $user -l; done
```
![crontab](/assets/ironshade/q3.png)  
In this instance, root had the cronjob which we were looking for. printer_app is an ELF file.  

### Question 4
Examine the running processes on the machine. Can you identify the suspicious-looking hidden process from the backdoor account?  
"ps" is our friend when it comes to anything related to running processes. If the process is killed/died we can look for the same in Syslog/Sysmon, auditd or journalctl. Since in this case it is a running process, "ps" should suffice.
```
ps -aux
```
Command explanation -  
-a - Show processes for all users  
-u - List users along with process info  
-x - Show background processes as well  

Since we knew the malicious user account, I added a small grep as well.  
![ps](/assets/ironshade/q4.png)

### Question 5
How many processes are found to be running from the backdoor account’s directory?  
The answer to this question can be found with last question only.  

### Question 6
What is the name of the hidden file in memory from the root directory?  
The "in memory" part of the question confused me but checking the root directory showed a somewhat misplaced hidden file and it was the solution.  
![hidden](/assets/ironshade/q6.png)

### Question 7
What suspicious services were installed on the server? Format is service a, service b in alphabetical order.  
Systemctl is the way to go when it comes to services in linux. This command from a previous room on TryHackMe linux forensics does it all. (The benefits of keeping notes!!)  
```
sudo systemctl list-units --all --type=service
```
A little bit of scrolling and we have the solution.  
![service1](/assets/ironshade/q7_1.png)  
![service2](/assets/ironshade/q7_2.png)  

Bonus service info -  
![service_info](/assets/ironshade/service_info.png)  
Sys_backup doesn't seem to exist. When checking in the Syslog, it was also visible that the file sys_backup never existed and it never ran successfully.
![syslog](/assets/ironshade/syslog.png)

Not getting sidetracked anymore, moving on to the next question

### Question 8
Examine the logs; when was the backdoor account created on this infected system?  
Auth.log is of utmost importance when it comes to user related evidence. A quick grep we can view when the user was created.  
![new_user](/assets/ironshade/ne_user.png)

This screenshot revealed one more thing. There was one more user which was created. That is worth investigating as well.  
![badactor](/assets/ironshade/badactor.png)  
It seems nothing much happened with this user. The user has been deleted and the rdp_updater doesn't exist either. Since the VM is not connected to the internet there's no way I could analyse this further without proper tools.

### Question 9
From which IP address were multiple SSH connections observed against the suspicious backdoor account?  
Auth.log for the win here as well.
```
cat auth.log* | grep -a mircoservice
```
![ssh_login](/assets/ironshade/ssh_login.png)

### Question 10
How many failed SSH login attempts were observed on the backdoor account?  
Once again a little bit of auth.log scrolling and we have the answer.  
![Failed_logins](/assets/ironshade/failed_login.png)

### Question 11
Which malicious package was installed on the host?  
For this question we can check the dpkg logs. It holds the logs for installation and deletion of all the packages.
```
cat dpkg.log | grep "install "
```
One application stands out and it was the malicious package.
![dpkg_package](/assets/ironshade/dpkg1.png)

### Question 12
What is the secret code found in the metadata of the suspicious package?  
This time around, instead of dpkg logs we can simply check the dpkg list with the filter of the package which we came to know about in the last question.  
```
dpkg -l | grep pscanner
```
![dpkg_list](/assets/ironshade/dpkg_list.png)


This concludes this room. I did revisit the questions and attempted to get the information through OSQuery and I learnt a lot more from that method but I didn't capture screenshots for that. So in this blog, we only get the simplistic method where we query and read everything manually. I wonder how much of it can be automated.