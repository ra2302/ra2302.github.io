---
title: Two - LetsDefend - PCAP Analysis Challenge
date: 2024-02-27 22:00:00 +0530
categories: [LetsDefend]
tags: [LetsDefend, WireShark, easy, PCAP, Network Traffic Analysis ]
---

## Intro
[PCAP Analysis Challenge](https://app.letsdefend.io/challenge/pcap-analysis) is a relatively easy challenge where we're provided with logs from P13's computer and we're supposed to answer the questions. 

### PCAP? What is that?
PCAP stands for packet capture. This blog is made up of many small packets which were transmitted to your system when you entered the URL and pressed enter. A packet capture file basically captures all the traffic(or packets) that are being captured and stores them for further analysis. There are a huge number of packets, that too with different protocols and ports, travelling to and from our machines so it is nearly impossible to analyse them manually. That is why we use WireShark. Probably the most used tools for network analysis.  
With this challenge, we will get a general overview of WireShark and PCAP analysis.

### Question 1
```
In network communication, what are the IP addresses of the sender and receiver?

Answer Format: SenderIPAddress,ReceiverIPAddress

Hint: Find the chat. Sender: P13
```
The first challenge is very straight-forward. P13 communicated with someone over chat. We have to find packets containing the chat and we will be able to find the sender and receiver IP address. 
We can simply put a filter to search for the user "P13". We bascially want the frame to contain "P13". We will use the following filter to search the same. ([Reference](https://www.wireshark.org/docs/man-pages/wireshark-filter.html))
```
frame contains "P13"
```
![chal_1](/assets/pcap-analysis/chal_1.png)

After applying the filter, we ca see that packets containing the conversation between P13 and CU713. And we have our first answer.

### Question 2
```
P13 uploaded a file to the web server. What is the IP address of the server?

```
Here we come to know that P13 uploaded a file to some server. Uploads are usually done with POST requests([More info on POST requests here](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/POST)). So we can simply use another filter for HTTP request and find our upload log. We'll use the following filter.
```
http.request.method == POST
``` 
![chat_2](/assets/pcap-analysis/chal_2_1.png)  
We have our destination IP address now. Third question is pretty much interconnected, let's head on to the TCP stream and see the packets in-depth.

### Question 3
```
What is the name of the file that was sent through the network?
```
The answer for this question is right there in the same packet as seen in the last segment. With the TCP stream open, we can clearly see the uploaded file's info.
![chal_3](/assets/pcap-analysis/chal_3.png)

### Question 4, 5
```
What is the name of the web server where the file was uploaded?

What directory was the file uploaded to?
```
Closing the TCP stream window, in the following stream, we can see a response HTTP packet from the server. 
![chal_4_1](/assets/pcap-analysis/chal_4_1.png)  

We should be able to find info about the server in the HTTP header. Let's open that up as well.

![chal_4n5](/assets/pcap-analysis/chal_4n5.png)

We find not just the answer to the 4th question related to the server info, but we can also see the uploads folder which is the next question. Hence, we hit two birds with one stone.

### Question 6
```
How long did it take the sender to send the encrypted file?

Hint: Check conversations on WireShark
```
The hint leads to the right place to check the time it took for the sender to send the encrypted file. Let check the conversations tab under statistics tab.
![chal_6](/assets/pcap-analysis/chal_6_1.png)

We'll head on to the IPv4 section and look for our server's IP address.

![chal_6_2](/assets/pcap-analysis/chal_6_2.png)

And voila! We can see the answer in the duration section and with this our challenge is completed.

### ***"The next performance will have a lot more.... zazz"***