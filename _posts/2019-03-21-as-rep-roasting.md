---
title: 'AS-REP Roasting'
date: '2019-03-21'
layout: post
tag: [Credential Theft, AD, Kerberos]
excerpt: >-
  AS-REP Roasting A Recipe for Kerberos Credential Theft
description: >-
  AS-REP Roasting A Recipe for Kerberos Credential Theft
image: '/images/posts/as-rep/kerberos-authentication.png'
thumb_image: '/images/posts/as-rep/kerberos-authentication.png'
style: code
---

# Introduction

[AS-REP Roasting](https://attack.mitre.org/techniques/T1558/004/) attacks are credential theft attacks that target the [Kerberos authentication protocol](https://pwntales.com/kerberoasting-attacking-the-three-headed-dog/) in Microsoft Active Directory environments. 
The attack exploits a vulnerability in how Kerberos handles authentication requests, allowing attackers to retrieve password hashes for user accounts with pre-authentication disabled.

## The Attack

When a user goes through the authentication process, they reach out to the Key Distribution Center (KDC) to request a Ticket-Granting Ticket (TGT) using an AS-REQ packet. If the user account exists, the KDC will send back a TGT encrypted with the account’s password. This means that only a legitimate user or authorized machine with the proper credentials can decrypt the ticket and proceed.

If a user’s _UserAccountControl_ settings have “Do not require Kerberos pre-authentication” enabled, i.e., Kerberos auth is disabled, it is possible to grab the user’s hash and brute-force it offline.

We can enumerate vulnerable accounts using Powerview

```powershell
Get-DomainUser -PreauthNotRequired -Properties distinguishedname -Verbose
```

![](/images/posts/as-rep/1.png)

in case we want to check if a specific user has the DONT_REQ_PREAUTH property, we can use it like this

```powershell
Get-DomainUser username | ConvertFrom-UACValue
```

![](/images/posts/as-rep/2.png)

If preauthorization is not disabled but we have _GenericWrite_ or _GenericAll_ rights, we can disable the Kerberos pre-authorization.
To achieve this, we must add a _4194304_ (dont_req_preauth) property flag on the user.

![](/images/posts/as-rep/3.png)

We can request the encrypted _(RC4)_ ticket for offline brute force using HarmJoy’s [ASREPRoast](https://github.com/HarmJ0y/ASREPRoast) or with [Rubeus](https://github.com/GhostPack/Rubeus). 
In this example, we’ll use ASREPRoast for ease of use.

Remember that the requested ticket is RC4 encrypted, which will be helpful later on in the detection part.

When you use the _Get-ASREPHash_ function, it essentially packages a New-ASReq to create the right AS-REQ for a specific user/domain. 
It then searches the domain controller for the specified domain, sends the customized AS-REQ, and finally gets back the response bytes.

```powershell
Get-ASREPHash -Domain domain -UserName username -Verbose
```

![](/images/posts/as-rep/4.png)

Alternatively, we can utilize the Invoke-ASREQRoast function to find all users who don’t require Kerberos pre-authentication

![](/images/posts/as-rep/5.png)

Now we take the Hash and crack it locally, for example, using hashcat. We’ll skip it for now.


## Detection and Mitigations

We can monitor domain controller logs for specific event IDs and ticket encryption types to detect AS-REP Roasting attacks.

For example, Event ID 4768 and Ticket Encryption Type 0x17(RC4) indicate a potential AS-REP Roasting attack. 
Additionally, we can use tools like PowerShell to query Active Directory for users with Kerberos pre-authentication disabled and export the results to a CSV file.

![](/images/posts/as-rep/7.png)

To mitigate AS-REP Roasting attacks, we can implement a combination of technical measures, security best practices, and proactive monitoring.

These include:

1. Enabling Kerberos pre-authentication for all user accounts
2. Restricting access to sensitive systems and data
3. Deploying Intrusion Detection Systems (IDS) and Security Information and Event Management (SIEM) solutions
4. Monitoring authentication traffic for suspicious activity
5. Implementing account lockout policies to prevent brute-force attacks
6. Using multi-factor authentication to reduce the risk of credential compromise

Another, but not the most optimal, way to protect your accounts is to use long, complex passwords that are not easily found in breached wordlists. This makes it much harder for attackers to crack them and gain access to your accounts.


















