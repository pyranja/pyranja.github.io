---
layout: post
title: Clean up Vagrant SMB shares
---

Currently Vagrant does not remove SMB shares, when destroying a VM. But there is of course an elegant way to clean up left over shares with Powershell :

```
Get-SmbShare | Where-Object -Property Name -Match  '.*[^\$]$' | Remove-SmbShare
```
