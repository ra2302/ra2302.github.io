---
title: Four - TryHackMe - Diskrupt Forensics Challenge
date: 2025-07-13 01:00:00 +0530
categories: [Days of Security, Digital Forensics, Incident Response, TryHackMe]
tags: [Digital Forensics]
---

## Intro
[Diskrupt](https://tryhackme.com/room/diskrupt) is a hard difficulty digital forensics challenge. The main objective is to fix the damaged disk, analyse the file system, and recover the deleted files. There are 12 questions as part of this challenge. My goal here is not to provide a walkthrough but to demonstrate my capabilities and my thought process so there might be some rambling and side-tracks here.

We're provided with a supposedly corrupted disk image from the user's device. The user is suspected of stealing cutting edge research from the lab and erasing her traces. These are the tasks as an Analyst - 

- Fix the damaged disk
- Examine the partitions
- Find evidence of access to sensitive research documents
- If any files were deleted or tampered with
- What are the hidden files on the disk
- Carve out important files deleted from the disk

Along with the task questions I will be answering these questions at the end as well.

### Question 1
What are the corrupted bytes in the boot sector that caused the disk to be damaged?  

Looking at the disk initially in FTK imager, we can see a populated MBR sector. It contains two partitions, boot drive being NTFS & the other being FAT32. Both are highlighted in different colours. 

![initial_highlight](/assets/diskrupt/initial_look.png)

One thing which pops out instantly is that the MBR signature is not correct. "AC DB" is not the correct signature as highlighted in the above screenshot (turquoise color). It should be 55 AA.  
This answers our first question. The corrupted bytes are ACBD.

This can be very quickly fixed using something like HxD. This makes me wonder how can we go around fixing boot sector in worse condition and how can we go around damaging it. I guess that is something for a future article.  

### Question 2
What are the bytes representing the total sector of the second partition? (Little Endian)

This question has quite a simple approach as well. I already have the file open in HxD so knowing how to read the partition section of MBR helps with this. Referring to my notes, I know the last 4 bytes refer to the total sectors. So for second partitions, the answer is highlighted below. 

![solutionQ2](/assets/diskrupt/q2_sectors.png)

This number comes out to be 20477952. This will help us in the next question.

### Question 3
What is the size of the first partition in GB? (up to 2 decimals e.g: 15.25)

This question was so hard for the so wrong reasons. It is explicitly mentioned that the answer should be in GB AND NOT IN GiB. But the task took the answer in GiB.  
Anyhow, to find the answer, we will be multiplying the number of sectors in a partition with 512, which is the standard sector size in MBR. For first partition, the sector size is highlighted.  

![solutionQ3](/assets/diskrupt/q3_sectors_size.png)

The number comes out to be 63401984, after multiplication and proper conversion, the size of the first disk comes out to be 30.23 ___GiB___ (Not ___GB___)

### Question 4
What is the size of the second partition in GB? (up to 2 decimals e.g: 15.25)

We already calculated the number of sectors for second drive in Question 2, so building upon that, I'll multiply the number i.e. 20477952 with 512 and convert the result to ___GiB___. The answer comes out to be 9.76 ___GiB___.

### Question 5
In the NTFS partition, when was the text file related to the password created on the system?

From here on out, the questions require to open the file in something like Autopsy or use EZTools. Make sure to save the file in HxD with correct MBR signature before loading into Autopsy!

I want to use both tools in conjunction. So I imported the the image with Keyword search & Recent activity ingest modules enabled.

This question can be easiliy answered by using MFTExplorer or MFTECmd tool from EZ tools suite so while the Autopsy was importing the image, I decided to go ahead with this approach.  
First I exported the $MFT using FTK Imager

![FTK_Export](/assets/diskrupt/MFT_export_ftk.png)

Next, parse it using MFTCmd.exe and import into Timeline Viewer

![MFT_CMD](/assets/diskrupt/mft_csv.png)

A quick search for txt files, and a small amount of scroll later, the file was found.

![MFT_Answer](/assets/diskrupt/MFTE_Password_file.png)

### Question 6
What is the full name of the sensitive pdf document accessed on this disk?

For this question, my Autopsy import was completed so I decided to check out the recent files. It was a simple as that.

![important_file](/assets/diskrupt/imp_file_auto.png)

It is good to note that this file has been deleted.

### Question 7
When this file was first found on this disk?

The simplest way to answer that is to refer back to our trusty MFT file. Now we know what we're looking for and our answer is one search away.

![solution_q7](/assets/diskrupt/file_created_mft.png)

### Question 8
What is the entry number of the directory in the Journal that was created and then deleted for exfiltration purposes on the disk?

For this question we'll need the journalling capabilities of NTFS filesystem. The journal file in an NTFS system is located at root\\$Extend\\$USNJrnl\\$J.  
Similar to MFT, MFTCmd can be used to parse the data, and once parsed, can be viewed in Timeline Explorer.

![J_export](/assets/diskrupt/J_export_ftk.png)  
![j_mfte](/assets/diskrupt/j_csv.png)

In Autopsy I found a deleted Zip file. Given the name and it's deleted nature, I figured that this is the directory which is being referred to here. 

![deleted_zip](/assets/diskrupt/Deleted_zip.png)

One quick search in the journal file and we have our entry number for the directory

![entry_number](/assets//diskrupt/entry_no.png)

### Question 9
What is the starting offset of the first zip file found after the offset 4E7B00000?

With this we pivot back to the hex view of the disk. It is a very simple question as well. All we have to do is to go to the mentioned offset and search for the magic number of the Zip file which is "50 4B 03 04". Simple as that.

![go-to](/assets/diskrupt/go-to.png)
![file_found](/assets/diskrupt/file_found.png)

### Question 10
What is the ending offset of the zip file?

To find the ending offset of the file, we're only required to search for the trailer/footer hex values for a zip archive which comes out to be "50 4B ????????????????? 00 00 00". [reference](https://filesig.search.org/)

![end-offset](/assets/diskrupt/end_offset.png)

### Question 11
What is the flag hidden within the file inside the zip file?

Since we have the file in hex form, all we have to do is to carve it out. I used cyberchef to carve the file out and get the super secret file. We can also use varied tools like scalpel, foremost etc.

![cyberchef](/assets/diskrupt/cyberchef.png)
![supersecret](/assets/diskrupt/recovered_zrip.png)

The best part is, that we don't even need to carve out the deleted file. There is another partition which has been deleted. If we open that in FTK Imager we can see the deleted zip & the secret file.

![betterway](/assets/diskrupt/new_way.png)

### Question 12
In the FAT32 partition, a tool related to the disk wiping was installed and then deleted. Can you find the name of that executable?

For this one, once again we have a very simple method. The FTK way. It is right there in the root section.

The easier method being the FTK imager.  
![easy_way](/assets/diskrupt/imager_simple.png)

Now for the question for the Analyst - 

- Fix the damaged disk  
It was a quick signature fix.
- Examine the partitions  
There were two partitions, one NTFS and second, FAT32, which was deleted.
- Find evidence of access to sensitive research documents  
In Autopsy & MFT Explorer we could see the user accessed and deleted the sensitive pdf file.  
- If any files were deleted or tampered with  
Yes, multiple file were deleted as can be seen in FTK imager's screenshots and in the autopsy's bin
- What are the hidden files on the disk  
The whole partition FAT32 could be considered as hidden which contained quite a lot of evidence of malicious activity.  
- Carve out important files deleted from the disk  
We were able to carve out their exfiltration plan from the disk image and other solid evidence was also visible.  

Up next, expect environment update & a lot more DFIR projects!