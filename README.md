# AI-HPC-Install-Config

## How to Make a Cluster Computer

### Install Ubuntu

Install Ubuntu in default mode on all nodes.

### Setting Up the Master Node

1. **Open a terminal and execute on the login node (master or head):**
    ```sh
    sudo apt update
    sudo apt install openssh-server
    sudo systemctl status ssh
    sudo ufw allow ssh
    ```

2. **SSH Server, Hostnames, and Passwordless Login among Nodes**

   - Generate an SSH key:
     ```sh
     ssh-keygen -t rsa
     ```

   - Copy the key to the nodes (compute or worker) from the master:
     ```sh
     ssh-copy-id login@node
     ```

   - If there is no DNS server, update `/etc/hosts` to connect to the server using names.

   At this stage, you can connect to and from any node in the cluster without needing a password.

## Installing and Configuring NFS for Cluster Computer

### On the Master Node (Login or Head)

1. **Connect to the login node and install the NFS service:**
    ```sh
    sudo apt-get install nfs-server
    ```

2. **Create the NFS folder:**
    ```sh
    sudo mkdir /nfs
    ```

3. **Configure the NFS exports:**
    ```sh
    echo "/nfs *(rw,sync)" | sudo tee -a /etc/exports
    sudo systemctl restart nfs-kernel-server
    ```

4. **Set ownership of the NFS folder:**
    ```sh
    sudo chown $USER /nfs
    ```

### On the Compute Nodes (Worker)

1. **Connect to the compute node and install the NFS client:**
    ```sh
    sudo apt-get install nfs-client
    ```

2. **Create the NFS directory:**
    ```sh
    sudo mkdir /nfs
    ```

3. **Set up automatic mounting in `/etc/fstab`:**
    ```sh
    sudo nano /etc/fstab
    ```
    Add the line:
    ```sh
    HRG-Server:/nfs /nfs nfs defaults 0 0
    ```

4. **Mount the NFS directory:**
    ```sh
    sudo systemctl daemon-reload
    sudo mount -a
    ```

5. **Verify NFS share:**
    - Create a file on the master node and check if it is accessible from the compute node.

## Installing Slurm on Login Node/Head Node

### Create MUNGE and SLURM Users

1. **Create MUNGE user:**
    ```sh
    export MUNGEUSER=1001
    sudo groupadd -g $MUNGEUSER munge
    sudo useradd -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge -s /sbin/nologin munge
    ```

2. **Create SLURM user:**
    ```sh
    export SLURMUSER=1002
    sudo groupadd -g $SLURMUSER slurm
    sudo useradd -m -c "SLURM workload manager" -d /var/lib/slurm -u $SLURMUSER -g slurm -s /bin/bash slurm
    ```

3. **Install and configure MUNGE:**
    ```sh
    sudo apt-get install -y munge
    sudo chown -R munge: /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
    sudo chmod 0700 /etc/munge/ /var/log/munge/ /var/lib/munge/ /run/munge/
    sudo scp /etc/munge/munge.key /nfs/slurm/
    sudo systemctl enable munge
    sudo systemctl start munge
    ```

### Set Up MariaDB and Slurm

1. **Install MariaDB server:**
    ```sh
    sudo apt-get install mariadb-server
    ```

2. **Install Slurm:**
    ```sh
    sudo apt-get install slurmdbd slurm-wlm
    ```

3. **Configure MariaDB for Slurm:**
    ```sh
    mysql
    ```
    Run the following commands in the MariaDB shell:
    ```sql
    GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost' IDENTIFIED BY 'hashmi12' WITH GRANT OPTION;
    CREATE DATABASE slurm_acct_db;
    EXIT;
    ```

4. **Create Slurm configuration directory:**
    ```sh
    sudo mkdir /etc/slurm-llnl
    ```

5. **Create and configure `slurmdbd.conf`:**
    ```sh
    sudo nano /etc/slurm-llnl/slurmdbd.conf
    ```
    Add the following content:
    ```conf
    AuthType=auth/munge
    DbdAddr=localhost
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
    PurgeEventAfter=12months
    PurgeJobAfter=12months
    PurgeResvAfter=2months
    PurgeStepAfter=2months
    PurgeSuspendAfter=1month
    PurgeTXNAfter=12months
    PurgeUsageAfter=12months
    ```

6. **Set file permissions:**
    ```sh
    sudo chown slurm:slurm /etc/slurm-llnl/slurmdbd.conf
    sudo chmod 600 /etc/slurm-llnl/slurmdbd.conf
    ```

7. **Generate and configure `slurm.conf`:**
    - Visit [Slurm Configurator](https://slurm.schedmd.com/configurator.html) to generate a `slurm.conf` file.
    ```sh
    sudo nano /etc/slurm-llnl/slurm.conf
    ```

8. **Allow necessary ports through the firewall:**
    ```sh
    sudo ufw allow 6817
    sudo ufw allow 6818
    sudo ufw allow 6819
    ```

9. **Create and set permissions for Slurm directories:**
    ```sh
    sudo mkdir /var/spool/slurmctld
    sudo chown slurm:slurm /var/spool/slurmctld
    sudo chmod 755 /var/spool/slurmctld
    sudo mkdir /var/log/slurm
    sudo touch /var/log/slurm/slurmctld.log /var/log/slurm/slurm_jobacct.log /var/log/slurm/slurm_jobcomp.log
    sudo chown -R slurm:slurm /var/log/slurm
    sudo chmod 755 /var/log/slurm
    ```

### Configure and Start Slurm Services

1. **Search and update PID file locations:**
    ```sh
    sudo find / -name "slurmctld.service"
    sudo find / -name "slurmd.service"
    sudo find / -name "slurmdbd.service"
    ```
    Update the paths in the respective service files:
    ```sh
    sudo nano /usr/lib/systemd/system/slurmctld.service
    sudo nano /usr/lib/systemd/system/slurmdbd.service
    sudo nano /usr/lib/systemd/system/slurmd.service
    ```

2. **Add CgroupMountpoint to the configuration:**
    ```sh
    echo "CgroupMountpoint=/sys/fs/cgroup" | sudo tee -a /etc/slurm-llnl/cgroup.conf
    ```

3. **Check Slurm configuration:**
    ```sh
    slurmd -C
    ```

4. **Start and enable Slurm services:**
    ```sh
    sudo systemctl daemon-reload
    sudo systemctl enable slurmdbd
    sudo systemctl start slurmdbd
    sudo systemctl enable slurmctld
    sudo systemctl start slurmctld
    ```

5. **Check the status of the services:**
    ```sh
    sudo systemctl status slurmdbd
    sudo systemctl status slurmctld
    ```

## Installing Slurm on Compute Nodes (Worker Nodes)

1. **Confirm that Slurm is running on the master node:**
    ```sh
    sudo systemctl status slurmctld
    ```

2. **Access each compute node and set up MUNGE and SLURM users:**
    ```sh
    export MUNGEUSER=1001
    sudo groupadd -g $MUNGEUSER munge
    sudo useradd -m -c "MUNGE Uid 'N' Gid Emporium" -d /var/lib/munge -u $MUNGEUSER -g munge -s /sbin/nologin munge
    export SLURMUSER=1002
    sudo groupadd -g $SLURMUSER slurm
    sudo useradd -m -c "SLURM workload

 manager" -d /var/lib/slurm -u $SLURMUSER -g slurm -s /bin/bash slurm
    sudo apt-get install -y munge
    ```

3. **Copy the MUNGE key to each compute node:**
    ```sh
    sudo scp /nfs/slurm/munge.key /etc/munge/
    sudo chown munge:munge /etc/munge/munge.key
    sudo chmod 400 /etc/munge/munge.key
    sudo systemctl enable munge
    sudo systemctl start munge
    ```

4. **Install Slurm on all compute nodes:**
    ```sh
    sudo apt-get install slurm-wlm
    ```

5. **Copy Slurm configuration files to each node:**
    ```sh
    sudo scp /nfs/slurm/slurm.conf /etc/slurm
    sudo scp /nfs/slurm/slurmdbd.conf /etc/slurm
    ```

6. **Set up Slurm directories and permissions:**
    ```sh
    sudo mkdir /var/spool/slurmd
    sudo chown slurm: /var/spool/slurmd
    sudo chmod 755 /var/spool/slurmd
    sudo mkdir /var/log/slurm
    sudo touch /var/log/slurm/slurmd.log
    sudo chown -R slurm:slurm /var/log/slurm/slurmd.log
    sudo chmod 755 /var/log/slurm
    sudo mkdir /run/slurm
    sudo touch /run/slurm/slurmd.pid
    sudo chown slurm /run/slurm
    sudo chown slurm:slurm /run/slurm
    sudo chmod -R 770 /run/slurm
    ```

7. **Confirm the PIDFile path:**
    ```sh
    sudo nano /usr/lib/systemd/system/slurmd.service
    ```

8. **Add CgroupMountpoint to the configuration:**
    ```sh
    echo "CgroupMountpoint=/sys/fs/cgroup" | sudo tee -a /etc/slurm/cgroup.conf
    ```

9. **Check Slurm configuration:**
    ```sh
    slurmd -C
    ```

10. **Enable and start Slurm services on compute nodes:**
    ```sh
    sudo systemctl enable slurmd.service
    sudo systemctl start slurmd.service
    sudo systemctl status slurmd.service
    ```

11. **If the service is not active, reboot the node and reconnect the NFS. Then check the status of `slurmd.service`.

12. **Check connectivity with the controller node:**
    ```sh
    scontrol ping
    ```

13. **Verify node configuration from the master node:**
    ```sh
    sinfo
    ```

## References

- [Slurm Documentation](https://slurm.schedmd.com/documentation.html)
- [Ubuntu Server Guide](https://ubuntu.com/server/docs)
