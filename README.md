# Three-Tier Architecture Deployment on Azure

This project documents the implementation of a secure three-tier application architecture deployed on Microsoft Azure. The infrastructure spans two global regions to demonstrate Global VNet Peering and tiered network isolation.

## 🏗 Architecture Overview

The architecture consists of three distinct layers hosted on Linux Virtual Machines, separated by subnets and regions[cite: 3, 20]:

1.  **Web Tier (Nginx):** Public-facing server handling HTTP requests[cite: 49].
2.  **Application Tier (Apache Tomcat):** Middleware server restricted to internal access only[cite: 50].
3.  **Database Tier (MySQL):** Backend database server restricted to the Application Tier[cite: 52].

### Network Topology

* **Resource Group:** `CharanRG01` (Region: Australia East)[cite: 5, 12].
* **VNet 1:** `CharanVNET01` (Region: **Korea Central**)[cite: 6, 7].
    * **Web Subnet:** `10.12.1.0/24` (Hosting `cwebserver-vm`)[cite: 8, 40].
    * **App Subnet:** `10.12.2.0/24` (Hosting `cappserver-vm`)[cite: 16, 42].
* **VNet 2:** `CharanVNET02` (Region: **Central India**)[cite: 9, 21].
    * **DB Subnet:** `10.1.0.0/24` (Hosting `cdbserver-vm`)[cite: 24, 47].
* **Connectivity:** Global VNet Peering connects `CharanVNET01` and `CharanVNET02`[cite: 25, 83].

---

## 🚀 Deployment Steps

### 1. Web Tier Configuration
**VM Name:** `cwebserver-vm`  
**Role:** Reverse Proxy / Web Server

1.  **Install Nginx:**
    ```bash
    sudo su
    apt update
    apt install nginx
    ```

2.  **Start Service:**
    ```bash
    systemctl start nginx
    systemctl status nginx
    ```

3.  **Verification:** Access the VM's Public IP in a browser. [cite_start]You should see the "Welcome to nginx!" page[cite: 121].

### 2. Application Tier Configuration
**VM Name:** `cappserver-vm`  
**Role:** Tomcat Application Server  
**Access:** Accessible only via SSH from `cwebserver-vm`.

1.  **Install Java (JDK):**
    ```bash
    sudo su -
    apt update
    apt install default-jdk -y
    ```

2.  **Install Tomcat (v10.1.50):**
    ```bash
    # Create user
    sudo useradd -m -U -d /opt/tomcat -s /bin/false tomcat

    # Download and extract
    cd /tmp
    wget [https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.50/bin/apache-tomcat-10.1.50.tar.gz](https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.50/bin/apache-tomcat-10.1.50.tar.gz)
    sudo tar xzvf apache-tomcat-10.1.50.tar.gz -C /opt/tomcat --strip-components=1

    # Set permissions
    sudo chown -R tomcat:tomcat /opt/tomcat
    sudo sh -c 'chmod +x /opt/tomcat/bin/*.sh'
    ```

3.  **Configure Systemd Service:**
    Create `/etc/systemd/system/tomcat.service` with the necessary environment variables (`JAVA_HOME`, `CATALINA_HOME`).

4.  **Start Tomcat:**
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl start tomcat
    sudo systemctl enable tomcat
    ```
    

### 3. Database Tier Configuration
**VM Name:** `cdbserver-vm`  
**Role:** MySQL Database  
**Access:** Accessible only via SSH from `cappserver-vm`.

1.  **Install MySQL:**
    ```bash
    sudo su
    apt update
    apt install mysql-server -y
    ```

2.  **Verify Installation:**
    ```bash
    systemctl status mysql
    mysql -e "Show VARIABLES LIKE 'port';"
    ```

---

## 🛡️ Network Security Rules (NSG)

The following Network Security Group rules enforce the architecture's isolation policies.

| Tier | Traffic Type | Port | Protocol | Source | Destination | Action |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Web** | Inbound | 80 (HTTP) | TCP | Any | Any | Allow |
| **Web** | Inbound | 22 (SSH) | TCP | Any | Any | Allow |
| **App** | Inbound | 8080 (Tomcat)| TCP | 10.12.1.4 (Web) | 10.12.2.4 (App) | Allow |
| **App** | Inbound | 22 (SSH) | TCP | 10.12.1.4 (Web) | 10.12.2.4 (App) | Allow |
| **DB** | Inbound | 3306 (SQL) | TCP | 10.12.2.4 (App) | 10.1.0.4 (DB) | Allow |
| **DB** | Inbound | 22 (SSH) | TCP | 10.12.2.4 (App) | 10.1.0.4 (DB) | Allow |

### 🔒 Security Validation
* **Public Access to App Server:** Attempting to access the App Server public IP on port 8080 results in a **Connection Timed Out** error, confirming that the NSG is correctly blocking public traffic.
* **Public Access to DB Server:** Direct public access is strictly denied; management is only possible via the App Server jump.
