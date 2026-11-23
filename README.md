# 3-Tier Web Application Deployment on AWS (Linux)

A hands-on 3-tier Linux infrastructure project built on AWS (`ap-south-1`), isolating the **Web**, **Application**, and **Database** tiers across separate subnets with tiered Security Group rules and Linux service configuration.

---

## Architecture Overview

- **Web Tier (`Web` — `t3.micro`):** Runs Apache HTTP Server configured as a reverse proxy to route incoming HTTP traffic to the backend application server.
- **Application Tier (`App` — `t3.micro`):** Runs a Python Flask application (`app.py`) listening on port `5000` inside a private subnet.
- **Database Tier (`Db` — `t3.micro`):** Runs MariaDB / MySQL (`threetier_db`) on port `3306` inside a private subnet with no direct internet exposure.

---

## Network & Security Design

- **Custom VPC (`Tier-vpc`):**
  - Segmented into `Web-Subnet` (public), `App-Subnet` (private), and `Db-subnet` (private).
  - Configured separate route tables (`Public-rt` and `Private-rt`), an Internet Gateway (`Tier-IGW`) for public ingress/egress, and a NAT Gateway (`Tier-NAT`) so private instances can install packages and updates securely.
- **Least-Privilege Security Group Chaining:**
  - **Web Security Group:** Allows HTTP (`80`) from `0.0.0.0/0` and SSH (`22`) from admin IP only.
  - **App Security Group:** Allows Custom TCP (`5000`) and SSH (`22`) exclusively from the Web Security Group (`sg-047b510ab86787c0a`).
  - **DB Security Group:** Allows MySQL/Aurora (`3306`) from the App Security Group (`sg-02f5125eb5c4d8c8c`) and SSH (`22`) from the Web Security Group (`sg-047b510ab86787c0a`).

---

## Linux Administration & Verification Tasks

1. **SSH & Bastion Access:** Connected to private `App` and `Db` instances via the public `Web` instance using key-based SSH authentication.
2. **Service & Process Management:** Installed, configured, and verified Apache (`httpd`), Python/Flask, and MariaDB services across all three Linux instances.
3. **End-to-End Connectivity Testing:**
   - Verified local Flask API responses (`curl http://localhost:5000/data`).
   - Verified reverse proxy routing from the Web tier to the App tier (`curl http://<APP-PRIVATE-IP>:5000/data`).
   - Verified database read/write integration between Flask and MariaDB (`threetier_db`).

---

## Project Screenshots

- **Infrastructure & Networking:** EC2 instances (`3/3 checks passed`), `Tier-vpc`, subnets, route tables, IGW, and NAT Gateway.
- **Security Rules:** Tier-to-tier Security Group inbound configurations.
- **CLI Verification:** Git Bash terminal sessions showing SSH access, service checks, logs, and `curl` validation.


## Step-by-Step Linux Commands (By Tier)

### 1. SSH & Bastion Key Transfer (Local Git Bash)
```bash
# Set strict permissions on the private key locally
chmod 400 "<YOUR.pem>"

# Copy the key to the public Web instance (bastion) to reach private App/DB instances
scp -i "<YOUR.pem>" "<YOUR.pem>" ec2-user@<WEB-IP>:~/<YOUR.pem>

# Connect to the public Web tier instance
ssh -i "<YOUR.pem>" ec2-user@<WEB-IP>

# From the Web tier, lock down the copied key and SSH into private tiers
chmod 400 ~/<YOUR.pem>
ssh -i ~/<YOUR.pem> ec2-user@<APP-PRIVATE-IP>
ssh -i ~/<YOUR.pem> ec2-user@<DB-PRIVATE-IP>


# Install and start Apache HTTP Server
sudo yum update -y
sudo yum install -y httpd
sudo systemctl enable --now httpd
sudo systemctl status httpd

# Configure reverse proxy to forward HTTP traffic to App tier (port 5000)
sudo vi /etc/httpd/conf.d/reverse-proxy.conf
sudo apachectl configtest
sudo systemctl restart httpd


# Install Python dependencies and start Flask in the background
pip3 install flask pymysql
nohup python3 app.py > flask.log 2>&1 &

# Verify process, listening port, and logs
ps aux | grep python3
ss -tulpn | grep 5000
cat flask.log

# Test local backend response
curl -i http://localhost:5000/data


# Install, enable, and secure MariaDB
sudo yum install -y mariadb105-server
sudo systemctl enable --now mariadb
sudo mysql_secure_installation

# Create database/user and test remote DB login from App tier
mysql -u root -p
mysql -h <DB-PRIVATE-IP> -u <DB_USER> -p threetier_db



