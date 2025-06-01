# Cloud-Web-Security-Lab-with-Waf-and-Mde

How Safeline WAF works:

![how-safeline-waf-works](https://github.com/user-attachments/assets/db25403e-711f-4880-b093-774d1e5c6c9f)



# WAF (SafeLine) Deployment & Attack Simulation 

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

<img width="1427" alt="vm-creation-richubuntu" src="https://github.com/user-attachments/assets/e216ac20-f42b-447e-a5db-3c800e78f662" />


2. **Onboard VM to Microsoft Defender for Endpoint**:

```bash
# Step 1 – Download your specific onboarding script to the vm via CLI
curl -o DefenderOnboarding.zip https://''''''''''''''''''/Linux-Server-GatewayWindowsDefenderATPOnboardingPackage.zip

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

## Phase 2 – Enabling GUI + RDP for Ubuntu 24.04 image 

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
# Log in to Ubuntu VM using Microsoft Windows App with Linux credentials.
```
<img width="1431" alt="Setting up GUI from the commandline" src="https://github.com/user-attachments/assets/8afbe365-49a9-4db9-8155-383ea437ee5b" />
Running Script to Enable Remote GUI to better visualize and ease of movement, etc throughout the lab.


<img width="1435" alt="Ubuntuimage-GUI-rich-ubuntu" src="https://github.com/user-attachments/assets/4653a820-5436-47c7-bda1-5477f4f7106b" />
Successful GUI access through Windows App post script 



## Phase 3 - DVWA Setup & Configuration

Step 1: Install DVWA Dependencies
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install apache2 mariadb-server php php-mysqli php-gd libapache2-mod-php git unzip -y
```

<img width="1223" alt="Install-DVWAdependencies" src="https://github.com/user-attachments/assets/a45b23c9-e354-4bcf-9af8-c7232d8b5071" />


Step 2: Install DVWA
```bash
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git
sudo chown -R www-data:www-data DVWA/
sudo chmod -R 755 DVWA/
```

<img width="1143" alt="Download and install DVWA" src="https://github.com/user-attachments/assets/93bbbb0e-eeef-44ad-a111-be4368e22084" />


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

<img width="1425" alt="Edit Lines in MariaDB" src="https://github.com/user-attachments/assets/7652f14b-58df-4021-a2d7-7dd7b569d993" />



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
# http://<your-public-ip>/DVWA
http://135.237.184.217/DVWA
```

<img width="1440" alt="Successfully accessed DVWA via browser " src="https://github.com/user-attachments/assets/d4881f68-8c4c-4e71-81d3-25dd2c77519a" />


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
<img width="995" alt="before apache change-port80" src="https://github.com/user-attachments/assets/4b139113-65d4-4b2b-ad64-61b77a1d39c9" />

before editing Apache port config 

<img width="999" alt="changing apache to port 8080-virtualhost" src="https://github.com/user-attachments/assets/975c3b5b-f92a-4854-9d75-b0ff8ea705a1" />

After editing pache port config to 8080


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
<img width="1169" alt="installing docker" src="https://github.com/user-attachments/assets/222a7399-8e03-4190-9556-784fd396b878" />


<img width="921" alt="sudo docker hello-world" src="https://github.com/user-attachments/assets/176a7543-6dd4-4d4c-b7f2-4ffd9a335a43" />
Test - 'hello-world'

Step 2: Install SafeLine WAF
```bash
bash -c "$(curl -fsSLk https://waf.chaitin.com/release/latest/manager.sh)" -- --en
```

<img width="1434" alt="Running the script to install the safepoint firewall automatically" src="https://github.com/user-attachments/assets/16d6492c-b01d-4964-9091-9af682a0d4c3" />



Example output:
```pgsql
[INFO] Initial username：admin
[INFO] Initial password：f5UQQxoN
[INFO] SafeLine WAF panel: https://<vm-ip>:9443/
```
<img width="1433" alt="Automatic script for WAF returns username and password" src="https://github.com/user-attachments/assets/fdc5fb81-4524-4149-a7b3-548aa1c41fed" />






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

<img width="1440" alt="dvwa key for ssl certificate" src="https://github.com/user-attachments/assets/485e4a7c-9b0c-43ac-882c-a07b63f98007" />


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

<img width="1405" alt="creating SSL certificate pt 1" src="https://github.com/user-attachments/assets/8fd68475-7010-4884-b3e9-bbfc46a566f6" />
Step 1 through step 4 


Step 5 : This creates dvwa.crt - self signed certificate for 365 days 
```bash
sudo openssl x509 -req -days 365 -in dvwa.csr -signkey dvwa.key -out dvwa.crt
```


<img width="1440" alt="dvwa crt for certificate" src="https://github.com/user-attachments/assets/c75a38f6-ba02-4923-9b6a-f1c9ec6fa26e" />


```pgsql
# Final Files in /etc/ssl/dvwa/:

| File       | Description                                |
| ---------- | ------------------------------------------ |
| `dvwa.key` | Private key                                |
| `dvwa.csr` | Certificate Signing Request                |
| `dvwa.crt` | Self-signed certificate                    |
```

<img width="1125" alt="SSL certificate  crt  csr " src="https://github.com/user-attachments/assets/4b53d17e-8353-4a19-af7f-d09fa070a6f3" />
Verified Certificates that were created in dvwa file





<img width="1434" alt="Adding certificate to WAF (safeline)" src="https://github.com/user-attachments/assets/515ee165-b3e2-4b42-9d2d-c65d2e25a3ff" />
Uploading SSL certificate to SafeLine WAF

## Phase 6 - Add Application to WAF to begin monitoring and protecting


 <img width="591" alt="Add-application-safeline-WAF" src="https://github.com/user-attachments/assets/b8fa1b35-e409-4cd3-a6c9-555999902a38" />
Add Application form where you configure for your specific setup


<img width="1429" alt="Application in safeline" src="https://github.com/user-attachments/assets/a517fbff-9693-4ef8-93a5-114c8710661b" />
Application successfully onbaorded to SafeLine WAF 

## Phase 7 - set up Http flood-rate limiting rule and add 1 user to set up authentication for the page and then breach rate limit + refresh url and have to authenticate through the WAF + perform sql injection and observe block 
 - WAF demonstrations with safeline (add authentic images of what happened in the lab simulation later):


### HTTP Flood simulation


<img width="1433" alt="HTTP-flood-accesslimitingrule-success" src="https://github.com/user-attachments/assets/1d687cf1-c12c-47ac-9538-bf19bdbfe69a" />

Create and Enable rate limiting rule to protect against potential DDoS

<img width="1376" alt="HTTP flood - blocked" src="https://github.com/user-attachments/assets/9224008d-ec79-40f8-8f6d-904b459d1455" />
When rate limit rule is breached by someone they are blocked from accessing the website by the WAF 


### Authnetication config
sign in page 


<img width="1428" alt="Configure-Authentication-Newusername" src="https://github.com/user-attachments/assets/22c68594-49e6-426b-84bc-c910d3fb6c17" />
Configure new user name and enable uthentication to log into the website through the WAF

<img width="1373" alt="Authenticationconfig-signin-WAF" src="https://github.com/user-attachments/assets/26bede40-02b0-42d7-a924-af63f830abf8" />
Once Authentication is enabled you will only be allowed to enter the website by entering a valid registered username and password

<img width="1416" alt="Authentication-WAF-attempts" src="https://github.com/user-attachments/assets/a054f039-9033-4caf-af4f-6c284f1c8038" />
From the WAF you can see all Authentication attempts at accessing the page and ip address associated 

### SQLinjection 

<img width="1432" alt="SQL injection without WAF intervention" src="https://github.com/user-attachments/assets/dbd2c2f7-7d1a-4160-bed4-2432446a489d" />
I performed a SQLi with the WAF on 'Audit' mode instead of 'Defense' and then i performed a SQLijection using the input below to observe what happens when a successful SQLi is carried out without WAF intervention.

```sql
' OR 1=1 -- -
```


<img width="1374" alt="SQLinjection-blocked" src="https://github.com/user-attachments/assets/f24886aa-0444-4684-8e85-a42186849e6b" />
SQLi performed with WAF turned on



<img width="1432" alt="Logs of Attacks-SQLi" src="https://github.com/user-attachments/assets/8572a006-0624-4c6e-b24d-6a4d55f3701b" />
Logs of SQLi attacks. Take note of the ones performed when the mode was in 'Audit' were not blocked, however they are observed in the logs as an Attack (IDS).


## Phase 8 - Monitoring Logs on MDE





## Conclusion

This lab successfully demonstrated the deployment of a Web Application Firewall (SafeLine) alongside a vulnerable application (DVWA) within a secure, monitored environment on Microsoft Azure. Despite the limitation of deploying both DVWA and SafeLine WAF on a single VM—an architecture that would be segmented in a production scenario—the implementation effectively simulated real-world security use cases such as SQL injection and XSS.

Additionally, Microsoft Defender for Endpoint (MDE) was used to monitor the VM, enabling visibility into endpoint activity and threat detection. Remote GUI access was also configured via XRDP, improving usability during lab interaction.

Through this hands-on experience, core cybersecurity and DevSecOps principles were practiced:

* Setting up layered defense (WAF + MDE)
* Simulating real-world attacks (SQLi)
* Observing WAF behavior and traffic blocking
* Managing secure service configurations (SSL, ports, access)
* Documenting repeatable and auditable procedures
* This lab illustrates not only technical proficiency in infrastructure and security deployment but also an understanding of security observability and layered protection strategies.

## Final Notes

* SafeLine WAF successfully mitigated simulated attacks against DVWA.
* Logs and behavior could be further integrated into centralized SIEM solutions for extended analysis.
* All services were containerized or configured following industry best practices within resource limitations.
* MDE provided detailed threat intel, enhancing endpoint-level protection and visibility.
* GUI access was successfully implemented for convenience, highlighting versatility across CLI and GUI management interfaces.

