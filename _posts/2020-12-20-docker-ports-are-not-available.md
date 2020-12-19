---
title: "# Docker - ports are not available"
excerpt_separator: "<!--more-->"
categories:
  - Post Formats
tags:
  - Post Formats
  - readability
  - standard
last_modified_at: 2020-12-20T00:31:00+01:00
---

# Docker - ports are not available

I remember the times when as a developer I had a VM that held a database, an application server, a web server, and whatever else was needed to run the application I was working on. What a productivity booster, I thought, compared to the olden days where I had to install everything on my host OS. No more! Need to fix a bug, while working on a feature? Just spin up a new VM, switch to new a branch, and off you go.

Moving on to Docker has made me recall that memory. With Docker, I got yet another productivity power-up. No need to run a [heavyweight VM](https://cloudacademy.com/blog/docker-vs-virtual-machines-differences-you-should-know/) with an extra layer of virtualization in form of a guest OS, just start and stop containers as needed. These days, I'm using [Docker Desktop](https://www.docker.com/products/docker-desktop) on Windows 10 to run Linux containers on WSL2. And the container I run the most often is Microsoft SQL Server 2019. For Linux. Who would have thought!

A few days ago I started seeing an error when attempting to run the MSSQL container. I thought perhaps it had to do with the [Windows 10 20H2 update](https://docs.microsoft.com/en-us/windows/whats-new/whats-new-windows-10-version-20h2) (build 19042), but I came across other Docker users reporting the same issue even before the update. The error complained about Docker not being able to bind TCP port 1433, which is the default port MSSQL listens on:
```
> docker run -d -p 1433:1433 --name mssql --restart unless-stopped mssql
docker: Error response from daemon: Ports are not available: listen tcp 0.0.0.0:1433: bind: 
An attempt was made to access a socket in a way forbidden by its access permissions.
```

The first thing I did was check which process might be occupying the port:
```
> netstat -abno -p TCP | find '"1433"'
```
(`find` requires the search string to be enclosed in double-quotes. When piping the result of `netstat` to `find` in PowerShell, wrap the whole search string including double-quotes in single-quotes)

Surprise, the port was not in use by another process. However it was _reserved_, that is, blocked for use by another process when that process might need it in the future:

```
> netsh int ipv4 show excludedportrange protocol=tcp

Protocol tcp Port Exclusion Ranges

Start Port    End Port
----------    --------
        80          80
      1433        1433
      2869        2869
      5357        5357
      9001        9001
     51563       51662
     51755       51854
     51855       51954
     52563       52662
     52758       52857
     52858       52957
     53414       53513
     53614       53713
     53814       53913
     54019       54118
     54219       54318
     54419       54518
     54738       54837
     54871       54970
     54971       55070
     55071       55170
     55171       55270
     55271       55370

* - Administered port exclusions.
```

Since no process was actively using the port, my first thought was to delete the exclusion for the one port I needed:
```
> netsh int ipv4 delete excludedportrange protocol=tcp startport=1433 numberofports=1
Access is denied.
```

The permission error signaled that while there might not be a process listening on that port right now, the process is currently running and preventing me from removing its port reservation. 

System services are known to reserve ports for their use. The most common suspect that kept turning up in my research was [Hyper-V](https://stackoverflow.com/questions/48478869/cannot-bind-to-some-ports-due-to-permission-denied). A [suggested workaround to disable Hyper-V](https://blog.sixthimpulse.com/2019/01/docker-for-windows-port-reservations/), delete the exclusion and enable Hyper-V did not help in my case.

After reading through [a similar report on superuser on StackExchange](https://superuser.com/questions/1579346/many-excludedportranges-how-to-delete-hyper-v-is-disabled) I confirmed that what was actually happening in my case was that WinNAT (Windows NAT) had been holding on to my port. It doesn't come as that much of a surprise, when we realize what [role WinNAT plays in virtual networking](https://techcommunity.microsoft.com/t5/virtualization/windows-nat-winnat-capabilities-and-limitations/ba-p/382303):
> Similar to how the NAT translates the source IP address of a device, NAT can also translate the source IP address of a VM or container to the IP address of a virtual network adapter (Host vNIC) on the host. 

To resolve the issue, I had to stop WinNAT, reserve the port for myself, and restart WinNAT.

```
> net stop winnat
The Windows NAT Driver service was stopped successfully.

> netsh int ipv4 add excludedportrange protocol=tcp startport=1433 numberofports=1
Ok.

> net start winnat
The Windows NAT Driver service was started successfully.
 
> netsh int ipv4 show excludedportrange protocol=tcp

Protocol tcp Port Exclusion Ranges

Start Port    End Port
----------    --------
        80          80
      1433        1433     * (My precious!!!)
      5357        5357
      9001        9001
 
* - Administered port exclusions.
```

As you can see from the output, port 1433 is now an "Administered port exclusion" which will prevent Windows services from using or reserving the port. After this change, I was then able to run the MSSQL Docker container as usual.