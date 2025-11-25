# DRBD & Keepalived

This is a writeup for a DRBD & Keepalived project by my professor Mr Noureddine grassa.
it implements a simple **2-node High Availability (HA) cluster** using:
- **DRBD** for real-time block-level replication
- **Keepalived** (VRRP) for Virtual IP failover
- **Nginx** to show which node is currently active and to share local files

(Project link)[http://n.grassa.free.fr/TD_admin/TP_DRBD_KEEPALIVED.pdf]

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
- server1
- server2

![Added user](screenshots/8.png)

Then i rename the hosts like this:
- drbd-node1
- drbd-node2

![Hostname](screenshots/9.png)

After renaming the hosts i proceeded with giving each node a static ip address by modifying the contents of **/etc/netplan/**

![firstIp](screenshots/10.png)

![secondIp](screenshots/11.png)

Then i ran the command **netplan apply** in order of updating the IPs.
Afterwards i added the domain of each computer in the other's **/etc/hosts** like shown below:

![firstDomain](screenshots/12.png)
![secondDomain](screenshots/13.png)

After all that is complete we can finally take our first step into the project.

---

## 1 - Setting up DRBD

Here i started with simply formatting the disk **sbd** by running the command **sudo fdisk /dev/sdb** then i followed the guide shown in the document provided by my professor by pressing n->p->l->enter->enter->w.
Afterwards i installed the package drbd-utils by simply running **apt install -y drbd-utils**.
Once the download is complete i run **modprobe drbd** to load the module DRBD. and verified if it's running by running **lsmod | grep drbd** like shown below:

![lsmod](screenshots/14.png)

So knowing that the module is up and running we can now start by configuring the **.conf** files, i run the following command **vim /etc/drbd.d/global_common.conf** and added the following 
```
global {
    usage - count no ;
}
common {
    net {
        protocol C;
    }
}
```
**On both nodes**

like this:

![conf1](screenshots/15.png)
![conf2](screenshots/16.png)

And then i created **r0.res** file inside **/etc/drbd.d/** directory and added the following content:

```
resource r0 {
    on drbd-node1 {
        device      /dev/drbd0;
        disk        /dev/sdb1;
        address     10.0.0.1:7789;
        meta-disk   internal;
    }
    on drbd-node2 {
        device      /dev/drbd0;
        disk        /dev/sdb1;
        address     10.0.0.2:7789;
        meta-disk   internal;
    }
}
```
**I did this on both nodes**

After I was done with editing the **conf** files i run the following command on both nodes: **drbdadm create-md r0** to create the metadata for the DRBD with the name r0. And then I run **drbdadm up r0** on both nodes in order to activate the resource. And i checked the state by running 
**drbdadm status r0** like shown below

![drState1](screenshots/17.png)
![drState2](screenshots/18.png)

We notice from both screenshots that both nodes are secondary we're gonna fix that by simply running the following command 
**drbdadm primary --force r0** and verify again

![drState3](screenshots/19.png)

Good now that it's "primary" we can move forward by formatting the device /dev/drbd0 by running **mkfs.ext4 /dev/drbd0** and mounting it on the folder we created **/mnt/r0/** by simply running **mount /dev/drbd0 /mnt/r0/** and we can check if it's not mounted or not by running **df -h | grep drbd** like shown below:

![mounted](screenshots/20.png)

>Note that this step is only done in the first node.

Now we're gonna try and test the drbd by creating random files and changing the primary node to the 2nd node.
Here we created the following files:

![files](screenshots/21.png)

And then we're gonna unmount the device and changing the current node to a secondary by using the following command **drbdadm secondary r0** and test if we can find the current files in the 2nd node by mounting the device and changing the role to primary like shown below:

![files2](screenshots/22.png)

We can see that the files have been transferred over to the second node.

![files3](screenshots/23.png)
