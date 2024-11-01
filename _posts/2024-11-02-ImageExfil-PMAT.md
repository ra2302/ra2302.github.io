---
title: Four - PMAT Lab - Bonus Binary Malware Analysis
date: 2024-11-02 01:00:00 +0530
categories: [Days of Security, Malware Analysis, PMAT]
tags: [Malware Analysis]
---

## Intro
So with my time being completely MIA and being lazy, I just want to mention that I was learning Malware Analysis and I have gained somewhat of a foothold in the same. This blog is just to highlight the same and share the first report that I wrote for a binary. Albeit it was a sample binary which is a part of the Practical Malware Analysis course from TCM Security. I still feel that it is a beginning to something wel, as I really really enjoyed my time working on the same and learning Malware analysis from scratch. 

### Bonus Binary from PMAT Labs

The binary, as I called it, is an ImageExfil malware, and as the name suggests, it is a exfiltration malware sample. It is a Nim compiled binary that run on 32/64 bit Windows operating system. It consists of one payload only which when is able to ping to a specific domain, starts with the exfiltration of cosmo.jpeg that is stored on the desktop of the system. The URLs are listed in report. A lot of encoded DNS requests are done from the endpoint and it seems to be in a constant loop.

### Report 

I'll be attaching the link to the report that is uploaded to my github reporsitory [here](https://github.com/ra2302/ra2302.github.io/blob/main/assets/pmat-bonus-binary/ImageExfil_Report.pdf). Please feel free to take a look at it. It contains all the data and I have written it as proper report following the template I found on the PMAT lab repository. 
The next few reports will be much more professional as I plan to review different reports and different bits of malware. And I am really excited to deep dive into Malware Analysis & Research and Threat Inteliigence in general. 
I do plan to implement the newly learned skills in my current workplace and hope it brings more value to me as a professional as well.
Looking forward to share more analysis in the future.