# Proxmox-Documentation
Proxmox documentation from CISA grad project 

Setting up Proxmox

Download the iso from the proxmox website
https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso 
Download rufus
Insert a USB drive into the pc
Format the USB drive using Rufus into a bootable Proxmox drive
Install proxmox
When the installation on the SSD prompt appears, click the options button 
Change the size to less than the full size of the SSD (we are doing that so that we can setup ceph later)
Proceed with the installation; change the hostname into an FQDN (example: group12.ca)
Complete the installation
Do this 3 times

Partition the disks for ceph

Go to each pc
Login using root and the password that you have set up
Using the following “lsblk”
Verify the output has space to be partitioned
Use fdisk /dev/sda
Enter “n” for new partition
Press Enter, enter, + ( amount of GBs for the ceph partitions ) eg (+100G)
Enter w to save changes

Connecting to your Proxmox servers

The IP will be listed on the initial screen of the Proxmox server
Enter the IP and port in a web browser on the same network (eg https://192.168.1.1:8006) 

Creating a cluster in Proxmox

On the manager node (the node you want to control every other computer)
Click the datacenter > Cluster
On the top: click Create cluster. 
Once the cluster is created, copy the join information
Go to the other nodes (promox computers in the web browser) (eg.https://192.168.1.2:8006)
Navigate to the cluster section in the datacenter 
Click join cluster
Paste the join information and then enter the root password of the manager server. 
This will kick you out of the node and the node will be sent to the managers web page


Installing ceph

Now that all the nodes are added to the manager
Select data center, click on ceph and install ceph
 if you have chosen proxmox 8 and above and you get an error saying you do not have a license
Look and the bottom right of the prompt before installing ceph, if it says “Enterprise (recommended) change it to standard
Follow the prompts and complete the ceph installation

Creating OSDs

Select a node
Navigate to the ceph section and select it
In the drop down below ceph OSDs will be listed
Click on OSDs
Create OSDs
Select the partition that you have created eg (/dev/sda4)
Do this for each node

Creating a pool 

Go under the ceph section in the manager node
Create a pool
Call the pool whatever you want (eg. Group12Pool)
Verify that the pool is created on the other nodes as well

Adding an ISO so that you can create a VM
Download the ISO that you will be using for the VMs 
In this case I will be using linux mint 20
https://linuxmint.com/edition.php?id=281 
Go to the node and select local, select iso and upload the iso

Creating a VM in proxmox

Select create a VM at the top right of the screen
Give the VM a name
Provided the VM with an iso image and virtual hardware specifications
Do this for each node

Adding the VMs to HA

Go to the datacenter section
In the side menu select HA
Click add and select the VMs that you have created
This allows the VM to migrate to another node if the node were to shutdown



Setting up Glusterfs for scalable containerized applications storage

Credit to: https://realtechtalk.com/GlusterFS_HowTo_Tutorial_For_Distributed_Storage_in_Docker_Kubernetes_LXC_KVM_Proxmox-2410-articles 

On all nodes 
Sudo apt install glusterfs-server
Systemctl start glusterd
Systemctl enable glusterd
On the manager
Gluster peer probe *ip of node 2*
Gluster peer probe *ip of node 3*
On node 2
Gluster peer probe *ip of node 1*
On node 3
Gluster peer probe *ip of node 1*

Create Gluster Volume

Do this on every node
mkdir -p /group12/group12Volume/brick0

On the manager
Gluster volume create group12Volume replica 3 *ip of node 1*:/group12/group12Volume/brick0 *ip of node 2*:/group12/group12Volume/brick0 *ip of node 3*:/group12/group12Volume/brick0
Gluster volume start group12Volume
If you get an error when creating the volume 
Gluster volume create group12Volume replica 3 *ip of node 1*:/group12/group12Volume/brick0 *ip of node 2*:/group12/group12Volume/brick0 *ip of node 3*:/group12/group12Volume/brick0 force
Then mount the volume into a directory
Mkdir /gluster
Mount -t glusterfs *ManagerNodeIP*:group12Volume /gluster/
Cd /gluster/
Mkdir g12GlusterTest
Mount it in /etc/fstab in order to make the mounting permanent
Nano /etc/fstab
localhost:/group12Volume /gluster glusterfs defaults,_netdev 0 0
DO A mount -a TO MAKE SURE YOU ARE NOT GONNA BREAK ANYTHING DURING A REBOOT


Setting up Docker for containerized applications


Installing docker

On each node
Sudo apt install docker-compose

Setting up docker swarm

On the VM that you want to be the manager
Docker swarm init --advisertise-addr IP address of your VM 
You will then be given a join command 
Eg docker swarm join --token  SWMTKN-1-4frt4od8te0oszxbl7gs27xyhb1q1erf308torchlf50smv3hm-avt1crisvgtwb8lssqt0rxlx1  192.168.1.249:2377 
Credit to: https://realtechtalk.com/Docker_Tutorial_HowTo_Install_Docker_Use_and_Create_Docker_Container_Images_Clustering_Swarm_Mode_Monitoring_Service_Hosting_Provider-2400-articles 

Setting up Wordpress using docker

Create a directory example in your home directory “mkdir wordpress”
Cd wordpress
Create a docker-compose.yaml file
“Touch docker-compose.yaml”
Nano docker-compose.yaml
Enter in the following script
Version: “3.5”

services:
  db:
    image: mysql:5.7
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: insecurerootpassword
      MYSQL_DATABASE: rttwp
      MYSQL_USER: rttwpuser
      MYSQL_PASSWORD: insecurerttpassword 
    
  wordpress:
    depends_on:
      - db
    image: wordpress:latest
    volumes:
      - wordpress_data:/var/www/html
    ports:
      - "7001:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: rttwpuser
      WORDPRESS_DB_PASSWORD: insecurerttpassword
      WORDPRESS_DB_NAME: rttwp
volumes:
  db_data: {}
  wordpress_data: {}

Credit to: https://realtechtalk.com/Docker_Tutorial_HowTo_Install_Docker_Use_and_Create_Docker_Container_Images_Clustering_Swarm_Mode_Monitoring_Service_Hosting_Provider-2400-articles for the script 
Change the db passwords and users to suit your needs
Inside the wordpress directory, enter the following command 
Sudo docker-compose up -d
To verify go to the node ip eg (192.168.1.1:7001) we specified the port number in the script

Scale wordpress to the other nodes in the swarm

Docker stack deploy --compose-file docker-compose.yaml wordpstack
Docker service scale wordpstack_wordpress=3
To update the stack 
Docker service update wordpstack_wordpress
To REMOVE stack
Docker stack rm wordpstack
Reference: https://github.com/sskender/wordpress-docker 


Wordpress should now be at scaled

Congrats









https://github.com/sameersbn/docker-owncloud/blob/master/kubernetes/owncloud.yaml 
https://doc.owncloud.com/server/next/admin_manual/installation/docker/
https://my.simplercloud.com/index.php?/knowledgebase/article/154/how-to-configure-owncloud-to-use-the-data-disk-for-storage-/#:~:text=By%20default%2C%20OwnCloud%20will%20store,how%20you%20can%20do%20it. 
https://www.google.com/search?q=owncloud+kubernetes&rlz=1C1CHBF_enCA986CA986&oq=owncloud+kube&gs_lcrp=EgZjaHJvbWUqBwgAEAAYgAQyBwgAEAAYgAQyBggBEEUYOTIICAIQABgWGB4yCAgDEAAYFhgeMggIBBAAGBYYHjINCAUQABiGAxiABBiKBTIGCAYQRRg9MgYIBxBFGDzSAQg0NzU2ajBqN6gCALACAA&sourceid=chrome&ie=UTF-8 
