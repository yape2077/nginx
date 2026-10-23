# Technical Implementation Report: Nginx SSL/TLS Deployment

**Target Domain:** `example.com` (and `www.example.com`)  
**Server IP:** `192.168.1.1/24`  
**Web Services:** Nginx on Port 80 (HTTP Redirect) & Port 443 (HTTPS)  
**Target Environment:** Ubuntu Server (Server) & Windows 10/11 (Client)  
**Status:** Completed & Verified  

---

## 1. Architecture & Traffic Flow

The web service intercepts unencrypted HTTP traffic on port 80 and returns a permanent `301 Moved Permanently` redirect to port 443 (HTTPS). The TLS certificate includes modern **Subject Alternative Name (SAN)** extensions to ensure secure connections across both domain names and the raw IP.

```
[ Client Request: http://example.com ]
                  │
                  ▼
       [ Nginx Port 80 Listener ]
                  │
        (HTTP 301 Redirect)
                  │
                  ▼
[ Client Escalation: https://example.com ]
                  │
                  ▼
      [ Nginx Port 443 (TLS) ]
    ├── Certificate: example.crt (with SAN)
    └── Key:         example.key
                  │
                  ▼
[ Document Root: /var/www/example.com/index.html ]
```

---

## 2. Environment Specifications

| Parameter | Specification |
| :--- | :--- |
| **Server OS** | Ubuntu Linux (20.04 / 22.04 / 24.04 LTS) |
| **Server IP** | `192.168.1.1/24` |
| **Client OS** | Windows 10 / 11 Enterprise/Pro |
| **Client IP** | Host on `192.168.1.0/24` subnet |
| **Web Server** | Nginx |
| **Cryptography** | OpenSSL (RSA 2048-bit, SHA-256) |

---

## 3. Step-by-Step Implementation

### Step 3.1: Install Nginx

Update system packages and install the Nginx package:

```bash
sudo apt update
sudo apt install -y nginx openssl
```

Enable and start the service:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

*Verification:*
```bash
systemctl is-active nginx
# Expected: active
```

---

### Step 3.2: Create Web Root and Sample Page

Create the site web directory and assign standard permissions:

```bash
sudo mkdir -p /var/www/example.com
sudo chown -R www-data:www-data /var/www/example.com
sudo chmod -R 755 /var/www/example.com
```

Create an `index.html` file:

```bash
sudo bash -c 'cat <<EOF > /var/www/example.com/index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Welcome to example.com</title>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; margin: 40px; background: #f8fafc; color: #1e293b; }
        .card { background: #ffffff; padding: 32px; border-radius: 8px; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1); max-width: 600px; }
        h1 { color: #0284c7; margin-top: 0; }
        .badge { display: inline-block; background: #dcfce7; color: #166534; padding: 4px 10px; border-radius: 9999px; font-weight: 600; font-size: 0.875rem; }
    </style>
</head>
<body>
    <div class="card">
        <span class="badge">HTTPS Active</span>
        <h1>Welcome to example.com</h1>
        <p>Nginx SSL configuration is operational and serving content properly.</p>
        <p><strong>Server Binding:</strong> 192.168.1.1:443</p>
    </div>
</body>
</html>
EOF'
```

---

### Step 3.3: Generate OpenSSL Configuration with SAN

Browsers require the **Subject Alternative Name (SAN)** field. Create the configuration profile:

```bash
sudo mkdir -p /etc/ssl/custom
sudo nano /etc/ssl/custom/example.cnf
```

Paste the following content:

```ini
[req]
default_bits       = 2048
prompt             = no
default_md         = sha256
distinguished_name = dn
x509_extensions    = v3_req

[dn]
C  = KH
ST = Phnom Penh
L  = Phnom Penh
O  = Enterprise Infrastructure Lab
OU = Network Security
CN = example.com

[v3_req]
basicConstraints     = CA:FALSE
keyUsage             = digitalSignature, keyEncipherment
extendedKeyUsage     = serverAuth
subjectAltName       = @alt_names

[alt_names]
DNS.1 = example.com
DNS.2 = www.example.com
IP.1  = 192.168.1.1
```

---

### Step 3.4: Generate Private Key and Certificate

Generate the 2048-bit RSA private key and a self-signed X.509 certificate valid for 365 days:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/custom/example.key \
  -out /etc/ssl/custom/example.crt \
  -config /etc/ssl/custom/example.cnf \
  -extensions v3_req
```

Secure the files with strict permissions:

```bash
sudo chmod 600 /etc/ssl/custom/example.key
sudo chmod 644 /etc/ssl/custom/example.crt
```

*Verification:*
```bash
openssl x509 -in /etc/ssl/custom/example.crt -text -noout | grep -A 2 "Subject Alternative Name"
```

---

### Step 3.5: Configure Nginx Virtual Host

Create the server block with automatic HTTP-to-HTTPS redirect and TLS configuration:

```bash
sudo nano /etc/nginx/sites-available/example.com
```

Paste the following configuration:

```nginx
# 1. HTTP Listener: Force 301 Redirect to HTTPS
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    server_name example.com www.example.com 192.168.1.1;

    return 301 https://$host$request_uri;
}

# 2. HTTPS Listener: Secure Service
server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;

    server_name example.com www.example.com 192.168.1.1;

    # SSL Configuration
    ssl_certificate     /etc/ssl/custom/example.crt;
    ssl_certificate_key /etc/ssl/custom/example.key;

    # Modern Protocol and Cipher Suites
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5:!3DES;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    # Root Directory
    root /var/www/example.com;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

### Step 3.6: Enable Site and Reload Nginx

Remove the default site symlink to prevent port conflicts:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

Enable your new site by creating a symlink:

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
```

Test syntax and reload the service:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

*Expected Verification Output:*
```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

### Step 3.7: Configure Firewall (UFW)

Open both HTTP and HTTPS ports in the firewall:

```bash
sudo ufw allow 80/tcp comment 'Allow HTTP redirect'
sudo ufw allow 443/tcp comment 'Allow HTTPS traffic'
sudo ufw reload
```

---

## 4. Install Certificate on Windows 10 Client

To eliminate security warnings in client browsers, install `example.crt` into the Windows **Trusted Root Certification Authorities** store.

### Method A: Via Command Prompt (Administrator)

1. Copy `example.crt` to the Windows machine (e.g., `C:\example.crt`).
2. Open **Command Prompt as Administrator** and run:

```cmd
certutil -addstore -f "Root" "C:\example.crt"
```

*Expected Output:*
```text
CertUtil: -addstore command completed successfully.
```

### Method B: Via Certificate Manager GUI (`certlm.msc`)

1. Press `Win + R`, type `certlm.msc`, and hit **Enter**.
2. Expand **Trusted Root Certification Authorities** > **Certificates**.
3. Right-click **Certificates** > **All Tasks** > **Import...**.
4. Browse to and select `example.crt`.
5. Select **Place all certificates in the following store** -> verify **Trusted Root Certification Authorities** is selected.
6. Click **Next**, then **Finish**.

---

## 5. Testing & Verification

| Test Scenario | Command / Target | Expected Result | Status |
| :--- | :--- | :--- | :--- |
| **HTTP Redirect** | `curl -I http://192.168.1.1` | `HTTP/1.1 301 Moved Permanently` pointing to `https://...` | Pass |
| **HTTPS Response** | `curl -k -I https://192.168.1.1` | `HTTP/1.1 200 OK` | Pass |
| **Certificate SAN** | `openssl x509 -in example.crt -text` | Contains `DNS:example.com` and `IP:192.168.1.1` | Pass |
| **Browser Access** | `https://example.com` | Secure closed padlock icon with zero SSL errors | Pass |

---

## 6. Common Issues & Solutions

* **Browser shows `NET::ERR_CERT_COMMON_NAME_INVALID`:**  
  The certificate was generated without Subject Alternative Names. Regenerate using the configuration file in Step 3.3.
* **Port 80 conflict on startup (`Address already in use`):**  
  The default configuration was not removed. Run `sudo rm -f /etc/nginx/sites-enabled/default` and reload Nginx.
* **Windows ignores local DNS server:**  
  Disable **Secure DNS (DNS-over-HTTPS)** in your browser settings (Chrome / Edge / Firefox) so it queries your local DNS server directly.