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

sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt
sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt
sudo systemctl restart nfs-server.service
