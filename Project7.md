## PROJECT 7: 

### **STEP 1 — PREPARING THE NFS SERVER**

First, I launched five EC2 instances: four Red Hat (one NFS server and three web servers) and one Ubuntu server  (to server as the database server).


Next, I created 3 volumes in the same AZ (Availability Zone) as my NFS Server, each of 10 GiB and respectively name nfs1, nfs2 and nfs3.

Next, I spinned up the Red Hat EC2 instance and ran the command below:

`lsblk`

![](./Images/lsblk.PNG)

To see all (available) mount points and free space on my server, I ran this command:

`df -h`

![](./Images/df-h.PNG)


Next, I used the *gdisk* utility to create a single partition on each of the 3 disks, beginning with the first volume, using the command below:

`sudo gdisk /dev/xvdf`

I repeated the last step for the *xvdg disk*, as below:

`sudo gdisk /dev/xvdg`

Similarly, the previous step was repeated the *xvdh disk*, as below:

`sudo gdisk /dev/xvdh`

Next, I used the `lsblk` utility to view the newly configured partition on each of the 3 disks. This is displayed below:


![](./Images/lsblk-utiliy.PNG)


Next, I installed *lvm2* package using sudo yum install lvm2 by running the command below

`sudo yum install lvm2 -y`


To check for available partitions, I ran the command below:

`sudo lvmdiskscan`

The output is shown below:

![](./Images/discscan.PNG)


Next, I used the *pvcreate* utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM

`sudo pvcreate /dev/xvdf1 /dev/xvdg1 /dev/xvdh1`

To verify that the Physical volume has been created successfully, I ran this command:

`sudo pvs`

![](./Images/pvcreate.PNG)

Next, I used the  *vgcreate utility* to add all 3 PVs to a volume group (VG). The VG was named *webdata-vg*

`sudo vgcreate webdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1`


To verify that this was done successfully, I ran the command:

`sudo vgs`

![](./Images/vgs.PNG)

Next, I used the *lvcreate utility* to create 3 logical volumes of equal sizes (9G each) named **lv-opt**, **lv-apps** and **lv-logs**, with the commands below:

`sudo lvcreate -n lv-opt -L 9G webdata-vg`

`sudo lvcreate -n lv-apps -L 9G webdata-vg`

`sudo lvcreate -n lv-logs -L 9G webdata-vg`

To verify that the Logical Volumes has been created successfully, i ran the command below:

`sudo lvs`


![](./Images/lvs.PNG)

`sudo vgdisplay -v #view complete setup - VG, PV, and LV`


`sudo lsblk`

Next, I used *mkfs -t xfs* to format the logical volumes with xfs filesystem, using the commands below:

`sudo mkfs -t xfs /dev/webdata-vg/lv-opt`

`sudo mkfs -t xfs /dev/webdata-vg/lv-apps`

`sudo mkfs -t xfs /dev/webdata-vg/lv-logs`


Next, I created mount points on /mnt directory for the logical volumes as follow:

    Mount lv-apps on /mnt/apps – To be used by webservers

    Mount lv-logs on /mnt/logs – To be used by webserver logs

    Mount lv-opt on /mnt/opt – To be used by Jenkins server in Project 8

 `sudo mkdir /mnt/apps`

 `sudo mkdir /mnt/logs`

 `sudo mkdir /mnt/opt`

Next, I mounted lv-apps on /mnt/apps

`sudo mount /dev/webdata-vg/lv-apps /mnt/apps`

Next, I mounted lv-logs on /mnt/logs

`sudo mount /dev/webdata-vg/lv-logs /mnt/logs`

Next, I mounted lv-opt on /mnt/opt

`sudo mount /dev/webdata-vg/lv-apps /mnt/apps`



Next, I installed NFS server, configured it to start on reboot and made sure it is up and running:

`sudo yum -y update`

`sudo yum install nfs-utils -y`

`sudo systemctl start nfs-server.service`

`sudo systemctl enable nfs-server.service`

`sudo systemctl status nfs-server.service`

![](./Images/active-nfs-server.PNG)


Next, xxport the mounts for webservers’ subnet cidr to connect as clients:

`sudo chown -R nobody: /mnt/apps`

`sudo chown -R nobody: /mnt/logs`

`sudo chown -R nobody: /mnt/opt`

`sudo chmod -R 777 /mnt/apps`

`sudo chmod -R 777 /mnt/logs`

`sudo chmod -R 777 /mnt/opt`

`sudo systemctl restart nfs-server.service`


Configure access to NFS for clients within the same subnet

Using a text edior live Vim;

`sudo vi /etc/exports`

Paste in the code below (format accordingly):

    /mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)

    /mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)

    /mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)

![](./Images/nfs-subnet.PNG)
    

`sudo exportfs -arv`

![](./Images/export.PNG)


Next, Check which port is used by NFS and open it using Security Groups (add new Inbound Rule):

`rpcinfo -p | grep nfs`

![](./Images/ports.PNG)


Important note: In order for NFS server to be accessible from your client, you must also open following ports: TCP 111, UDP 111, UDP 2049




![](./Images/port-opening.PNG)











___
### **STEP 2: CONFIGURING THE DATABASE SERVER**
___
On the database server

Install MYSQL server


`sudo apt update`

`sudo apt install mysql-server -y`


Verify that the service is up and running by using

`sudo systemctl status mysql`

![](./Images/mysql.PNG)


Create a database and name it **tooling**

`sudo mysql`

`CREATE DATABASE tooling;`

Create a database user and name it **webaccess**

Grant permission to **webaccess** user on **tooling** database to do anything only from the webservers subnet cidr

`CREATE USER 'webaccess'@'subnet-cidr' IDENTIFIED BY `password`:`

`GRANT ALL PRRIVILEGES ON tooling.* TO 'webaccess'@'subnet-cidr';`

`FLUSH PRIVILEGES`

`SHOW DATABASES;`

![](./Images/database.PNG)

`exit`



