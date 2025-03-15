# kubernetes
kubernetes



# HA NFS

Objective:
This document explains how to set up a highly available NFS cluster using two NFS servers:

nfs-server-1: 192.168.100.130 → MASTER

nfs-server-2: 192.168.100.129 → BACKUP


VIP: 192.168.100.200 → The IP address used by Kubernetes and clients to access NFS.
 Tools Used:

NFS for shared storage
rsync for automatic synchronization
Keepalived for VIP and failover management

# 1: Install Dependencies on Both Servers
 Install NFS, rsync, and Keepalived:
```bash

sudo apt update && sudo apt install -y nfs-kernel-server rsync keepalived

```
 Create NFS Directory and Set Permissions:
```bash
sudo mkdir -p /srv/nfs4
sudo chmod 777 /srv/nfs4
```

# 2: Configure NFS on Both Servers

Edit the /etc/exports file and add the NFS path:

```bash

sudo vi /etc/exports

```

Add this line on both nfs-server-1 and nfs-server-2:

```bash

/srv/nfs4 192.168.100.0/24(rw,sync,no_root_squash,no_subtree_check)

```

#  Apply changes and restart NFS:
```bash

sudo exportfs -r
sudo systemctl restart nfs-kernel-server
```

# configure rsync for NFS Synchronization
 Create an SSH Key and enable passwordless authentication between servers:

```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id aress@nfs-server-2
ssh-copy-id aress@nfs-server-1
```
Run an initial rsync from nfs-server-1 to nfs-server-2:
```bash
rsync -avz --delete --numeric-ids --perms --owner --group --rsync-path="sudo rsync" /srv/nfs4/ aress@nfs-server-2:/srv/nfs4/

```
# 4: Configure Keepalived for VIP Management
 1. Configure Keepalived on nfs-server-1 (MASTER)
 Edit /etc/keepalived/keepalived.conf:

```bash

sudo vi /etc/keepalived/keepalived.conf

```
Content:

```bash


vrrp_instance NFS_HA {
    state MASTER
    interface ens34
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        192.168.100.200
    }
    notify_master "/usr/local/bin/sync-forward.sh"
}


```
 Restart Keepalived on nfs-server-1:

 ```bash
 sudo systemctl restart keepalived
sudo systemctl enable keepalived

 ```


 # 2-Configure Keepalived on nfs-server-2 (BACKUP)

  Edit /etc/keepalived/keepalived.conf:
```bash

sudo vi /etc/keepalived/keepalived.conf

 ```

  Content:

   ```bash
   
vrrp_instance NFS_HA {
    state BACKUP
    interface ens34
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    virtual_ipaddress {
        192.168.100.200
    }
    notify_master "/usr/local/bin/sync-back.sh"
}
```

Restart Keepalived on nfs-server-2:

```
sudo systemctl restart keepalived
sudo systemctl enable keepalived
```

# 5: Run rsync on Failover and Failback


 Create the script /usr/local/bin/sync-back.sh on nfs-server-2:

 ```bash
sudo vi /usr/local/bin/sync-back.sh
```

```bash

#!/bin/bash
echo "$(date) - Syncing from nfs-server-2 to nfs-server-1" >> /var/log/rsync-failover.log
rsync -avz --delete --numeric-ids --perms --owner --group --rsync-path="sudo rsync" /srv/nfs4/ aress@nfs-server-1:/srv/nfs4/
``
 Make the script executable:
``bash
sudo chmod +x /usr/local/bin/sync-back.sh
``

Create the script /usr/local/bin/sync-forward.sh on nfs-server-1:

``bash

 sudo vi /usr/local/bin/sync-forward.sh
``

 Content:
``bash
#!/bin/bash
echo "$(date) - Syncing from nfs-server-1 to nfs-server-2" >> /var/log/rsync-failover.log
rsync -avz --delete --numeric-ids --perms --owner --group --rsync-path="sudo rsync" /srv/nfs4/ aress@nfs-server-2:/srv/nfs4/

```

 Make the script executable:

 ```bash
sudo chmod +x /usr/local/bin/sync-forward.sh
 ```











