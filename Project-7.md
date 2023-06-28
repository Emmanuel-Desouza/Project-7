# PROJECT 7: DEVOPS TOOLING WEBSITE SOLUTION

## STEP 1 – PREPARE NFS SERVER

### Spin up a new EC2 instance with RHEL Linux 8 Operating System.

### Based on your LVM experience from Project 6, Configure LVM on the Server.

![configuring LVM](./images/lv1.png)

![configuring LVM](./images/xvdf1.png)
![configuring LVM](./images/xvdf2.png)
![configuring LVM](./images/xvdg1.png)
![configuring LVM](./images/xvdg2.png)
![configuring LVM](./images/xvdh1.png)
![configuring LVM](./images/xvdh2.png)

```
sudo pvcreate /dev/xvdf1
sudo pvcreate /dev/xvdg1
sudo pvcreate /dev/xvdh1
```
![configuring LVM](./images/lv2.png)
![configuring LVM](./images/lv3.png)
![configuring LVM](./images/lv4.png)


### Instead of formating the disks as ext4, I will format them as xfs

```
sudo mkfs.xfs -f /dev/xvdf
sudo mkfs.xfs -f /dev/xvdg
sudo mkfs.xfs -f /dev/xvdh
```

![formatting to xfs](./images/xvdf-xfs.png)
![formatting to xfs](./images/xvdg-xfs.png)
![formatting to xfs](./images/xvdh-xfs.png)

### Ensure there are 3 Logical Volumes. lv-opt lv-apps, and lv-logs

```
sudo lvcreate -n lv-opt -L 9G webdata-vg
sudo lvcreate -n lv-apps -L 9G webdata-vg
sudo lvcreate -n lv-logs -L 9G webdata-vg
```
`sudo lvs`
![confirming LVs](./images/sudo-lvs.png)

![verifying setup LVs](./images/verify1.png)
![verifying setup LVs](./images/verify2.png)
![verifying setup LVs](./images/verify3.png)
![verifying setup LVs](./images/verify4.png)

```
sudo mkfs.xfs -f /dev/webdata-vg/lv-opt
sudo mkfs.xfs -f /dev/webdata-vg/lv-apps
sudo mkfs.xfs -f /dev/webdata-vg/lv-logs
```

![formatting LVs to xfs](./images/xfs.png)

### Create mount points on /mnt directory for the logical volumes as follow:

- Mount lv-apps on /mnt/apps – To be used by webservers
- Mount lv-logs on /mnt/logs – To be used by webserver logs
- Mount lv-opt on /mnt/opt – To be used by Jenkins server in Project 8

```
sudo mkdir -p /mnt/apps
sudo mount /dev/webdata-vg/lv-apps /mnt/apps
```

```
sudo mkdir -p /mnt/logs
sudo mount /dev/webdata-vg/lv-logs /mnt/logs
```

```
sudo mkdir -p /mnt/opt
sudo mount /dev/webdata-vg/lv-opt /mnt/opt
```

![mounting LVs](./images/mkdir-mount.png)
### Install NFS server, configure it to start on reboot and make sure it is up and running

```
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```

![NFS Server installation](./images/yumupdate.png)
![NFS Server Installation](./images/yuminstalutils.png)
![Configuring NFS to start on reboot](./images/startenablestatus.png)
### Export the mounts for webservers’ subnet cidr to connect as clients. For simplicity, I will install all three Web Servers inside the same subnet, but in production set up I would probably want to separate each tier inside its own subnet for higher level of security.

### To check subnet cidr – I will open EC2 details in AWS web console and locate ‘Networking’ tab and open a Subnet link:

### Making sure I set up permission that will allow my Web servers to read, write and execute files on NFS:

```
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```

![permissions](./images/chown.png)
### Configuring access to NFS for clients within the same subnet (example of Subnet CIDR – 172.31.32.0/20 ):

```
sudo vi /etc/exports

/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)

Esc + :wq!

sudo exportfs -arv
```
![configuring access to NFS](./images/cidrexport.png)
### To check which port is used by NFS and open it using Security Groups (add new Inbound Rule)
`rpcinfo -p | grep nfs`

![Checking Port used by NFS](./images/portnfs.png)
## Important note: In order for NFS server to be accessible from the client, I must also open following ports: TCP 111, UDP 111, UDP 2049

![Opening Ports](./images/inboundrules.png)
## STEP 2 — CONFIGURE THE DATABASE SERVER

### Install MySQL server
`sudo apt update`
![apt update](./images/sudoaptupdate.png)

`sudo apt install mysql-server`
![Install mysql Server](./images/install-mysqlserver.png)

### To configure MySQL server to allow connections from remote hosts.

`sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf`

![Install mysql Server](./images/mysqld.cnf.png)

![mysql to remote host](./images/0000.png)

### Create a database and name it 'tooling'
`sudo mysql`
`CREATE DATABASE tooling;`

![mysql to remote host](./images/createtooling.png)

### Create a database user and name it 'webaccess'
`CREATE USER 'webaccess'@'172.31.0.0/20' IDENTIFIED BY 'webaccess';`
![creating user webaccess](./images/createuser.png)

### Grant permission to webaccess user on tooling database to do anything only from the webservers subnet cidr
`GRANT ALL ON tooling.* TO 'webaccess'@'172.31.0.0/20';`
`FLUSH PRIVILEGES;`
`SHOW DATABASES;`

![grant all privileges](./images/grantall.png)

## Step 3 — Prepare the Web Servers

### We need to make sure that our Web Servers can serve the same content from shared storage solutions, in our case – NFS Server and MySQL database.
### You already know that one DB can be accessed for reads and writes by multiple clients. For storing shared files that our Web Servers will use – we will utilize NFS and mount previously created Logical Volume lv-apps to the folder where Apache stores files to be served to the users (/var/www).

### This approach will make our Web Servers stateless, which means we will be able to add new ones or remove them whenever we need, and the integrity of the data (in the database and on NFS) will be preserved.

### During the next steps we will do following:

### Configure NFS client (this step must be done on all three servers)
### Deploy a Tooling application to our Web Servers into a shared NFS folder
### Configure the Web Servers to work with a single MySQL database
### Launch a new EC2 instance with RHEL 8 Operating System
### Install NFS client
`sudo yum install nfs-utils nfs4-acl-tools -y`
![Installing NFS Client](./images/acl-tools.png)