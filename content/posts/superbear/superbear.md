---
title: "Reverse engineering SuperBear RAT."
date: 2023-08-31T20:43:39+01:00
draft: false
---

You're probably thinking, why is it called SuperBear? Well, here's why:

![pic](/superbear/image.png)

I spent some time analyzing this attack campaign that was impacting civil society groups and thought it would be a good idea to document the technical analysis for the low-level infosec consumers. You can read our high-level report of the malware campaign here on [Interlab's website](https://interlab.or.kr/archives/19416).

Nethertheless I found this sample to be quite interesting since it utilized some interesting techniques. Notably, the usage of AutoIT to perform process hollowing, and then the C2 protocol itself being somewhat similar to that of commodity RATs. 

### AutoIT initial access

In the initial finding of the RAT disclosed on the [Interlab website](https://interlab.or.kr/archives/19416) discusses how we found it to be deployed using an AutoIT script. I won't go into the original maldoc or powershell commands since it's covered in that publication. So let's start by looking at the AutoIT script. 

On inital view, I’d found that the script appeared to be compiled and packed. Since this is a typical feature of AutoIT scripts, I used AutoITExtractor to decompile the script (we made all payloads available on both open-source and commercial malware zoo sites, so if you want to see any of this data yourself check the Interlab post). The source code detailed a trivial process injection operation by hollowing memory from a spawned instance of Explorer.exe. It decrypted a payload and injected it into the hollowed memory.  

![Alt text](/superbear/image-2.png)

The script is too large to cover in this post and not really necessary. The threat actor actually just modified an open-source script that I'd found discussed across a bunch of different forums:
- https://www.autoitscript.com/forum/topic/99412-run-binary/page/8/
- https://syra.forumcommunity.net/?t=55181142
- https://autoit-script.ru/threads/peredacha-parametrov-komandnoj-stroki.24834/

I couldn’t be bothered to reverse the entire crypto operation to get the payload, if you look at the sample you'll see what I mean, so for time sake I just executed it dynamically with the intention of carving out the injected PE. 

When the script spawns explorer.exe we can see a Private address space with RWX permissions. A notable sign of injection!

![Alt text](/superbear/image-3.png)

Inspecting the memory we see an MZ header.

![Alt text](/superbear/image-4.png)

... and contained within that memory is also a string relating to the C2 server that I’d previously found in sandbox detonation. 

![Alt text](/superbear/image-5.png)

From a detection standpoint, this process hollowing operation is pretty loud and I’d be surprised if most Avs didn’t pick it up. You can of course use Process Hacker etc to dump the PE file. Though I ran @hasherezade’s [Hollows Hunter](https://github.com/hasherezade/hollows_hunter) to get a dump of the PE file. Once dumped, I took the PE file to IDA and here’s what I found. 

### SuperBear RAT static reverse engineering

Much like other RATs the malicious code first creates a mutex object using “CreateMutexW”. The mutex name is ("BEARLDR-EURJ-RHRHR"). In then handles an “already running” case by checking if the mutex creation result is 5. If it is, it displays a message stating “already running” and proceeds to exit the process. If it’s not, it calls sub_401070 which establishes the C2 connection. 

![Alt text](/superbear/image-6.png)

The C2 mechanism begins by first allocating memory and initializing a memory block to hold a string “AAAAAA” providing space to store data. It then defines three variables for URI paths “/id1”, “/id2” & “/id3”. 

![Alt text](/superbear/image-7.png)

It then validates the C2 connection by looping over a call to “InternetOpenW” till it successfully establishes a valid “hInternet” handle. Once complete, it uses “InternetConnectW” to connect to the C2 server, in this instance “hironchk[.]com” checking the defined URI paths previously. It iterates through these three URI paths indefinitely until a connection is established. After successful connection it sleeps for 10 seconds before continuing the loop again. 

![Alt text](/superbear/image-8.png)

During dynamic analysis, you can of course see the malware attempting connecting to these URI paths.

Once the C2 connection is established, it performs a HTTP request. Interestingly, the actor left a push instruction that pushes the address of the string “Connected to vk.com” onto the stack. Despite this, the C2 address is not anything to do with VK, though this is an interesting observation a TI perspective.

![Alt text](/superbear/image-9.png)

It checks if the request was successful and then allocates memory using “VirtualAlloc”. The allocated memory is then read from the HTTP connection using the InternetReadFile function. This is looped, and retrieves data from the connection into the “lpBuffer”. 

![Alt text](/superbear/image-10.png)

The HTML data is checked for the string “NdBrldr”, if the string “NdBrldr” is not found during the processing of the loop the loop will exit. If it’s found, it will continue and a string “Found watermark” is pushed to the stack.

After the loop ends the allocated memory is released using “VirtualFree” and the request handle is closed using “InternetCloseHandle”.

Once the connection is established it will do one of four operations depending on the command message received from the C2. 

- Do nothing
- Exfiltrate process and system data
- Download and execute a shell command
- Download and run a DLL

When exfiltrating data, the RAT uses “CreateToolhelp32Snapshot” to create a snapshot of the running processes and saves them to a file located here: 
“C:\\Users\\Public\\Documents\\proc.db”

It then executes the SystemInfo command and saves the output to a file located here:
“C:\Users\Public\Documents\sys.db”

Each of these text files are uploaded to the C2 located at URI “/upload/upload.php”

![Alt text](/superbear/image-11.png)

If the download execute command is seen, it reads a base64 encoded string from the C2 server and decodes it. It then uses the ShellExecuteW to execute this. 

![Alt text](/superbear/image-12.png)

If the DLL command is found, it will pull a DLL payload from the C2 and use rundll32 to execute it. This DLL data is base64 encoded, and when pulled from the C2 it decodes it and stores the decoded result in a memory block. The resultant payload is allocated using “VirtualAlloc” and memory freed by VirtualFree. It attempts to generate a random string for the DLL filename, if it can’t it uses a default string of “SuperBear”.

![Alt text](/superbear/image-13.png)

I won't document the "do nothing" portion :). In the sample I had, the C2 was instructing to only initiate the exfiltration recon activity described above. I wasn't able to pull an addition DLL payload. Maybe we will see what that DLL contains in the future. Definitely something to look out for. 

### Final thoughts

I made all these samples available here:

VT Collection:
https://www.virustotal.com/gui/collection/454cfe3be695d0a387d7877c11d3b224b3e2c7d22fc2f31f349b5c23799967ec/summary

Malware Bazaar:
- SuperBear RAT (dumped PE from memory): https://bazaar.abuse.ch/sample/282e926eb90960a8a807dd0b9e8668e39b38e6961b0023b09f8b56d287ae11cb
- AutoIT process injector: https://bazaar.abuse.ch/sample/5305b8969b33549b6bd4b68a3f9a2db1e3b21c5497a5d82cec9beaeca007630e/

Generally I think the RAT is pretty trivial and also demonstrated functionality similar to xRAT/Quasar. Signature detection for it seem either generic or heuristic, so I'm not sure how much more samples we'll see but since Kimsuky have utilized that in the past, I am wondering why this seems novel. Why did they move from using Quasar to something like this? Also another question that makes me ponder is the utilization of AutoIT. There have been recent reports of open-source tooling being used more by other threat groups in NK, is there a connection here? Maybe I'm just trying to make something out of nothing but I think it's an interesting point to consider.  

When I get round to it, I'll add some Yara rules here and update this post. 

Ovi