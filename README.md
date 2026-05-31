## 📐 Architectural Infrastructure & Traffic Flow

The following diagram represents how an external user request securely interacts with the AWS EC2 environment and the internal Nginx server loops:

```text
       [ Public Internet ]
                │
                │ (User requests: http://13.52.182.176)
                ▼
   ┌────────────────────────┐
   │   AWS Security Group   │ ◄─── Filters inbound rules globally
   │  (Ports: 80 & 443 Open)│      (Drops unspecified packets)
   └────────────┬───────────┘
                │
                │ (Traffic passed through)
                ▼
   ┌────────────────────────┐
   │    Ubuntu OS Kernel    │ ◄─── Evaluates Local Firewall (UFW)
   └────────────┬───────────┘
                │
                ▼
   ┌────────────────────────┐
   │  Nginx Server Engine   │
   │                        │
   │  ┌──────────────────┐  │
   │  │  Port 80 Block   │  │ ◄─── Catches unsecured HTTP requests
   │  │ (Http Redirect)  │  │      Sends a 301 Permanent Redirect
   │  └─────────┬────────┘  │
   │            │           │
   │            ▼           │
   │  ┌──────────────────┐  │
   │  │  Port 443 Block  │  │ ◄─── Processes secure HTTPS requests
   │  │  (SSL Handshake) │  │      Uses: yourdomain.crt & yourdomain.key
   │  └─────────┬────────┘  │
   └────────────┼───────────┘
                │
                │ (Decrypted internal routing)
                ▼
   ┌────────────────────────┐
   │ Static Web File Root   │
   │   (/var/www/html/)     │ ◄─── Serves requested asset: index.html
   └────────────────────────┘
```

### 🔄 Request Lifecycle Walkthrough

1. **Edge Perimeter Validation**: The user inputs the URL into their browser. The packet arrives at the AWS network boundary. The **AWS Security Group** evaluates the packet; because rules are defined for `0.0.0.0/0` on ports 80 and 443, the layer allows transit.
2. **OS Kernel Ingress**: The traffic hits the Ubuntu operating system network stack. **UFW** checks its rule list, confirms no drop flags are active, and hands the process to the socket layers.
3. **Protocol Enforcement (Nginx Routing)**:
   * **If HTTP (Port 80)**: Nginx intercepts the packet, immediately breaks processing, and throws a header response back to the client browser containing a `301 Moved Permanently` payload targeting the identical string over `https://`.
   * **If HTTPS (Port 443)**: Nginx initiates an **SSL/TLS handshake** utilizing the local keypair (`yourdomain.crt` / `yourdomain.key`). The asymmetric keys negotiate a symmetric session state with the browser client.
4. **Content Serialization**: Once the secure encrypted tunnel is established, Nginx scans the directory index path designated by the internal site configuration parameters. It verifies standard filesystem read constraints, locates the valid target file (`index.html`), and feeds the static structural markup back through the secure pipe to render on the client screen.

# Secure Nginx Configuration with HTTPS on AWS EC2

This repository documents the step-by-step process of configuring and enabling HTTPS traffic on an Nginx web server hosted on an Ubuntu AWS EC2 instance using a self-signed SSL certificate.

---

## 🛠️ Prerequisites & Architecture
* **Cloud Provider**: AWS EC2 (Ubuntu Linux)
* **Web Server**: Nginx
* **Encryption**: Self-Signed SSL/TLS Certificate (for IP-based secure routing)
* **Active Ports**: 80 (HTTP) & 443 (HTTPS)

---

## 🚀 Step-by-Step Implementation Guide

### Step 1: Generate the SSL/TLS Certificates
Because a raw Public IP address cannot use automated third-party certificate authorities (like Let's Encrypt), a self-signed certificate was generated manually using OpenSSL.

1. Create a dedicated directory to store the secure encryption keys:
   ```bash
   sudo mkdir -p /etc/nginx/ssl
   ```
2. Generate a 2048-bit RSA private key and a self-signed certificate valid for 365 days:
   ```bash
   sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout /etc/nginx/ssl/yourdomain.key \
     -out /etc/nginx/ssl/yourdomain.crt
   ```

### Step 2: Configure the Nginx Virtual Host
The default configuration file was modified to handle secure routing on port 443 and automatic port 80 traffic redirection.

1. Open the active Nginx configuration block:
   ```bash
   sudo nano /etc/nginx/sites-available/default
   ```
2. Apply the dual-server block schema:
   ```nginx
   # 1. HTTPS Secure Server Block
   server {
       listen 443 ssl default_server;
       listen [::]:443 ssl default_server;

       ssl_certificate /etc/nginx/ssl/yourdomain.crt;
       ssl_certificate_key /etc/nginx/ssl/yourdomain.key;

       ssl_protocols TLSv1.2 TLSv1.3;
       ssl_prefer_server_ciphers on;

       root /var/www/html;
       index index.html index.htm;
       server_name _;

       location / {
           try_files \(uri\)uri/ =404;
       }
   }

   # 2. HTTP to HTTPS Automatic Redirect
   server {
       listen 80 default_server;
       listen [::]:80 default_server;
       server_name EC2_PublicIP;
       return 301 https://\(host\)request_uri;
   }
   ```

### Step 3: Serve Web Assets & Fix Directory Indexing
Nginx throws a `403 Forbidden` error if the folder is missing a default matching index index file. 
1. Create or rename the web root asset file to match the configuration rules:
   ```bash
   sudo mv /var/www/html/index.nginx-debian.html /var/www/html/index.html
   ```
2. Assign strict operational permissions to the Nginx user runtime:
   ```bash
   sudo chown -R www-data:www-data /var/www/html
   sudo chmod 644 /var/www/html/index.html
   ```

### Step 4: Validate and Apply
1. Test Nginx code compilation syntax for typos or broken symlinks:
   ```bash
   sudo nginx -t
   ```
2. Safely cycle the runtime process daemon:
   ```bash
   sudo systemctl restart nginx
   ```

### Step 5: Unblock Networking (Firewalls)
Traffic must be cleared through both the machine-level kernel firewall and the external AWS networking wrapper.
1. **Ubuntu Local Firewall (UFW)**: Set to inactive or allowed:
   ```bash
   sudo ufw allow 'Nginx Full'
   ```
2. **AWS Security Group Inbound Rules**:
   * **Rule 1**: Type `HTTP` | Port `80` | Source `Anywhere-IPv4 (0.0.0.0/0)`
   * **Rule 2**: Type `HTTPS` | Port `443` | Source `Anywhere-IPv4 (0.0.0.0/0)`

---

## 🔍 Troubleshooting Commands Used
* Check active listening ports locally:  
  `sudo ss -tulpn | grep nginx`
* Check the structural file layout of active components:  
  `ls -la /etc/nginx/sites-available/`
* Inspect runtime error logs:  
  `sudo journalctl -xeu nginx.service`

---

## 💡 Key Takeaways
1. **IP vs Domain SSL Warnings**: Browsers display a "Not Secure" alert when connecting via a raw IP address using self-signed certificates. This is normal behavior because identity cannot be verified by an external trusted root authority, though the protocol **fully encrypts** the payload transit layer.
2. **Symlink Structuring**: Avoid literal configuration placeholders inside system commands (`/etc/nginx/sites-enabled/`) to prevent runtime daemon service startup initialization crashes.

