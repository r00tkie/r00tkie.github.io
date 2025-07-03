---
title: 'IFEO'
date: '2019-05-10'
layout: post
tag: [Exploitation, Windows]
excerpt: >-
  Image File Execution Options
description: >-
  Image File Execution Options
image: '/images/posts/ifeo/5.jpg'
thumb_image: '/images/posts/ifeo/5.jpg'
style: code
---

# Introduction

Image File Execution Options (IFEO) is a feature in Windows that allows administrators to control the execution of specific executable files. IFEO provides a way to specify custom behavior for certain executable files, such as:

1. Redirecting the execution of an executable to a different file or command.
2. Specifying command-line arguments to be passed on to the executable.
3. Controlling the environment variables that are set when the executable is run.
4. Logging or debugging the execution of the executable.

We need Administrator privileges for this attack to work, as IFEO settings are stored in Windows Registry

## The flow

For our example, we’ll investigate the scenario of attaching a debugger to an application after launch.

When a process is created, a debugger will be attached to the application’s name, launching the new process under the debugger.

We can also launch a monitor program when the specified program silently exits. To do this, enable “_GlobalFlag_” on IFEO and add the monitor value to “_SilentProcessExit_“.

This Windows behavior makes persistence easy since an arbitrary executable can be used as a “_Debugger_” of a specific process or as a “_MonitorProcess_” and affects all Windows versions.  
In both scenarios, code execution will be achieved by creating a process or exiting an application.  
As mentioned previously, implementing this persistence technique requires Administrator privileges since the registry location where the keys need to be added is under _HKEY_LOCAL_MACHINE_.

## The “Debugger”

![](/images/posts/ifeo/1.jpg)

To implement this technique, we must create a registry key and a payload that will be executed upon a specific action.  
The registry key will redirect the execution of any application to a different executable.

We can use whatever payload we like, but in this case, we will modify a C++ reverse shell so that when the targeted EXE launches, it will start our payload in the background.  
Otherwise, the original EXE would not be visible to the user, who might suspect something is wrong.

We added the following code to our reverse shell main function:

![](/images/posts/ifeo/2.jpg)

After uploading our payload to the target, we create the necessary registry keys to implement the persistence technique via “Debugger.”

![](/images/posts/ifeo/3.jpg)

In order to validate the Debugger registry key, we will check the registry on “_HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options_” and verify that the Debugger registry key has been added successfully on notepad.exe

![](/images/posts/ifeo/4.jpg)

When the user launches Notepad, we get a shell that loses connection only if the machine shuts down, and we regain the connection when Notepad runs again.  
This can be done on a startup process instead of a notepad, so we will always have a connection.

![](/images/posts/ifeo/5.jpg)

## The GlobalFlag

For this technique, we need to create three registry keys and a payload that will be executed upon a specific action (e.g., when Notepad is closed)

![](/images/posts/ifeo/6.jpg)

The value **512(0x200)** in the “GlobalFlag” registry key enables the silent exit monitoring for the notepad process.

![](/images/posts/ifeo/7.jpg)

The _ReportingMode_ registry key enables the Windows Error Reporting process (WerFault.exe), which will be the parent process of the “MonitorProcess” **lootsec.exe**

![](/images/posts/ifeo/8.jpg)

As we see on Process Explorer, after closing Notepad, a new process called “**WerFault.exe**” has our payload as a child process.

![](/images/posts/ifeo/9.jpg)

# Mitigations

This attack technique cannot be easily mitigated with preventive controls since it is based on abusing system features.

## References

- [https://attack.mitre.org/techniques/T1183/](https://attack.mitre.org/techniques/T1183/)
- [https://gist.github.com/netbiosX/ee35fcd3722e401a38136cff7b751d79](https://gist.github.com/netbiosX/ee35fcd3722e401a38136cff7b751d79)
- [https://blogs.msdn.microsoft.com/mithuns/2010/03/24/image-file-execution-options-ifeo/](https://blogs.msdn.microsoft.com/mithuns/2010/03/24/image-file-execution-options-ifeo/)
- [https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/registry-entries-for-silent-process-exit](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/registry-entries-for-silent-process-exit)




