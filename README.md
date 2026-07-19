Markdown
# DevOps Tooling Website Solution (LAMP Stack with Remote Database & NFS Storage)

## Project Overview
This project implements a robust, stateless web solution for a DevOps team. By separating the application tier (Apache Web Servers) from the data storage layer (Shared NFS Server) and database layer (Remote MySQL), we ensure data integrity, high availability, and the ability to scale web servers horizontally with ease.

---

## Architecture Diagram & Infrastructure Breakdown
1. **NFS Server**: RHEL 8 EC2 Instance providing shared network file storage for application code and web server logs.
2. **Database Server**: RHEL 8 EC2 Instance running MySQL DBMS.
3. **Web Servers**: 3 stateless RHEL 8 EC2 Instances running Apache (HTTPD) and PHP 7.4, pulling shared files simultaneously from the NFS server.

---

## Step-by-Step Implementation Guide

### Step 1: Prepare the NFS Server

1. Spin up a new EC2 instance with RHEL Linux 8 Operating System.
2. Configure LVM on the Server with 3 Logical Volumes: `lv-opt`, `lv-apps`, and `lv-logs`.
3. Format the disks using the **XFS** file system.
4. Create mount points on the `/mnt` directory for the logical volumes as follows:
   * Mount `lv-apps` on `/mnt/apps` – To be used by webservers
   * Mount `lv-logs` on `/mnt/logs` – To be used by webserver logs
   * Mount `lv-opt` on `/mnt/opt` – To be used by Jenkins server

5. Install NFS server, configure it to start on reboot, and ensure it is up and running:
   ```bash
   sudo yum -y update
   sudo yum install nfs-utils -y
   sudo systemctl start nfs-server.service
   sudo systemctl enable nfs-server.service
   sudo systemctl status nfs-server.service
Network & Permissions Setup:
Export the mounts for your Web Servers’ subnet CIDR. To find this, check your EC2 details in the AWS web console, locate the 'Networking' tab, and open the Subnet link.

Set up permissions that allow your Web servers to read, write, and execute files on NFS:

Bash
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt
sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt
sudo systemctl restart nfs-server.service
Configure access to NFS for clients within the same subnet:

Bash
sudo vi /etc/exports
Add the following entries (replace <Subnet-CIDR> with your actual subnet CIDR, e.g., 172.31.32.0/20):

Plaintext
/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
Apply the changes:

Bash
sudo exportfs -arv
Firewall & Security Groups Entry:
Check which port is used by NFS and open it using Security Groups by adding a new Inbound Rule:

Bash
rpcinfo -p | grep nfs
⚠️ Important Note: In order for the NFS server to be accessible from your client, you must open the following inbound ports: TCP 2049, UDP 2049, TCP 111, and UDP 111.

Step 2: Configure the Database Server
Connect to your Database Server and install MySQL:

Bash
sudo yum update -y
sudo yum install mysql-server -y
sudo systemctl start mysqld
sudo systemctl enable mysqld
Log into MySQL as root and run the following queries to configure the database and user permissions:

Bash
sudo mysql
SQL
-- Create the database
CREATE DATABASE tooling;

-- Create the user with access restricted to your Web Server subnet
CREATE USER 'webaccess'@'172.31.16.0/255.255.240.0' IDENTIFIED BY 'NewSecurePassword123!';

-- Grant all privileges on the tooling database to this user
GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'172.31.16.0/255.255.240.0';

-- Flush privileges to apply changes and exit
FLUSH PRIVILEGES;
EXIT;
Configure MySQL to listen to remote connections by opening the configuration file:

Bash
sudo vi /etc/my.cnf
Find the bind-address line and change it to 0.0.0.0 or comment it out:

Plaintext
bind-address = 0.0.0.0
Restart the service to apply changes:

Bash
sudo systemctl restart mysqld
Note: Ensure the DB Server's AWS Security Group has an inbound rule allowing TCP Port 3306 from your Web Server subnet.

Step 3: Prepare and Configure the Web Servers
Note: These steps must be performed on all three identical Web Servers to maintain a stateless architecture.

Install the NFS client utilities:

Bash
sudo yum update -y
sudo yum install nfs-utils nfs4-acl-tools -y
Create the target directory and mount the NFS server's shared application export to /var/www:

Bash
sudo mkdir -p /var/www
sudo mount -t nfs -o rw,nosuid 172.31.17.253:/mnt/apps /var/www
Create the Apache log directory and mount it to the NFS server's log export:

Bash
sudo mkdir -p /var/log/httpd
sudo mount -t nfs -o rw,nosuid 172.31.17.253:/mnt/logs /var/log/httpd
Make the mounts permanent so they persist after a reboot:

Bash
sudo vi /etc/fstab
Add these two lines at the very bottom of the file:

Plaintext
172.31.17.253:/mnt/apps /var/www nfs defaults 0 0
172.31.17.253:/mnt/logs /var/log/httpd nfs defaults 0 0
Verify they mount correctly:

Bash
sudo mount -a
df -h
Install Apache, enable required repositories, and install PHP 7.4 along with its extensions:

Bash
sudo yum install httpd -y
sudo dnf install [https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm](https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm) -y
sudo dnf install dnf-utils [http://rpms.remirepo.net/enterprise/remi-release-8.rpm](http://rpms.remirepo.net/enterprise/remi-release-8.rpm) -y
sudo dnf module reset php -y
sudo dnf module enable php:remi-7.4 -y
sudo dnf install php php-opcache php-gd php-curl php-mysqlnd -y
sudo systemctl start httpd && sudo systemctl enable httpd
sudo systemctl start php-fpm && sudo systemctl enable php-fpm
🛠️ Hard-Fought Fixes & Troubleshooting Highlights
1. Fixing Apache Permission Controls (HTTP 403 Forbidden Errors)
SELinux blocks Apache from reading NFS mounts by default. To open up access and make it permanent:

Bash
# Allow Apache to execute memory processes (required for PHP-FPM to work)
sudo setsebool -P httpd_execmem 1

# Disable SELinux temporarily
sudo setenforce 0

# Disable permanently on reboot
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
2. Code Ownership Adjustments
Since /var/www is shared via NFS, ensure correct permissions are mapped across your infrastructure so Apache can safely parse the repository's html files:

Bash
sudo chown -R apache:apache /var/www/html
sudo chmod -R 755 /var/www/html
3. Verification of System Services and Shares
To ensure that client mounts are exported correctly and web services are active:

4. Database Seeding & Admin Configuration
Update your database credentials in /var/www/html/functions.php. Apply the tooling-db.sql migration script:

Bash
mysql -h <database-private-ip> -u webaccess -p tooling < tooling-db.sql
Execute this insert query inside MySQL to provision your administrative user credentials:

SQL
INSERT INTO `users` (`id`, `username`, `password`, `email`, `user_type`, `status`) 
VALUES (1, 'myuser', '5f4dcc3b5aa765d61d8327deb882cf99', 'user@mail.com', 'admin', '1');
Verification
Open your browser and navigate to http://<Web-Server-Public-IP-Address-or-Public-DNS-Name>/index.php. You will be greeted by the tooling system login panel where you can successfully authenticate using myuser!
