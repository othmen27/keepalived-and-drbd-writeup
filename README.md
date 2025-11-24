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

### 1 - Preparation

What I did here is I set up both computers to have a NAT network connection in the same network.
And then I basically set up a custom network adapter in order for the DRBD to run and operate smoothly like shown in the screenshot below:

![Custom network adapter image](screenshots/first.png)

After that i added that network adapter to both computers like shown below:

![Network adapter](screenshots/third.png)
![Network adapter2](screenshots/second.png)

After that i checked if both interfaces are up and running by simply using **ip a**

![Interface1](screenshots/forth.png)
![Interface2](screenshots/fifth.png)

Then i simply checked the connectivity between computers by simply running **ping (ipaddress)**

![Ping](screenshots/seventh.png)
