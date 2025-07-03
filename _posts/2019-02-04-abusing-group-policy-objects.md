---
title: 'GPO Abuse'
date: '2019-02-04'
layout: post
tag: [AD, Group Policy ]
excerpt: >-
  Abusing Group Policy Objects
description: >-
  Abusing Group Policy Objects
image: '/images/posts/gpo-abuse/gpo-abuse.jpg'
thumb_image: '/images/posts/gpo-abuse/gpo-abuse.jpg'
style: code
---

# Introduction

In a previous post, we discussed about [Active Directory ACL/ACE](https://pwntales.com/blog/abusing-ad-acl-ace) and how to abuse them. Now we will see how to abuse Active Directory’s GPOs


## What is Group Policy Objects (GPO)

Active Directory Group Policy Objects (GPOs) are a feature of Microsoft’s Active Directory that allows administrators to easily manage and configure settings for users, computers, and other Active Directory objects.  
GPOs, in general, allow us to define and enforce policies across a network, such as security settings, software installations, and user preferences. This is super useful for keeping everything running smoothly!

GPOs are rules and preferences assigned to objects in the Active Directory and stored in the Active Directory database.  
To apply a GPO to an object, we can use the Group Policy Management Console (GPMC) or other management tools.

Settings we can configure using GPO include, but not limited to:

1. Security settings: Configuring password policies, account lockout policies, and other security settings.
2. Software deployment: Deploying software applications and updates to computers and users.
3. User settings: Configuring desktop settings in general.
4. Network settings: Configuring network settings, such as IP addresses, DNS settings, and proxy settings.
5. Group Policy Preferences: Defining custom preferences and settings for specific applications or system components.

![](/images/posts/gpo-abuse/screenshot.jpg)

Every OU with a GPO can be discovered via the **gpLink** attribute.  
GpLink is an attribute of the AD Object to which the group policy is linked.

![](/images/posts/gpo-abuse/screenshot_1.jpg)

The Group Policy Objects are located in the Policies container.

![](/images/posts/gpo-abuse/screenshot_2.jpg)

Creating a new GPO in the Group Policy Management Console will place the object in the **CN=Policies**.

# Abusing Group Policy Objects

Again, we can use [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1) to start enumerating the permissions for all GPOs in the current domain.

```powershell
Get-NetGPO | %{Get-ObjectAcl -ResolveGUIDs -Name $_.Name}
```

We find that the User ‘GPOWRITE.USER’ can modify the ‘Default Domain Controllers Policy’ GPO linked to the ‘Domain Controllers’ OU, which contains the ‘FIRST-DC’ computer object

In Bloodhound, this looks like this:

![](/images/posts/gpo-abuse/screenshot_3.jpg)

Now, to abuse this, we can use [SharpGPOAbuse](https://github.com/FSecureLABS/SharpGPOAbuse)

![](/images/posts/gpo-abuse/screenshot_4.jpg)

The command was successful, and the GPO is now updated.  
The updated file is visible on the First-DC, and the Group Policy Management console shows the precise settings.

![](/images/posts/gpo-abuse/screenshot_5.jpg)

The GPO is refreshed by default about every 90 minutes. We can afterward verify our results on the target system.

Looking at the Task Scheduler, we can see that **Taks1** has been created.  
When executed, we receive our callback.

![](/images/posts/gpo-abuse/screenshot_6.jpg)

And we got our shell.

![](/images/posts/gpo-abuse/screenshot_7.jpg)

## Things to consider from a pentesting perspective

- The impact could be significant, so verify how this affects your position. I.e., how many machines will this concern, what sort of machines, what infrastructure criticality, etc.  
    An alternative route to reach your goal may have less of an impact.
- An alternative approach to highlighting this risk to a client might be demonstrating through RSAT by creating a blank GPO and linking it to the OU without creating the task or any actions.
- To the best of my knowledge, I couldn’t see a ‘remove’ or ‘reverse’ action within SharpGPO Abuse to revert back/clean up our modifications.  
    You should be mindful of this as a Pentester/Red Team member.
- This is not OPSEC safe. Admins may see the changes in the GPO console.
- Always remember to clear up after yourself.














