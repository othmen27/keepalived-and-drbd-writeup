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

### 2 - Setting up Keepalived

We're gonna start by downloading the **keepalived** package then modify the folder /etc/keepalived/keepalived.conf into this config:

```
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 101
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass secret123
    }

    virtual_ipaddress {
        192.168.47.100/24
    }

    notify_master "/etc/keepalived/master.sh"
    notify_backup "/etc/keepalived/backup.sh"
    notify_fault "/etc/keepalived/backup.sh"
    notify_stop "/etc/keepalived/backup.sh"
}

```
> Note this config is for the main node as we need to change the priority to a smaller number for the secondary like this 

```
vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 51
    priority 100
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass secret123
    }

    virtual_ipaddress {
        192.168.47.100/24
    }

    notify_master "/etc/keepalived/master.sh"
    notify_backup "/etc/keepalived/backup.sh"
    notify_fault "/etc/keepalived/backup.sh"
    notify_stop "/etc/keepalived/backup.sh"
}
```
After adding this config to each node we can then create our shell scripts they are really simple.

master.sh :

```sh
#!/bin/bash

/bin/echo "$(date) - Devenu MASTER" >> /var/log/keepalived-transitions.log

/bin/sleep 2

/sbin/drbdadm primary r0

/bin/mount /dev/drbd0 /mnt/r0

/bin/echo -e "Subject: Keepalived Failover - MASTER\n\nNode $(hostname) is now MASTER." \
    | msmtp --from=gmail mariojabri1@gmail.com
```

backup.sh:

```sh
#!/bin/bash

/bin/echo "$(date) - Devenu MASTER" >> /var/log/keepalived-transitions.log

/bin/umount /dev/drbd0

/bin/sleep 2

/sbin/drbdadm secondary r0

/bin/echo -e "Subject: Keepalived Failover - BACKUP\n\nNode $(hostname) is now BACKUP." \
    | msmtp --from=gmail mariojabri1@gmail.com

```

then make the executable by running **chmod +x /etc/keepalived/script.sh** with script being either master or backup.
Before we jump into the testing we gotta setup the SMTP to send an email it's really simple.
First we gotta install the msmtp and msmtp-mta packages then modify the **~/.msmtprc** for user specific or **/etc/msmtprc** for global uses in my case my config looks like this:

```
# Set default account
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        ~/.msmtp.log

# Gmail account
account        gmail
host           smtp.gmail.com
port           587
from           EMAIL@EMAIL.com
user           EMAIL@EMAIL.com
password       APP_PASSWORD_HERE
```
> You gotta use the APP password that you get from your google's account from security pannel.

Now we can jump into testing finally let's enable the keepalived service on startup by running **systemctl enable keepalived** then start it **systemctl start keepalived** (ON BOTH NODES).
we can now test if the MASTER has the virtual ip in my case 192.168.47.100, let's start by pinging the ip from the MASTER node

![ping](screenshots/24.png)

Good it works let's check the network interface and see if we have the IP on the MASTER node by running **ip a** like below:

![vipIp](screenshots/25.png)

Good it's there now we can test by stopping the service from the first node by running **systemctl stop keepalived** and see if the 2nd node became the master or not.

**First node :**

![firstNOde](screenshots/26.png)

**Second node :**

![secondNode](screenshots/27.png)

**Emails recieved :**

![emails](screenshots/28.png)

### 3 - Conclusion

This project demonstrated the setup of a 2-node High Availability cluster using DRBD and Keepalived. DRBD provided real-time block-level replication, ensuring data consistency between nodes, while Keepalived managed a virtual IP for automatic failover, maintaining service continuity.

Through hands-on configuration, I verified the replication by creating and accessing files across nodes and tested failover scenarios with seamless transitions. Integrating Nginx allowed visual confirmation of the active node, and msmtp enabled automated email notifications for state changes.

Overall, the project provided practical experience in HA clustering, network configuration, and Linux system administration, highlighting how DRBD and Keepalived can be combined to build a reliable and resilient infrastructure for critical services.
