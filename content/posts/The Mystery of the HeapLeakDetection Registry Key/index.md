---
title: "The Mystery of the HeapLeakDetection Registry Key"
date: "2022-07-16"
author: "Harel Segev"
tags: ["Registry", "Reverse Engineering"]
draft: true
---

I was working on a case the other day, when I first came across a rather interesting registry key, `HKLM\Software\Microsoft\RADAR\HeapLeakDetection\DiagnosedApplications`. It caught my eye, because it has sub-keys for (what appears to be) applications executed on the system. This is what it looks like on my own system:

![](images/regedit_key_hierarchy.png)

It has quite a few sub-keys, and each one has a *LastDetectionTime* QWORD value, containing what appears to be a Windows FILETIME timestamp:

![](images/regedit_last_detection_time.png)

This key is associated with the Memory Leak Diagnoser component of Windows Resource Exhaustion Detection and Resolution (RADAR). RADAR is a technology embedded in Windows to detect memory leaks in real-time, so that data can be collected and used to correct issues in applications.

RADAR is surprisingly old; it was introduced in Windows Vista. Nevertheless, I couldn't find any research into this registry key - so I had to conduct my own. With forensics in mind, I tried to answer these 2 questions:

* Under what conditions is a sub-key created beneath the *DiagnosedApplications* key?
* Under what conditions is the *LastDetectionTime* value updated?

My research was conducted on Windows 10 machines, and may not apply to prior versions of Windows.

## Looking at the Settings

Luckily, there's a *Settings* key beneath the *HeapLeakDetection* key:

![](images/regedit_settings.png)

Settings are always helpful when you're trying to figure out how something works. Looking at these values, I already had some hypotheses. I figured whether an application is diagnosed has something to do with the amount of memory it allocated. I had a guess that *CommitFloor* and *CommitCeiling* are actually in megabytes, and that the amount of committed memory has to lie between them for a process to be registered. That lead me to write a little C program, to test this theory:

```c
#include<stdio.h>
#include<Windows.h>

int main() {
    // set nAllocatedSize to be between CommitFloor and CommitCeiling
    size_t nAllocatedSize = 1024 * 1024 * 250;
    
    if(VirtualAlloc(NULL, nAllocatedSize, MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE)) {
        printf("Committed 250 MB of memory\n");
        
        while(1) {
            printf("Sleeping...\n");
            Sleep(5000);
        }
    }
}
```

I executed the binary both on my computer at home, and on the one in my office - and the results threw me off completely. It was registered immediately on my computer at home, but wasn't on the one in my office. They're both running the same Windows 10 version, so what's going on?



## Reverse Engineering *RdrpReadHeapLeakSettings*

I needed more information to figure this out. Using Procmon, I was able to pinpoint the DLL which manages the *DiagnosedApplications* registry key, *radardt.dll*:

![](images/procmon.png)

I figured I should start my analysis in the *RdrpReadHeapLeakSettings* function. I hoped it would help me understand the settings better. Take *DetectionInterval*, for example; it's 0x1E, or 30 decimal - but 30 what? seconds? minutes? I had no idea.

![](images/physics_meme.png)

This turned out to be a great idea, because the function transforms each value in a way that enabled me to understand what it means. For example, the *DetectionInterval* value is multiplied by 86400 after it is read from the registry. 86400 happens to be 60 * 60 * 24. This means *DetectionInterval* is stored days, and is converted to seconds:

![](images/ida32_DetectionInterval.png)

In a similar fashion, I was able to conclude that *TimerInterval* is in minutes.

*DetectionInterval* is stored in a global variable after it's converted to seconds - to make it available for other functions to use. This is also the case for *CommitThreshold*, *MaxReports* and *TimerInterval*. However, both *CommitFloor* and *CommitCeiling* aren't stored anywhere after they're read. That lead me to believe they aren't actually used at all!

The next value I looked at is *CommitThreshold*, which seems to be 5 by default. After it is read from the registry, the function checks if it's between 1 to 100:

![](images/ida64_CommitThreshold_01.png)

After that, there's a call to *NtQuerySystemInformation* to get a *SystemBasicInformation* struct. Unfortunately, the structure of this struct is not documented on MSDN:

![](images/msdn_SystemBasicInformation.png)

Nevertheless, I found documentation for it on some other, more questionable website. Then, I was able to define it as a struct in IDA Pro. What I found then, was the explanation to the confusing results of my test:

![](images/ida64_CommitThreshold_02.png)

*CommitThreshold* is multiplied by `(PageSize * NumberOfPhysicalPages) / 100`. That lead me to conclude it's the amount of memory an application has to commit in order to get diagnosed, as a percentage of the total amount of physical memory on the system.

I was able to verify this theory through further testing. This artifact behaves differently depending on the amount of RAM you have installed. My computer at home has 4GB, but the one in my office has 8GB. The 250MB my test application allocated are more than 5 percent out of the 4GB I have at home, but not out of the 8GB I have at my office.

## Conclusions

The Memory Leak Diagnoser is triggered every *TimerInverval* minutes (5, by default). If it detects a process which has a large enough amount of committed memory - more than *CommitThreshold* (5, by default) percent of the amount of installed RAM, it creates a key  for it under `DiagnosedApplications`, if there isn't one already.

If a key exists already, the `LastDetectionTime` value is updated only if at least 30 days have passed since the last detection.
