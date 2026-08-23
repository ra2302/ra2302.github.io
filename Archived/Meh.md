## Intro
[Red Team Capstone](https://tryhackme.com/room/redteamcapstonechallenge) challenge is the final challenge for the Red team pathway. And it consist of one AD forest with two domains, corp & bank. The main aim of the challenge can be better described by the project brief video [here](https://youtu.be/oIOjkGh5m2U?si=e898NLHXnE03xbuJ).

The in-scope activities include:  
- Security testing of TheReserve's internal and external networks, including all IP ranges accessible through your VPN connection.
- OSINTing of TheReserve's corporate website, which is exposed on the external network of TheReserve. Note, this means that all OSINT activities should be limited to the provided network subnet and no external internet OSINTing is required.
- Phishing of any of the employees of TheReserve.
- Attacking the mailboxes of TheReserve employees on the WebMail host (.11).
- Using any attack methods to complete the goal of performing the transaction between the provided accounts.

The out-of-scope activities include:
- Security testing of any sites not hosted on the network.
- Security testing of the TryHackMe VPN (.250) and scoring servers, or attempts to attack any other user connected to the network.
- Any security testing on the WebMail server (.11) that alters the mail server configuration or its underlying infrastructure.
- Attacking the mailboxes of other red teamers on the WebMail portal (.11).
- External (internet) OSINT gathering.
- Attacking any hosts outside of the provided subnet range. Once you have completed the questions below, your subnet will be displayed in the network diagram. This 10.200.X.0/24 network is the only in-scope network for this challenge.
- Conducting DoS attacks or any attack that renders the network inoperable for other users.

The end goal of the engagement is to perform a transaction between two accounts using the bank's swift(hypothetical) banking system.

## Before we begin
I completed this challenge two months back. And now revisiting my notes to write this, I feel that I could've done so much better. After working on forensic labs, I pretty much just gave myself away in the first step. So much I could've done better and stealthier.  
Anyhow, this was my first ever proper AD environment, and it was a great stepping stone. There will be many articles which will be inspired from this and in the future I plan to work on a similar environment with a proper EDR as well.  
I intend to revisit this challenge again and be a lot more stealthy.

### Initial compromise
Initially we're provided with a typo-squatted domain's access (corp.th3reserve.loc). Our goal is to gain initial foothold in Corp AD environment, elevate our privileges and then carry on the exploitation into the bank AD environment where the emulated "SWIFT" software resides.

We're provided with a bit of information like the password policy, a base words list to generate passwords from, and a few tools (mimikatz, powersploit, rubeus, kekeo, spoolsample, PowerView, ForgeCert) which can be helpful in our expedition.  

So the first step was to generate our wordlist. The provided password policy was - 
```
The password policy for TheReserve is the following:

* At least 8 characters long
* At least 1 number
* At least 1 special character
```

I came up with a very simple python script to generate a wordlist - 
```
import itertools

bw = open("./base_words.txt").read().splitlines()
nos = "0123456789"
symbols = "!@#$" 

def generate(word):
    final = []
    for d in nos:
        for s in symbols:
            pw = [
                word + d + s,
                word + s + d,
                s + word + d,
                d + word + s,
                word.capitalize() + d + s,
                word.upper() + s + d,
            ]
            for p in pw:
                if len(p) >= 8:
                    final.append(p)
    return final

with open("password_list.txt", "w") as out:
    for word in bw:
        for password in generate(word):
            out.write(password + "\n")=
```
It might not have been the perfect one but it definitely did the job!
 
There are three internet facing servers -
- VPN server
- Mail server
- Web Server

The web server hosted the main website of the Reserve Bank and it held a surprising lot of information. To start with it had a list of all the C-suite employees and senior members, along with the lead engineers who worked on the website itself. 

When looking at the source of the page it revealed a naming pattern. All the names are stored in this format only - firstname.lastname.  
![usernames](/assets/red-team-capstone/usernames.png)  
Having seen a lot of organisations, this seemed to be a common practice to have standard user principal names.  
Extracting all the users and appending @corp.thereserve.loc, we've got a user list.  
![userlist](/assets/red-team-capstone/userlist.png)

On enumerating the subdomains using gobuster on all the _internet_ facing servers. The most interesting result was on the VPN server.  
![Vpn_gobuster](/assets/red-team-capstone/gobuster_vpn.png)

This discovered path led to directory listing which contained the key to internal network.  
![vpn](/assets/red-team-capstone/vpn.png)

Downloading that ovpn file, it required a slight configuration.  
The remote IP was set incorrectly.  
![error](/assets/red-team-capstone/vpn_config.png)  

Fixed config -  
![fixed](/assets/red-team-capstone/fixed-vpn.png)  

