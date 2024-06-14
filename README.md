# AI-hpc-install-config

## How to Make a Cluster Computer

Install a default mode Ubuntu

## How to Make a Cluster Computer

Open a terminal and execute on login node (master or head):

```
sudo apt update
sudo apt install openssh-server
sudo systemctl status ssh
sudo ufw allow ssh
```

### SSH Server, Hostnames, Passwordless Login among Nodes

Then create a Key:
```
ssh-keygen -t rsa
```

And them copy it to the nodes (compute or worker) using the following, from the master one:
```
ssh-copy-id login@node
```

If the environment has no DNS server, it will be necessary to update the /etc/hosts to conect on server using name.

At this part you can conect to and from any node og cluster without key password need.

## Installing/configure NFS for cluster computer

### On the master node (login or head)

Conect to the login node and install NFS service:
```
sudo apt-get install nfs-server
```

Now create the folder folder:
```
sudo mkdir /nfs
```

And them configure the exports adding:
```
/nfs *(rw,sync)
```

And them restar the service:
```
sudo service nfs-kernel-server restart
```

Confirm the creation and take owner:
```
ls -ld /nfs
sudo chown hashmi /nfs
```

### On the compute nodes (worker)

Conecet to compute node and execute: 
```
sudo apt-get install nfs-client
```

Create the dir:
```
sudo mkdir /nfs
```

Set it to be mounted pointing to master NFS share, automatically during the login:
```
sudo nano /etc/fstab
```
Add line:
```
HRG-Server:/nfs /nfs nfs
```
Now execute:
```
sudo systemctl daemon-reload
sudo mount -a
```

Confirm that NFS share is working properly creating a file using master and founding it using the compute node.

## Installing Slurm on Login Node/Head Node

Conect to the login node: 

```
export MUNGEUSER=1001
```

```
sudo groupadd -g $MUNGEUSER munge
```

```
sudo useradd -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge -s /sbin/nologin munge
```

```
export SLURMUSER=1002
```

```
sudo groupadd -g $SLURMUSER slurm
```

```
sudo useradd -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm -s /bin/bash slurm
```

```
sudo apt-get install -y munge
```

```
sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
```

```
sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
```

```
sudo scp /etc/munge/munge.key /nfs/slurm/
```

```
sudo systemctl enable munge
sudo systemctl start munge
```

Go to /nfs/slurm/ and them:

```
sudo apt-get install mariadb-server
```

```
sudo apt-get install slurmdbd
```

```
sudo apt-get install slurm-wlm
```

Now configure slurm and setup DB:

run `mysql` and run: 
```
grant all on slurm_acct_db.* TO 'slurm'@'localhost' identified by 'hashmi12' with grant option; 
create database slurm_acct_db;
exit
```

Them create:
```
mkdir /etc/slurm-llnl
```

Create a file on it:
```
sudo nano /etc/slurm-llnl/slurmdbd.conf
```

With the content:
```
AuthType=auth/munge
DbdAddr=localhost
#DbdHost=master0
DbdHost=localhost
DbdPort=6819
SlurmUser=slurm
DebugLevel=4
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/run/slurm/slurmdbd.pid
StorageType=accounting_storage/mysql
StorageHost=localhost
StorageLoc=slurm_acct_db
StoragePass=hashmi12
StorageUser=slurm
###Setting database purge parameters
PurgeEventAfter=12months
PurgeJobAfter=12months
PurgeResvAfter=2months
PurgeStepAfter=2months
PurgeSuspendAfter=1month
PurgeTXNAfter=12months
PurgeUsageAfter=12months
```

Now we need to give ownership of this file.
```
chown slurm:slurm /etc/slurm/slurmdbd.conf
chmod -R 600 slurmdbd.conf
```

Now lets ccreate the configuration file /etc/slurm/slurm.conf by visiting the website (https://slurm.schedmd.com/configurato...) to generate a slurm configuration file:
```
sudo nano /etc/slurm-llnl/slurm.conf
```

Allow the ports in firewall:
```
sudo ufw allow 6817
sudo ufw allow 6818
sudo ufw allow 6819
```

Continue executing:
```
mkdir /var/spool/slurmctld
chown slurm:slurm /var/spool/slurmctld
chmod 755 /var/spool/slurmctld
mkdir  /var/log/slurm
touch /var/log/slurm/slurmctld.log
touch /var/log/slurm/slurm_jobacct.log /var/log/slurm/slurm_jobcomp.log
chown -R slurm:slurm /var/log/slurm/
chmod 755 /var/log/slurm
```

## Search and change location of PID file

Search the pid file installation path:
```
find / -name "slurmctld.service"
find / -name "slurmd.service"
find / -name "slurmdbd.service"
```

Them change the default path by the founded ones, respectivly:
```
nano /usr/lib/systemd/system/slurmctld.service
nano /usr/lib/systemd/system/slurmdbd.service
nano /usr/lib/systemd/system/slurmd.service
```

Run the following as root:
```
echo CgroupMountpoint=/sys/fs/cgroup &gt &gt /etc/slurm-llnl/cgroup.conf
```

Confirm that its working by the command (shows details):
```
slurmd -C
```

Start and enable automatic SLURM Services on Login Node
```
systemctl daemon-reload
systemctl enable slurmdbd
systemctl start slurmdbd
systemctl enable slurmctld
systemctl start slurmctld
```

At this point see the status of the started services:
```
systemctl status slurmdbd
systemctl status slurmctld
```

## Installing Slurm on Computer Nodes/Worker Nodes

Confirm that SLURM is runing on master node:
```
systemctl status slurmctld
```
Now access slurm node and execute:

```
export MUNGEUSER=1001 
sudo groupadd -g $MUNGEUSER munge 
sudo useradd -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge -s /sbin/nologin munge 
export SLURMUSER=1002 
sudo groupadd -g $SLURMUSER slurm 
sudo useradd -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm -s /bin/bash slurm
sudo apt-get install -y munge
```

Now copy the munge authentication key from /nfs/slurm/ on every node:
```
sudo scp /nfs/slurm/munge.key /etc/munge/
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
sudo systemctl enable munge
sudo systemctl start munge
```

Start Installation of SLURM (On all compute nodes)
```
sudo apt-get install slurm-wlm
```

Copy both slurm.conf and slurmdbd.conf to each node at /etc/slurm
```
sudo scp /nfs/slurm/slurm.conf /etc/slurm
sudo scp /nfs/slurm/slurmdbd.conf /etc/slurm
```

On the compute nodes: (login as root and then run all the below commands till 3.5.10)
```
mkdir /var/spool/slurmd 
chown slurm: /var/spool/slurmd
chmod 755 /var/spool/slurmd
mkdir /var/log/slurm/
touch /var/log/slurm/slurmd.log
chown -R slurm:slurm /var/log/slurm/slurmd.log
chmod 755 /var/log/slurm
mkdir /run/slurm
touch /run/slurm/slurmd.pid
chown slurm /run/slurm
chown slurm:slurm /run/slurm
chmod -R 770 /run/slurm
```

Confirm the PIDFile path:
```
nano /usr/lib/systemd/system/slurmd.service
```

Then execute:
```
echo CgroupMountpoint=/sys/fs/cgroup &gt &gt /etc/slurm/cgroup.conf
```

Confirme the configuration using the command below, that shows config details:
```
slurmd -C
```

Finally:
```
systemctl enable slurmd.service 
systemctl start slurmd.service 
systemctl status slurmd.service
```

If the service is active, you are all good, otherwise just reboot the node and reconnect the NFS. Then check the status of slurmd.service. In any case, a reboot at this stage is necessary.

The following command can check the connectivity with the controller node:
```
scontrol ping
```

Execute `sinfo` in master node to confirm if nodes in slurm conf file are configurated properly.


## References
