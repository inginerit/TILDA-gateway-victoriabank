# VictoriaBank Payment Gateway — Tilda Integration

Installation and configuration guide for merchants and developers.

---

## Overview

The Tilda Gateway is a standalone PHP application that acts as a payment bridge between your Tilda website and VictoriaBank. It handles all payment methods: Visa/Mastercard, StarCard Rate, StarCard Points, and MIA (QR instant payment).

**Architecture:**
- Tilda sends the customer to `https://pay.yoursite.md/pay?...` after order creation
- The gateway redirects the customer to the bank
- The bank POSTs results back to the gateway callback URL
- The gateway updates the order status and notifies your backend (optional webhook)

---

## Requirements

- PHP 8.1 or higher (with `openssl`, `pdo_mysql`, `curl` extensions)
- MySQL 5.7+ or MariaDB 10.3+
- A dedicated subdomain or path for the gateway (e.g. `https://pay.yoursite.md`)
- HTTPS with a valid SSL certificate
- RSA key pair issued by VictoriaBank (for e-Gateway)
- MIA API credentials from VictoriaBank (for QR payments)

---

## 1. Installation

### 1.1 Copy files to server

Upload the gateway directory to your server. Recommended location:

```
/var/www/pay.yoursite.md/
├── config.json          ← your configuration (do NOT commit to git)
├── config.json.template ← template (safe to commit)
├── config.php           ← configuration loader
├── index.php            ← main router
├── install.php          ← DB installer (delete after use)
├── src/                 ← PHP classes
├── pages/               ← route handlers
├── admin/               ← admin dashboard
└── logs/                ← log files (must be writable)
```

### 1.2 Web server configuration

**Apache** — create a VirtualHost:

```apache
<VirtualHost *:443>
    ServerName pay.yoursite.md
    DocumentRoot /var/www/pay.yoursite.md

    SSLEngine on
    SSLCertificateFile    /etc/ssl/yoursite.md.crt
    SSLCertificateKeyFile /etc/ssl/yoursite.md.key

    <Directory /var/www/pay.yoursite.md>
        AllowOverride All
        Options -Indexes
        Require all granted
    </Directory>
</VirtualHost>
```

Create `.htaccess` in the gateway root:

```apache
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*)$ index.php [QSA,L]
```

**nginx** — server block:

```nginx
server {
    listen 443 ssl;
    server_name pay.yoursite.md;
    root /var/www/pay.yoursite.md;
    index index.php;

    ssl_certificate     /etc/ssl/yoursite.md.crt;
    ssl_certificate_key /etc/ssl/yoursite.md.key;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # Block direct access to config and src
    location ~ ^/(config|src|logs)/ {
        deny all;
    }
}
```

### 1.3 File permissions

```bash
chmod 755 /var/www/pay.yoursite.md
chmod 644 /var/www/pay.yoursite.md/config.json
chmod 777 /var/www/pay.yoursite.md/logs
```

---

## 2. Configuration

### 2.1 Create config.json

Copy the template and fill in your values:

```bash
cp config.json.template config.json
```

Edit `config.json`:

```json
{
  "test_mode": false,

  "admin_password": "your_strong_admin_password",

  "db": {
    "host": "localhost",
    "name": "vb_gateway",
    "user": "vb_user",
    "pass": "vb_password"
  },

  "egw": {
    "terminal_id":      "YOUR_VISA_TERMINAL_ID",
    "merchant_id":      "YOUR_VISA_MERCHANT_ID",
    "merchant_name":    "Your Shop Name",
    "merchant_url":     "https://yoursite.md",
    "merchant_address": "str. Exemplu 1, Chisinau, Moldova",
    "merchant_gmt":     "+02:00",
    "language":         "RO",
    "private_key_path": "/var/secure/vb_private.pem",
    "public_key_path":  "/var/secure/victoria_pub.pem",

    "starcard_terminal_id": "YOUR_STARCARD_TERMINAL_ID",
    "starcard_merchant_id": "YOUR_STARCARD_MERCHANT_ID"
  },

  "mia": {
    "api_username":   "your_mia_api_username",
    "api_password":   "your_mia_api_password",
    "merchant_iban":  "MD99VBBL00000000000000",
    "vbca_cert_path": "/var/secure/VBCA.crt"
  },

  "payment_methods": {
    "visa_master":     true,
    "starcard_rate":   true,
    "starcard_points": true,
    "mia":             true
  },

  "currency": "MDL",

  "gateway_url": "https://pay.yoursite.md",

  "log_dir": "/var/log/vb-gateway"
}
```

> **Security:** Store RSA key files and the VBCA certificate outside the web root (e.g. `/var/secure/`). Set permissions `chmod 640` on key files.

### 2.2 RSA key setup

VictoriaBank provides:
- `private.pem` — your merchant private key (signs payment requests)
- `victoria_pub.pem` — bank's public key (verifies P_SIGN in callbacks)

For StarCard: if VictoriaBank issued separate credentials, provide separate key files. If StarCard uses the same terminal, leave `starcard_terminal_id` and `starcard_merchant_id` empty — the plugin falls back to the main Visa/Mastercard values automatically.

### 2.3 Install the database

1. Create a MySQL database and user:

```sql
CREATE DATABASE vb_gateway CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'vb_user'@'localhost' IDENTIFIED BY 'vb_password';
GRANT ALL PRIVILEGES ON vb_gateway.* TO 'vb_user'@'localhost';
FLUSH PRIVILEGES;
```

2. Run the installer by opening in your browser:

```
https://pay.yoursite.md/install
```

You should see:
```
✓ Table `vb_orders` created
✓ Table `vb_mia_tokens` created
✓ Installation complete!
```

3. **Delete `install.php` immediately after installation:**

```bash
rm /var/www/pay.yoursite.md/install.php
```

---

## 3. Callback URLs — Register with VictoriaBank

> **This is the most important configuration step.** You must provide these URLs to VictoriaBank so the bank can POST payment results to your server.

### Visa / Mastercard + StarCard Rate + StarCard Points

All card payment methods share a single callback URL:

```
https://pay.yoursite.md/callback
```

This endpoint handles TRTYPE=0 (authorization), TRTYPE=21 (completion), and TRTYPE=24 (refund/reversal) for Visa/Mastercard, StarCard Rate, and StarCard Points.

### MIA IPS — Webhook (push notification)

```
https://pay.yoursite.md/mia/webhook
```

> **Note:** Replace `pay.yoursite.md` with your actual gateway domain. These must be publicly accessible HTTPS URLs. No password or authentication is required — the gateway verifies signatures internally.

#### How callbacks work

| TRTYPE | Sender | Purpose | Gateway action |
|--------|--------|---------|----------------|
| 0 | Browser (client redirect) | Authorization confirmed | Verify P_SIGN → save RRN/INT_REF → send TRTYPE=21 → return HTTP 200 |
| 21 | Bank server (server-to-server) | Completion confirmed | Verify P_SIGN → mark order `paid` → notify merchant webhook |
| 24 | Bank server (server-to-server) | Refund confirmed | Verify P_SIGN → mark order `refunded` |

**P_SIGN is verified on every callback.** If verification fails, the gateway returns HTTP 200 without updating the order (prevents information leakage to attackers).

---

## 4. Tilda Integration

### 4.1 Tilda → Gateway redirect

After a customer places an order in Tilda, configure Tilda to redirect to the gateway payment page:

```
https://pay.yoursite.md/pay?method=visa_master&order_id={order_id}&amount={amount}&currency=MDL&description={description}&return_url={return_url}&cancel_url={cancel_url}&notify_url={notify_url}
```

Parameters:

| Parameter | Required | Description |
|-----------|----------|-------------|
| `method` | Yes | `visa_master`, `starcard_rate`, `starcard_points`, or `mia` |
| `order_id` | Yes | Your internal order ID (string, max 50 chars) |
| `amount` | Yes | Amount to charge (e.g. `150.00`) |
| `currency` | No | Currency code (default: `MDL`) |
| `description` | No | Payment description shown to customer |
| `return_url` | No | Where to redirect after successful payment |
| `cancel_url` | No | Where to redirect after cancelled/failed payment |
| `notify_url` | No | Your backend URL to receive payment notifications (JSON POST) |

### 4.2 Payment method links

Create buttons or links in Tilda pointing to:

**Visa / Mastercard:**
```
https://pay.yoursite.md/pay?method=visa_master&order_id=ORDER_ID&amount=AMOUNT
```

**StarCard Rate (installments):**
```
https://pay.yoursite.md/pay?method=starcard_rate&order_id=ORDER_ID&amount=AMOUNT
```

**StarCard Points:**
```
https://pay.yoursite.md/pay?method=starcard_points&order_id=ORDER_ID&amount=AMOUNT
```

**MIA (QR instant payment):**
```
https://pay.yoursite.md/pay?method=mia&order_id=ORDER_ID&amount=AMOUNT
```

### 4.3 Merchant webhook (notify_url)

When a card payment is completed (TRTYPE=21 received from bank), the gateway sends a POST request to your `notify_url`:

```json
{
  "order_id": "your-order-123",
  "status": "paid",
  "amount": "150.00",
  "currency": "MDL",
  "rrn": "123456789012",
  "timestamp": "2026-04-10T14:30:00Z"
}
```

Your backend must return HTTP 200. The gateway retries once if the first attempt fails.

---

## 5. Gateway Pages

| URL | Method | Description |
|-----|--------|-------------|
| `/pay` | GET | Payment initiation — redirects to bank or shows QR |
| `/callback` | POST | e-Gateway callback (Visa/Master/StarCard) |
| `/mia/webhook` | POST | MIA JWT push notification from bank |
| `/mia/status` | GET | AJAX polling — returns JSON payment status |
| `/mia/payment` | GET | QR payment page shown to customer |
| `/admin` | GET | Admin dashboard (password protected) |
| `/install` | GET | DB installer — delete after use |

---

## 6. Admin Dashboard

Access the admin panel at:

```
https://pay.yoursite.md/admin
```

Login with the password set in `config.json` → `admin_password`.

The dashboard shows:
- Recent transactions
- Payment status overview
- Ability to trigger manual refunds

---

## 7. Refunds

### Card refund (Visa/Mastercard, StarCard)

From the admin dashboard, click "Refund" on a completed transaction.

Or via API call from your backend:

```
POST https://pay.yoursite.md/admin/refund
Content-Type: application/json

{
  "admin_password": "your_admin_password",
  "order_id": "your-order-123"
}
```

### MIA refund

MIA refunds are initiated via the admin dashboard. The gateway calls the MIA API DELETE endpoint automatically.

---

## 8. Logs

Logs are written to the directory configured in `config.json` → `log_dir` (default: `logs/` inside the gateway directory).

Log files:
- `gateway.log` — all payment events
- `error.log` — errors only

Example entries:

```
[2026-04-10 14:30:01] INFO  Callback received: ORDER=000001 TRTYPE=0 ACTION=0 RC=00
[2026-04-10 14:30:01] INFO  TRTYPE=0: authorized — sending TRTYPE=21
[2026-04-10 14:30:02] INFO  TRTYPE=21 sent: RC=00
[2026-04-10 14:30:05] INFO  TRTYPE=21: payment completed: order_id=your-order-123
[2026-04-10 14:35:10] INFO  MIA status poll: uuid=abc123... status=Paid
```

Ensure the log directory is writable by the web server:

```bash
chmod 777 /var/log/vb-gateway
# or
chown www-data:www-data /var/log/vb-gateway
```

---

## 9. Production Checklist

- [ ] `"test_mode": false` in `config.json`
- [ ] `"gateway_url"` set to your real HTTPS domain
- [ ] RSA key files outside web root, readable by web server user only
- [ ] MIA `VBCA.crt` certificate file in place
- [ ] `install.php` deleted
- [ ] Callback URLs registered with VictoriaBank:
  - Visa/Mastercard: `https://pay.yoursite.md/callback`
  - StarCard Rate: `https://pay.yoursite.md/callback`
  - StarCard Points: `https://pay.yoursite.md/callback`
  - MIA webhook: `https://pay.yoursite.md/mia/webhook`
- [ ] HTTPS certificate valid (not self-signed)
- [ ] Log directory writable
- [ ] Admin password changed from default
- [ ] `.htaccess` or nginx `deny all` configured for `/config`, `/src`, `/logs`
- [ ] Tested a full payment cycle in test mode before going live

---

## 10. Troubleshooting

### P_SIGN verification fails
- Ensure `victoria_pub.pem` is the bank's public key, not your own
- Check file permissions and path in `config.json`
- In test mode, use the test public key provided by the bank

### Callback not received
- Verify the callback URL is publicly accessible (not localhost/tunnel)
- Check that HTTPS certificate is valid
- Review logs for incoming POST requests

### MIA QR not loading
- Verify `api_username` and `api_password` credentials
- Check that `VBCA.crt` path is correct and file is readable
- Review gateway logs for authentication errors

### Mixed Content errors
- Ensure `"gateway_url"` starts with `https://`
- If behind Cloudflare or a proxy, ensure the proxy forwards HTTPS headers

### Database connection fails
- Verify MySQL credentials in `config.json`
- Ensure the database and user exist with correct permissions
- Check that MySQL is running: `systemctl status mysql`
