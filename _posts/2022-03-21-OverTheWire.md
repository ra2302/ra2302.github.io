---
title: OverTheWire Wargames - Bandits
date: 2022-03-21 10:00:00 +0530
categories: [overthewire]
tags: [overthewire, basics, privlege-escalation]
---

## Intro
[Bandits](https://overthewire.org/wargames/bandit/) wargame is one of the most basic game and is aimed toward beginners. There are a total of 33 levels and the levels get harder and the underlying vulnerable technology changes. The levels start with a focus on basics of linux and end with some challenges related to git. At the end of each level, we will find either the password to next level, or we will get the ssh key. In this blog I will demonstrate my journey through the game and explain the command wherever required. So let's start with it!

Note - If you want in depth explaination of some particular command, you can google or dig into the man pages as recommended on the OverTheWire website.

### Level 0
Level 0 is as simple as it can get. We are just required to ssh into the server and read the password file
![lvl0](/assets/bandits/1.png)

### Level 1
In level 1 we are supposed to read a file with a "-"(dash) as it's name.  
![lvl1](/assets/bandits/2.png)

### Level 2
Level 2 has us read a file with multiple spaces in the file name.  
![lvl2](/assets/bandits/3.png)

### Level 3
Here we are supposed to read a hidden file.  
![lvl3](/assets/bandits/4.png)

### Level 4
In this level we are supposed to find a human readable file from a number of files.
I used a simple for loop to iterate through all the files present. We can use grep command to get better output.  
![lvl4](/assets/bandits/5.png)

### Level 5
To reach level 6 we need to find a file which is:  
* Human-readable  
* 1033 bytes in size   
* Not executable  

![lvl5](/assets/bandits/6.png)  
The solution to this levels looks kind of a handful. Let's break it down further. We will begin with the find command
```
find . -type f ! -executable -exec du -ab {} +
```
* "-type f will look for regular files only. 
* "! -executable" will omit the files with the executable privileges. 
* "-exec" will execute the command which follows it whenever there is a match.  
```
du -ab {} +
``` 
The du command is used within the find command and is run whenever the find command finds a match. Here:   
* -a makes sure that the size for all the files are calculated.  
* -h prints the data in the form of bytes.
```
sort -rh | grep 1033
```  
Here the sort command will first reverse(-r) the sorted results and grep will output the line containing the text 1033.  
And with this we got the desired output.

### Level 6
Level 6 has us looking for another file with the following properties:  
* Owned by user Bandit7
* Owned by group Bandit6
* 33 Bytes in size   

![lvl6](/assets/bandits/7.png)  
Here we used the find command similar to previous level but this time we differentiated on the basis of group(-group) and owner(-user). The du command operates similarly as well.  

### Level 7   
Here we are supposed to find the word millionth and right next to it, we will find the password.  
![lvl7](/assets/bandits/8.png)

### Level 8
In this one we are supposed to find the unique line of text in the given file.  
![lvl8](/assets/bandits/9.png)  
Here we used the uniq command with -u parameter printing only the unique files. And used the sort command as a method to feed the data.

### Level 9 
Next up we had to find the humand readlable string followed by "=" characters. We can used the "strings" command for the same.  
![lvl9](/assets/bandits/10.png)

### Level 10
Here we are supposed to decode the data. We used the inbuilt base64 library to decode.  
![lvl10](/assets/bandits/11.png)

### Level 11 
In this level, the data has been rot13 encoded and we are supposed to decode it. We could've used a utility like cyberchef, but the website recommended using tr command.  
![lvl11](/assets/bandits/12.png)

### Level 12
Next up, we are provided with a hexdump file which has been repeatedly compressed.  
![lvl12](/assets/bandits/13.png)  
Here we repeatedly used the following commands to decompress the respective archives.  
```
bzip -x data
```
```
gzip -d data
```
```
tar -xf data
```
### Level 13
In this one, we were provided with a ssh key file which was conveniently placed in our user's home directory.  

### Level 14
Level 14 introduces us to the netcat utility command. We are required to submit the password of the current user(Bandit14) to port 30000. In the previous level we were provided with a ssh key. To get the password we have to dig into /etc/bandit_pass directory. Once we got the password, we can use the nc command as shown below to complete the level.  
![lvl14](/assets/bandits/15.png)  

### Level 15
Here, the concept is pretty similar to the previous level but this time the connection should be SSL encrypted. For that we used openssl s_client command.
```
openssl s_client -connect localhost:30001
```
After the above command, You just have to enter the password to current level and get the pass to next one.  
![lvl15](/assets/bandits/16.2.png)

### Level 16
Level 16 introduces us to another network utility called nmap. We are supposed to find a port running a specific service and using SSL as well. The port range given is 31000-32000. First we will use nmap and afterwards just like previous level, openssl.  
* Note: -sV makes nmap do a service scan, revealing the services running on the open ports.  

![lvl16](/assets/bandits/17.1.png)  
![lvl16.2](/assets/bandits/17.2.png)  
![lvl16.3](/assets/bandits/17.3.png)

### Level 17 
Here we are just supposed to find the only line which is different in the two provided files. diff command makes it really simple.  
![lvl17](/assets/bandits/18.png)

### Level 18
In this level, whenever we try to log in using ssh, we are instantly logged out with a message "ByeBye!". Since we know there is a file called readme in the home directory and we have the ssh password, we can use a utility like scp and transfer the file from the game server to our local system.  
![lvl18](/assets/bandits/19.png)  

### Level 19
Level 19 aims to introduce us to the concept of Setuid bits, I recommend reading about them if you are new to it. 
![lvl19](/assets/bandits/20.png)

### Level 20
In this level, we are provided with another suid bit binary. It makes a connection to the localhost on a specified port. We are supposed to provide the binary the password of bandit20 and we will get the pass to bandit21.  
Now the way binary works is that, we have to first start a listener(using nc) on a desired port and make the binary connect to that port. We need both of them to be running concurrently. For that, we will background one process using CTRL+Z and use fg command to bring the desired process up. Once the binary is connected, Using nc we will supply the current password. Next we will switch to the suconnect binary using fg, where the password will be checked. Once verified we will get the password to bandit21, on our nc listener process.  
![lvl20](/assets/bandits/21.png)
