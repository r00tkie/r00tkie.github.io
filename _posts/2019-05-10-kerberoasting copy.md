---
title: 'Kerberoasting'
date: '2019-05-10'
layout: post
tag: [Credential Theft, AD, Kerberos]
excerpt: >-
  Roasting the three header dog
description: >-
  Roasting the three header dog
image: '/images/posts/kerberosting/kerberoasting.png'
thumb_image: '/images/posts/kerberosting/kerberoasting.png'
style: code
---

# Introduction

When users access computer systems, they do so by entering a password. The challenge with this authentication method is that if someone maliciously obtains the password, he can take on the user’s identity and gain access to an organization’s network. Organizations need a better way to protect their systems and users. This is where the Kerberos protocol comes into play, as it’s designed to provide secure authentication over a network.

## How Does Kerberos Authentication Work?

Kerberos uses symmetric key cryptography and a key distribution center (KDC) to authenticate and verify user identities.  

KDC contains the following components:

- An Authentication server (AS) that performs the initial authentication and issues ticket-granting tickets (TGT) for users.
- The Ticket Granting server (TGS) issues service tickets that are based on the initial TGT.

#### Kerberos flow:

1. Client by logging in sends a request to the Authentication server for server access. The request is encrypted with the client’s password
2. The Authenticator server retrieves the client’s password from the Database based on the client’s ID and uses his password to decrypt the request and then if the user is verified it sends a TGT encrypted with a key which is shared between the Authenticator and the Ticket-granting server (TGS).
3. The client sends the TGT to the Ticket-granting server to request access to the server
4. The TGS  decrypt the TGT and sends a Kerberos token to the client
5. The client sends request using the Kerberos token to the server
6. Finally, the server allows access to the client

![](/images/posts/kerberosting/1.jpg)

When performing Internal Penetration Testing or a Red Team Assessment after the initial access, one of the most common and well documented attacks a Pentester will try is Kerberoasting.  
Below we are going to explain how this attack works and possible mitigations.

## What is Kerberoasting

Kerberoasting abuses traits of the Kerberos protocol to harvest password hashes for Active Directory domain accounts with servicePrincipalName (SPN) values.

Every user of a domain (without administrative rights!) is able to request a ticket-granting ticket (TGT)  for any SPN, and parts of it may be encrypted with the RC4 using the password hash of the service account assigned the requested SPN as the key.

An attacker can extract TGT from memory or by sniffing the network, extract the service account’s password hash and attempt to crack the offline. By getting the TGT he can decide on which SPN’s the attack could work and which is not worth attacking because they are encrypted with more complex passwords.

## Performing the attack

At this point there are many ways that we can list all SPNs in a domain that use user accounts and are therefore suitable for Kerberoasting attacks:

We can use PowerView, GetUserSpn or build an LDAP filter to get the accounts with SPN values registered on the current domain

**GetUserSpn**

![](/images/posts/kerberosting/2.png)

**PowerView**

![](/images/posts/kerberosting/3.png)

**LDAP Filter**

![](/images/posts/kerberosting/4.png)

Once the attacker has determined the SPN to be attacked, he can use the following Powershell commands to issue a service ticket:

![](/images/posts/kerberosting/5.png)

Now, we have tickets in memory.  
We will use Mimikatz to export the tickets from memory.  
Running klist shows the new Kerberos service ticket.

![](/images/posts/kerberosting/6.png)

The issued ticket can be extracted from the memory using mimikatz:

![](/images/posts/kerberosting/7.png)

Now all the extracted Kerberos tickets (with the file extension .kirbi) should be in our directory.  
The .kirbi files can be converted into other formats like Hashcat or JohnTheRipper.  
We will use the tgsrepcrack.py from the [Kerberoast](https://github.com/nidem/kerberoast) project because it can brute-force .kirbi files without converting them.  
For better performance use Hashcat.

Let’s crack the TGS ticket

![](/images/posts/kerberosting/8.png)

This brute force attack happens offline, meaning no more communication with Active Directory needs to occur. As a result, this step is undetectable. An attacker may even exfiltrate the list of hashes and use a high-performance system designed just for password cracking. Tools such as hashcat are used to perform the offline brute force attack. Having obtained the plaintext password, the attacker can use the credentials to authenticate to any resources the service account has access to, potentially allowing them to compromise data or escalate privileges.

# Mitigations

As we’ve demonstrated, Kerberoasting can lead to the compromise of sensitive service account passwords. However, several mitigations exist that greatly reduce or even eliminate these risks.

- Service Account Passwords should be hard to guess and have a minimum of 25 characters and be routinely changed. That’s the effective mitigation for this attack.
- Use where’s possible Managed Service Accounts and Group Manage Service Accounts as it automates password changing and delegated SPN Management
- Audit the assignment of servicePrincipalNames to sensitive user accounts. For example, members of Domain Admins should not be used as service accounts (and therefore not have SPNs assigned).