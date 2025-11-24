# DRBD & Keepalived

This is a writeup for a DRBD & Keepalived project by my professor Mr Noureddine grassa.
it implements a simple **2-node High Availability (HA) cluster** using:
- **DRBD** for real-time block-level replication
- **Keepalived** (VRRP) for Virtual IP failover
- **Nginx** to show which node is currently active and to share local files

---

## What I did

> This section documents the exact steps I performed to build the DRBD + Keepalived HA cluster.
> For each step I include the commands I ran, what to check, and a placeholder for a screenshot.

### 0 - Preparation

What I did here is I set up both computers to have a NAT network connection in the same network.
And then I basically set up a custom network adapter in order for the DRBD to run and operate smoothly like shown in the screenshot below:

![Custom network adapter image](screenshots/first.png)

After that I added that network adapter to both computers like shown below:

![Network adapter](screenshots/third.png)
![Network adapter2](screenshots/second.png)

After that I checked if both interfaces are up and running by simply using **ip a**

![Interface1](screenshots/forth.png)
![Interface2](screenshots/fifth.png)

Then I simply checked the connectivity between computers by simply running **ping (ipaddress)**

![Ping](screenshots/seventh.png)

Next I made new user with the following names:
-server1
-server2

![Added user](screenshots/8.png)

Then i rename the hosts like this:
-drbd-node1
-drbd-node2

![Hostname](screenshots/9.png)

After renaming the hosts i proceeded with giving each node a static ip address by modifying the contents of **/etc/netplan/**

![firstIp](screenshots/10.png)

![secondIp](screenshots/11.png)

Then i ran the command **netplan apply** in order of updating the IPs.
Afterwards i added the domain of each computer in the other's **/etc/hosts** like shown below:

![firstDomain](screenshots/12.png)
![secondDomain](screenshots/13.png)

After all that is complete we can finally take our first step into the project.

## 1 - Setting up DRBD
