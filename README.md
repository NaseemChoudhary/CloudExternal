# KVM Setup and VM Verification — Complete Steps

OS: Ubuntu Linux Hypervisor: KVM (Type 1) VM OS: Ubuntu VM RAM: 2 GB | CPU: 2 cores | Disk: 25 GB

## 1. Install KVM

Open the Ubuntu terminal on your base machine and run each command separately.

```
egrep -c '(vmx|svm)' /proc/cpuinfo
```

A result greater than `0` indicates CPU virtualization support is detected.

```
sudo apt update
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients virt-manager ovmf -y
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt,kvm "$USER"
ls -l /dev/kvm
```

Log out and log back in for the group changes to apply.

## 2. Verify KVM Installation

```
virsh -c qemu:///system list --all
```

If the command works, KVM/libvirt is accessible.

## 3. Create a Virtual Machine

Launch Virtual Machine Manager:

```
virt-manager
```

Then follow these steps:

1. Click Create a new virtual machine.
2. Select Local install media (ISO image).
3. Browse and select the Ubuntu ISO.
4. Allocate 2048 MB RAM and 2 CPUs.
5. Create a virtual disk of 25 GB.
6. Enter the VM name, such as `Ubuntu-KVM`.
7. Click Finish to start the OS installation.

## 4. Enable Paravirtualization Settings

In Virtual Machine Manager:

1. Shut down the VM.
2. Open the VM and select Show virtual hardware details.
3. Under Overview, ensure the hypervisor is KVM.
4. Select NIC (Network Interface) and set the device model to `virtio`.
5. Under Disk, select the disk device type as Virtio if available.
6. Click Apply, then start the VM.

Verify VirtIO devices inside the guest:

```
lspci | grep -i virtio
lsmod | grep virtio
```

Result: KVM is selected as the hypervisor, and VirtIO paravirtualized devices are configured where supported.

## 5. Make Manual Hard Disk Partitions

During Ubuntu installation inside the VM:

1. Start the VM using the Ubuntu ISO.
2. Proceed until the disk setup screen.
3. Select Manual partitioning or Something else, if available.
4. Create partitions on the 25 GB virtual disk.

Example partition layout:

| Partition       | Size            | Mount point |
| --------------- | --------------- | ----------- |
| EFI (UEFI only) | 512 MB          | `/boot/efi` |
| Root            | 20 GB           | `/`         |
| Swap            | Remaining space | Swap area   |

5. Select the correct disk and confirm the partition changes.
6. Continue installing Ubuntu.

Important: This modifies the VM's virtual disk, not your host disk. Partition options may vary by Ubuntu installer and boot mode.

## 6. Verify VM Hardware and Storage

Run these commands inside the Ubuntu VM:

```
lsblk -f
df -h
lspci | grep -i virtio
lsmod | grep virtio
```

- `lsblk -f` displays partitions and filesystem types.
- `df -h` displays mounted filesystem usage.
- `lspci` checks for VirtIO devices.
- `lsmod` lists loaded VirtIO-related modules.

## 7. Check VM Status and Details

Run these commands on the base Ubuntu machine:

```
virsh -c qemu:///system list --all
```

Then check the VM details:

```
virsh -c qemu:///system dominfo Ubuntu-KVM
```

Replace `Ubuntu-KVM` with your actual VM name.

### Result

KVM was installed and verified. A virtual machine was configured, VirtIO paravirtualization settings were enabled where supported, and manual disk partitioning was performed during OS installation.

--- 

# To create a virtual machine using VirtualBox (Type 2 Hypervisor) and ping the base machine from the VM.

### Procedure

1. Open VirtualBox and click New.
2. Enter the VM name and select the Ubuntu ISO file.
3. Allocate RAM (4048 MB) and CPU cores.
4. Create a virtual hard disk of 25 GB.
5. Click Finish and install Ubuntu.
6. Open VM Settings → Network → Adapter 1 and select Bridged Adapter.
7. Start the VM and find its IP using `ifconfig`.
8. Find the host machine's IP using `ipconfig` in Windows.
9. Open the VM terminal and type `ping <host-IP>`.
10. Verify that replies are received from the base machine.

### Result

A virtual machine was configured using VirtualBox, and network connectivity with the base machine was verified using the ping command.

---

# Nextcloud Installation on Ubuntu Server

Aim: To set up a Nextcloud storage server, access it from another device over LAN, and upload files to the server.

## 1. Install Required Packages

```
sudo apt update
sudo apt upgrade -y
sudo apt install -y apache2 mariadb-server libapache2-mod-php php php-cli php-mysql php-gd php-curl php-mbstring php-intl php-gmp php-bcmath php-xml php-zip php-imagick php-apcu unzip wget bzip2
sudo systemctl enable --now apache2 mariadb
```

## 2. Create the Database

```
sudo mariadb
```

Run inside MariaDB:

```
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'ncuser'@'localhost' IDENTIFIED BY 'YourStrongPassword';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'ncuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## 3. Download and Extract Nextcloud

```
cd /tmp
wget https://download.nextcloud.com/server/releases/latest.tar.bz2
sudo tar -xjf latest.tar.bz2 -C /var/www/
sudo mkdir -p /var/www/nextcloud-data
sudo chown -R www-data:www-data /var/www/nextcloud /var/www/nextcloud-data
```

## 4. Configure Apache

Enable required modules:

```
sudo a2enmod rewrite headers env dir mime
sudo nano /etc/apache2/sites-available/nextcloud.conf
```

Paste this configuration:

```
<VirtualHost *:80>
    DocumentRoot /var/www/nextcloud

    <Directory /var/www/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
</VirtualHost>
```

Save and exit (`Ctrl+O`, Enter, `Ctrl+X`), then run:

```
sudo a2ensite nextcloud.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

## 5. Open Nextcloud in the Browser

Find the server IP:

```
hostname -I
```

Open this address in the server's browser:

```
http://SERVER-IP/
```

Replace `SERVER-IP` with the actual IP address.

## 6. Complete Nextcloud Setup

On the setup page, enter:

- Admin username: Choose a username
- Admin password: Choose a strong password
- Data folder: `/var/www/nextcloud-data`
- Database user: `ncuser`
- Database password: The password created in Step 2
- Database name: `nextcloud`
- Database host: `localhost`

Click Install.

## 7. Access Nextcloud from Another Device (LAN)

Allow HTTP traffic through the firewall if UFW is enabled:

```
sudo ufw allow 80/tcp
```

Find the server IP:

```
hostname -I
```

On another device connected to the same LAN/Wi-Fi, open:

```
http://SERVER-IP/
```

Log in with your Nextcloud account.

If Nextcloud reports an untrusted domain, add the server IP to the `trusted_domains` section in:

```
sudo nano /var/www/nextcloud/config/config.php
```

## 8. Upload Files to Nextcloud

1. Log in to Nextcloud.
2. Open Files.
3. Click + → Upload file.
4. Select a file from your device.
5. Verify that the uploaded file appears in the file list.

## 9. Verify Where Files Are Stored

List the data directory:

```
sudo ls -la /var/www/nextcloud-data/
```

View a particular user's files:

```
sudo ls -la /var/www/nextcloud-data/USERNAME/files/
```

Replace `USERNAME` with the actual Nextcloud username.

Uploaded files are stored in the user's `files` directory under the configured data folder.

## 10. Optional: Access Over WAN Using ngrok

1. Install ngrok and authenticate it using your ngrok account.
2. Start the tunnel:

```
ngrok http 80
```

3. Copy the HTTPS forwarding URL shown by ngrok.
4. Add the ngrok hostname to Nextcloud's `trusted_domains` configuration.
5. Open the HTTPS URL on another device and log in.

Use this only for a lab demonstration; configure trusted proxy settings if required.

## Result

Nextcloud was installed and configured on Ubuntu. Files were uploaded successfully, accessed from another device over LAN, and their storage location on the server filesystem was verified.

---

# Study and Implementation of IaaS Cloud Service Model Using AWS EC2

### Task 1: Launching an EC2 Instance

1. Log in to AWS Console and open EC2.
2. Click Launch Instance.
3. Select an AMI and instance type.
4. Create a key pair and configure security settings.
5. Configure storage and click Launch Instance.
6. Verify the instance status is Running.

Result: An EC2 instance is launched successfully.

### Task 2: Monitoring the EC2 Instance

1. Open the EC2 Dashboard.
2. Select the required instance.
3. Check its state and status checks.
4. Open the Monitoring tab.
5. View CPU utilization and network activity.
6. Use instance screenshots for troubleshooting.

Result: The instance status and performance are monitored.

### Task 3: Accessing the Web Server

1. Select the EC2 instance.
2. Open its security group.
3. Click Edit inbound rules.
4. Add an HTTP rule on port 80.
5. Save the rules.
6. Open the instance's public IP in a browser.

Result: The web server becomes accessible through its public IP.

---

### Task 1: Creating an S3 Bucket

1. Log in to AWS Console and open S3.
2. Click Create bucket.
3. Enter a unique bucket name.
4. Select the AWS Region.
5. Keep public access blocked.
6. Click Create bucket.

Result: An S3 bucket is created successfully.

### Task 2: Uploading Files to S3

1. Open the S3 bucket.
2. Click Upload.
3. Select Add files.
4. Choose the required files.
5. Click Upload.
6. Verify the uploaded files under Objects.

Result: Files are stored successfully in the S3 bucket.

### Task 3: Enabling Static Website Hosting

1. Open the S3 bucket.
2. Go to Properties.
3. Select Static website hosting.
4. Click Edit and enable hosting.
5. Enter `index.html` as the index document.
6. Save changes and access the website endpoint.

Result: Static website hosting is configured on AWS S3.

---


# LAB MANUAL: INTRODUCTION TO DOCKER & CONTAINERIZATION


### Step 1: Create a Working Directory

```bash
mkdir docker-lab
cd docker-lab
nano index.html
```

Add the following HTML:

```html
<!DOCTYPE html>
<html>
<head><title>Docker Lab</title></head>
<body>
<h1>Hello! Application running inside Docker!</h1>
</body>
</html>
```

Save and exit.

### Step 2: Create a Dockerfile

```bash
nano Dockerfile
```

Add:

```dockerfile
FROM nginx:alpine
WORKDIR /usr/share/nginx/html
COPY index.html .
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Step 3: Build the Docker Image

Replace `YOUR_USERNAME` with your Docker Hub username.

```bash
docker build -t YOUR_USERNAME/my-web-app:v1.0 .
docker images
```

### Step 4: Run the Container

```bash
docker run -d -p 8080:80 --name my-running-app YOUR_USERNAME/my-web-app:v1.0
docker ps
```

Open `http://localhost:8080` in your browser.

### Step 5: Push the Image to Docker Hub

```bash
docker login
docker push YOUR_USERNAME/my-web-app:v1.0
```

## 3. Pull and Run ownCloud

### Step 1: Pull the Image

```bash
docker pull owncloud/server
docker images
```

### Step 2: Run ownCloud

```bash
docker run -d -p 8080:8080 --name my-owncloud owncloud/server
```

If port 8080 is already occupied by your custom web app, stop that container first:

```bash
docker stop my-running-app
```

### Step 3: Verify the Container

```bash
docker ps
docker logs my-owncloud
```

### Step 4: Access ownCloud

Open the browser and visit:

`http://localhost:8080`

**Result:** Successfully created a custom Docker image, ran a container, pushed the image to Docker Hub, and pulled and ran the ownCloud image.
