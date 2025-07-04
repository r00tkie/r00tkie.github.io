---
title: 'Delegation Attacks (PT1)'
date: '2021-02-05'
layout: post
tag: [Delegation Theft, AD, Kerberos]
excerpt: >-
  Kerberos delegation attacks and their impact
description: >-
  Kerberos delegation attacks and their impact
image: '/images/posts/kerbgt-delegation-1/header.jpg'
thumb_image: '/images/posts/kerbgt-delegation-1/header.jpg'
style: code
---

# Introduction

Kerberos Constrained Delegation is a powerful security feature in Microsoft’s Active Directory that enables a service to act on behalf of a user and securely access resources. 
This advanced capability ensures that the service can only interact with specified resources in the user’s name, adding an extra layer of protection to your system.

There are three types of Kerberos Delegation Attacks:

### 1. Kerberos Constrained Delegation (KCD)**

- **What it is**: KCD allows a service to impersonate a user to access resources on the user’s behalf. It’s a secure way to delegate user credentials.
- **How it works**: When a user requests a service that requires delegation, the service sends a request to the Key Distribution Center (KDC) to obtain a service ticket. The KDC returns a service ticket that allows the service to impersonate the user.
- **Security implications**: KCD can be a security risk if not properly configured, as it allows the service to access any resource that the user has access to. This can lead to unauthorized access to sensitive data.

### 2. Kerberos Unconstrained Delegation**

- **What it is**: Unconstrained Delegation is an older form of delegation that allows a service to impersonate a user without any restrictions on the resources that can be accessed. It is like constrained delegation but without not contains
- **How it works**: Similar to KCD, the service sends a request to the KDC to obtain a service ticket. However, the KDC returns a service ticket that allows the service to impersonate the user without any constraints on the resources that can be accessed.
- **Security implications**: Unconstrained Delegation is considered less secure than KCD and RBCD, as it allows the service to access any resource that the user has access to. This can lead to unauthorized access to sensitive data.

### 3. Resource-Based Constrained Delegation (RBCD)**

- **What it is**: RBCD is a more secure form of delegation that allows a service to impersonate a user, but only for specific resources that have been explicitly granted permission to the service account.
- **How it works**: When a user requests a service that requires delegation, the service sends a request to the KDC to obtain a service ticket. The KDC returns a service ticket that allows the service to impersonate the user, but only for specific resources that have been granted permission to the service account.
- **Security implications**: RBCD provides a more granular level of control over what resources the service can access, which reduces the risk of unauthorized access to sensitive data.

To summarize, here’s a comparison of the three types of delegation:

|Type of Delegation|Security Implications|
|---|---|
|Kerberos Constrained Delegation (KCD)|If not properly configured, it can be a security risk, as it allows the service to access resources that the user has access to.|
|Kerberos Unconstrained Delegation|It is less secure than KCD and RBCD, as it allows the service to access any resource.|
|Resource-Based Constrained Delegation (RBCD)|Provides a more granular level of control over what resources the service can access, reducing the risk of unauthorized access to sensitive data.|

# Attacking Kerberos Constrained Delegation

The _userAccountControl_ determines the type of delegation attribute for the object that receives the “**TRUSTED_TO_AUTH_FOR_DELEGATION**” (T2A4D) flag. 
This enables _S4u2self_, which means a service account can obtain a TGS for itself on behalf of any other user using the NTLM password hash. This allows the service to impersonate the domain.

The **msDS-AllowedToDelegateTo** attribute holds the keys to the kingdom, allowing service accounts to obtain Tickets Granting Service (TGS) on behalf of any user to access the service.

We don’t need to set both properties. If _s4u2proxy_ is allowed without _s4u2self_, user interaction is required to get a valid TGS.

To abuse constrained delegation, we need to compromise the password or hash of an account that is configured with constrained delegation to a service. 
Once that occurs, we can possibly impersonate any user in the environment to access that service. 
By doing so, we will have access not only to the service that the user is able to impersonate but also to any service that uses the same account as the allowed one.

### User Account Configured For Constrained Delegation

If we have compromised the plaintext password or the hash for a _user_ account that has constrained delegation enabled, we can request a TGT, execute the S4U TGS request, and then access the target service. We can enumerate for such a user using [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1)

```powershell
Get-DomainUser -TrustedToAuth
```

![](/images/posts/kerbgt-delegation-1/screenshot.jpg)

From the results, we can see that the _constrained.user_ is trusted to delegate and authenticate to CIFS service on First-DC on behalf of any user.

### Requesting a TGT for the user account with constrained delegation enabled

```powershell
.\Rubeus.exe asktgt /user:constrained.user /password:Password@1 /outfile:constrained-user.kirb
```

![](/images/posts/kerbgt-delegation-1/screenshot_1.jpg)

### Using the Rubeus s4u module to obtain a TGS of the Administrator user to self

```powershell
.\Rubeus.exe s4u /ticket:constrained-user.kirbi /impersonateuser:Administrator /msdsspn:cifs/First-DC.first.local /ptt
```

![](/images/posts/kerbgt-delegation-1/screenshot_2.jpg)

Using _klist,_ we can see our stored TGS ticket for the administrator account, and if we attempt to access the file system of First-DC, we can confirm that we have impersonated the domain administrator account that can authenticate to CIFS service.

![](/images/posts/kerbgt-delegation-1/screenshot_3.jpg)

We can confirm that we now have access to the CIFS service on First-DC.

# Mitigations

One way to protect ourselves is to put sensitive accounts that should not be delegated in the Protected Users group and configure them to be ‘_Account is sensitive and cannot be delegated_.’  
Also, we can limit domain admin logins to specific services and keep delegation servers secured by patching them and setting firewall rules according to server purpose and delegation configuration giving limited privileged access.  
It is also important to use strong passwords for the service accounts trusted for delegation.




































