# Cloud-Web-Security-Lab-with-Waf-and-Mde
Install a Web Application Firewall (WAF), simulate web attacks, and observe how foreign attacks are detected and blocked.


# Network-Focused Lab: WAF Deployment & Attack Simulation – 30th May 2025

This lab focused on deploying a Web Application Firewall (WAF) using SafeLine, simulating common attacks (e.g., SQLi, XSS), observing foreign attack traffic, and analyzing behavior via Microsoft Defender for Endpoint (MDE). The lab was performed using a single Ubuntu VM hosted on Microsoft Azure.

> **Note:** Due to Azure VM constraints, DVWA and the WAF were deployed on the same VM. In production environments, these services should be separated to ensure WAF integrity.

---

## Table of Contents



---

## Tools Used

| Tool / Service               | Purpose                                                      |
|-----------------------------|--------------------------------------------------------------|
| **Microsoft Azure**         | Hosting Ubuntu VM                                            |
| **Ubuntu Server 24.04 LTS** | Operating system for hosting DVWA and SafeLine              |
| **Microsoft Defender for Endpoint (MDE)** | Endpoint detection and monitoring                    |
| **xRDP & Xfce4**            | Enable GUI access to Ubuntu VM                              |
| **Apache2**                 | Web server used for DVWA                                     |
| **MariaDB**                 | Database backend for DVWA                                    |
| **PHP**                     | DVWA web application dependency                             |
| **DVWA (Damn Vulnerable Web App)** | Target for security testing                        |
| **Docker**                  | Container platform used to deploy SafeLine WAF              |
| **SafeLine WAF**            | Open-source Web Application Firewall                        |
| **OpenSSL**                 | Used to generate self-signed SSL certificates                |

---

## Phase 1 – VM Creation & MDE Onboarding

1. **Create Ubuntu VM** in Azure:
   - Image: Ubuntu Server 24.04 LTS
   - Name: `rich-ubuntu`
   - Inbound Port: SSH (22)

2. **Onboard VM to Microsoft Defender for Endpoint**:

```bash
# Step 1 – Download the onboarding script
curl -o DefenderOnboarding.zip https://sacyberrange00.blob.core.windows.net/mde-agents/Linux-Server-GatewayWindowsDefenderATPOnboardingPackage.zip

# Step 2 – Unzip and navigate
sudo apt install unzip
unzip DefenderOnboarding.zip
cd Linux*

# Step 3 – Run onboarding script
sudo python3 MicrosoftDefenderATPOnboardingLinuxServer.py

# Step 4 – Verify MDE agent health
mdatp health


# If mdatp is not found:

# Install manually
sudo apt install mdatp
# OR:
curl -o mdatp.deb https://aka.ms/linux-mdatp-download
sudo dpkg -i mdatp.deb
sudo apt --fix-broken install

# Verify installation
mdatp --version
mdatp health --field org_id
```

## Phase 2 – Enabling GUI + RDP

Enable remote GUI access with RDP and XFCE4:

```bash
# Update and install desktop environment
sudo apt update
sudo apt install xrdp xfce4 xfce4-goodies -y

# Set default session
echo xfce4-session > ~/.xsession

# Enable and start xrdp
sudo systemctl enable xrdp
sudo systemctl start xrdp

# Allow RDP in firewall
sudo ufw allow 3389/tcp
```
You can now log in to your Ubuntu VM using Microsoft Remote Desktop (Windows/macOS/Linux) with your Linux credentials.


## Phase 3 - DVWA Setup & Configuration

Step 1: Install DVWA Dependencies
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install apache2 mariadb-server php php-mysqli php-gd libapache2-mod-php git unzip -y
```

Step 2: Install DVWA
```bash
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git
sudo chown -R www-data:www-data DVWA/
sudo chmod -R 755 DVWA/
```

Step 3: Configure DVWA
```bash
cd /var/www/html/DVWA/config
sudo cp config.inc.php.dist config.inc.php
sudo nano config.inc.php
```

Update the following lines
```php
$_DVWA[ 'db_server' ] = '127.0.0.1';
$_DVWA[ 'db_database' ] = 'dvwa';
$_DVWA[ 'db_user' ] = 'dvwa';
$_DVWA[ 'db_password' ] = 'P@ssw0rd!';
$_DVWA[ 'db_port' ] = '3306';
```

Step 4: Configure the Database 
```bash
sudo mysql -u root
```

Inside MySQL:
```sql
CREATE DATABASE dvwa;
CREATE USER 'dvwa'@'localhost' IDENTIFIED BY 'P@ssw0rd!';
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Step 5: Restart Services
```bash
sudo systemctl restart apache2
sudo systemctl restart mysql
```

Step 6: Access DVWA
```bash
http://<your-public-ip>/DVWA
```

## Phase 3 Part 2 – Insert Users via SQL
```bash
INSERT INTO users (user_id, first_name, last_name, user, password, avatar, last_login, failed_login) VALUES
(6, 'Alice', 'Smith', 'alice', '5f4dcc3b5aa765d61d8327deb882cf99', 'avatars/alice.jpg', '2025-05-20 12:45:00', 0),
(7, 'Bob', 'Johnson', 'bobby', '7c6a180b36896a0a8c02787eeafb0e4c', 'avatars/bob.png', '2025-05-19 08:20:00', 1),
(8, 'Charlie', 'Adams', 'charlie', 'e99a18c428cb38d5f260853678922e03', 'avatars/charlie.jpg', '2025-05-18 22:10:00', 2),
(9, 'Eve', 'Hacker', 'eve', '21232f297a57a5a743894a0e4a801fc3', 'avatars/eve.png', '2025-05-20 01:30:00', 5),
(10, 'Mallory', 'Danger', 'mallory', '81dc9bdb52d04dc20036dbd8313ed055', 'avatars/mallory.jpg', '2025-05-21 10:00:00', 0);
```

## Phase 3 Part 3 – Change Apache Port to 8080
```bash
# Edit Apache port config
sudo nano /etc/apache2/ports.conf
# Change: Listen 80 → Listen 8080

# Edit virtual host
sudo nano /etc/apache2/sites-available/000-default.conf
# Change: <VirtualHost *:80> → <VirtualHost *:8080>

# Restart Apache
sudo systemctl restart apache2

# Optional: Allow port 8080 through firewall
sudo ufw allow 8080/tcp
```

Now DVWA can be accessed via:
```bash 
http://<your-public-ip>:8080/DVWA
```

## Phase 4 – Installing Docker & SafeLine WAF

Step 1: Install Docker
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker's GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker repo
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Test
sudo docker run hello-world
```

Step 2: Install SafeLine WAF
```bash
bash -c "$(curl -fsSLk https://waf.chaitin.com/release/latest/manager.sh)" -- --en
```

Example output:
```pgsql
[INFO] Initial username：admin
[INFO] Initial password：f5UQQxoN
[INFO] SafeLine WAF panel: https://<vm-ip>:9443/
```

## Phase 5 – Configuring SSL for DVWA


SSL creation process: 

Step 1 - Create a directory to store SSL files (this helps to keep things organized)
```bash
sudo mkdir -p /etc/ssl/dvwa
cd /etc/ssl/dvwa
```

Step 2 - generate the private key

```bash
# This creates a 4096-bit RSA private key (dvwa.key).
sudo openssl genrsa -out dvwa.key 4096
```



Step 3 - Generate a CSR using the private key 

```bash
sudo openssl req -new -key dvwa.key -out dvwa.csr
```

```pgsql
Prompted to enter values :
Country Name (2 letter code) [AU]: US
State or Province Name (full name) [Some-State]: California
Locality Name (eg, city) []: Los Angeles
Organization Name (eg, company) []: DVWA Lab
Organizational Unit Name (eg, section) []: Web Security
Common Name (e.g. server FQDN or YOUR name) []: 135.237.184.217
Email Address []: admin@yourlab.local

# Important: Use your IP address (135.237.184.217) as the Common Name (CN), unless you own a domain.
```

Step 4 : Verify the CSR 
```bash
sudo openssl req -text -noout -verify -in dvwa.csr
# This will show the subject (CN, email, etc.) and public key info.
```


Step 5 : This creates dvwa.crt - self signed certificate for 365 days 
```bash
sudo openssl x509 -req -days 365 -in dvwa.csr -signkey dvwa.key -out dvwa.crt
```

```pgsql
# Final Files in /etc/ssl/dvwa/:

| File       | Description                                |
| ---------- | ------------------------------------------ |
| `dvwa.key` | Private key                                |
| `dvwa.csr` | Certificate Signing Request                |
| `dvwa.crt` | Self-signed certificate                    |
```


 





