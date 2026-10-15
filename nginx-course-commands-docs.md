# Nginx Course - Complete Command & Script Reference
### Every terminal command, config snippet, and script used across the course - Ubuntu Linux & macOS

> Compiled from the actual lecture scripts as they stand today. Where a lecture's demo environment
> is Ubuntu only, the macOS equivalent is added here based on section 2's install/service notes.
> Commands are grouped by lecture so you can jump straight to the one you're recording or
> troubleshooting.

---

## Quick reference - Ubuntu vs macOS

| | Ubuntu (apt) | macOS (Homebrew) |
|---|---|---|
| Install | `sudo apt update && sudo apt install nginx` | `brew install nginx` |
| Start | `sudo systemctl start nginx` (or `enable --now`) | `brew services start nginx` |
| Stop | `sudo systemctl stop nginx` | `brew services stop nginx` |
| Restart | `sudo systemctl restart nginx` | `brew services restart nginx` |
| Reload (no downtime) | `sudo systemctl reload nginx` or `nginx -s reload` | `nginx -s reload` |
| Enable on boot | `sudo systemctl enable nginx` | `brew services start nginx` (brew services auto-manages this) |
| Status | `systemctl status nginx` / `systemctl is-active nginx` | `brew services list` |
| Config root | `/etc/nginx/` | `/opt/homebrew/etc/nginx/` (Apple Silicon) or `/usr/local/etc/nginx/` (Intel) |
| Default port | 80 (needs `sudo`) | 8080 (no root needed, Homebrew default) |
| Logs | `/var/log/nginx/` | `/opt/homebrew/var/log/nginx/` (or `/usr/local/var/log/nginx/`) |
| Sites config pattern | `sites-available` / `sites-enabled` + symlink | not used by default - edit `nginx.conf` or files under `servers/` |
| Test config | `sudo nginx -t` | `nginx -t` |
| Find paths actually compiled in | `nginx -V 2>&1 \| tr ' ' '\n' \| grep -- '--conf-path\|--prefix'` | same |

RHEL / Rocky / CentOS uses `dnf` instead of `apt`: `sudo dnf install nginx`, otherwise `systemctl` commands are identical to Ubuntu.

---

## Section 2 - Installation & Setup

### 2.1 - Installing Nginx on Linux

**Ubuntu / Debian (apt):**
```bash
sudo apt update
sudo apt install nginx
nginx -v
systemctl is-active nginx
systemctl is-enabled nginx
curl -I localhost
sudo ufw allow 'Nginx Full'
```

**RHEL / Rocky (dnf):**
```bash
sudo dnf install nginx
systemctl is-active nginx
sudo systemctl enable --now nginx
curl -I localhost
```

### 2.2 - Installing Nginx on macOS and Windows

**macOS (Homebrew):**
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install nginx
brew --prefix
brew services start nginx
# default port is 8080, not 80 - no sudo needed
```

**Windows (native zip):**
```
cd c:\nginx
start nginx
tasklist /fi "imagename eq nginx.exe"
nginx -s reload
nginx -s quit
nginx -s stop
```

**Windows (WSL2)** - run in PowerShell as Administrator first:
```powershell
wsl --install
```
Then inside the WSL Ubuntu shell:
```bash
sudo apt update
sudo apt install nginx
sudo service nginx start
```

**Final check, all platforms:**
```bash
nginx -v
```

### 2.3 - Nginx Directory Structure & Config Files

```bash
ls /etc/nginx
sudo nginx -T
sudo nano /etc/nginx/nginx.conf
ls -l /etc/nginx/sites-available /etc/nginx/sites-enabled

# enabling a site (symlink pattern)
sudo ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# disabling a site
sudo unlink /etc/nginx/sites-enabled/mysite

ls /var/www/html
sudo tail -f /var/log/nginx/access.log

# find the exact paths THIS build of nginx uses
nginx -V 2>&1 | tr ' ' '\n' | grep -- '--conf-path\|--prefix'
```
> macOS note: Homebrew nginx does not use `sites-available`/`sites-enabled` by default - you generally edit `nginx.conf` directly or `include` files from the `servers/` directory under the Homebrew prefix.

### 2.4 - Managing the Nginx Service

**Ubuntu (systemd):**
```bash
systemctl status nginx
sudo systemctl stop nginx
sudo systemctl start nginx
sudo systemctl restart nginx
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl enable nginx
```

**macOS (Homebrew services / direct binary):**
```bash
brew services start nginx
brew services stop nginx
brew services restart nginx
nginx -s reload
```

---

## Section 3 - Nginx Core Configuration

### 3.1 - Understanding nginx.conf (Main Config)

```bash
sudo nano /etc/nginx/nginx.conf
sudo nginx -t
nproc
ps -eo pid,ppid,user,args | grep nginx | grep -v grep
curl -sI http://127.0.0.1:8080/a/ | grep -i '^x-'
curl -sI http://127.0.0.1:8080/b/ | grep -i '^x-'
sudo nginx -T
```

### 3.2 - Server Blocks / Virtual Hosts

```bash
curl -s -H "Host: www.rajasingh.com" http://127.0.0.1/
sudo ln -s /etc/nginx/sites-available/rajasingh /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/tutorial /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/studio /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo nginx -T | grep "^# configuration file .*sites-enabled"

mkdir -p /var/www/rajasingh /var/www/tutorial /var/www/studio
echo "<h1>www.rajasingh.com</h1>" > /var/www/rajasingh/index.html

# testing all three virtual hosts in one go
for h in www.rajasingh.com tutorial.com studio.com; do
  curl -s -H "Host: $h" http://127.0.0.1/
done
```
Local DNS override for testing (edit the hosts file):
- Linux/macOS: `/etc/hosts`
- Windows: `C:\Windows\System32\drivers\etc\hosts`

### 3.3 - Location Block Matching Rules

```bash
mkdir -p /var/www/assets/static
echo "ROOT-STYLE" > /var/www/assets/static/app.css

# exact / prefix / regex priority testing, across ports 8082-8088
for u in /images/a.png /images/ /docs/readme /images/logo.PNG; do
  curl -s "http://127.0.0.1:8082$u"
done

curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8083/some/path

sudo tail -1 /var/log/nginx/error.log
```

---

## Section 4 - Serving Content

### 4.1 - Serving Static Websites

```bash
find /var/www/portfolio -type f

# enable + reload
sudo ln -s /etc/nginx/sites-available/portfolio /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

for u in / /css/style.css /js/app.js /about/ /nope.html; do
  printf "%-16s -> %s\n" "$u" "$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:8090$u)"
done

sudo tail -1 /var/log/nginx/error.log

# permissions nginx actually needs
sudo chown -R www-data:www-data /var/www/portfolio
sudo find /var/www/portfolio -type d -exec chmod 755 {} \;
sudo find /var/www/portfolio -type f -exec chmod 644 {} \;

# directory listing test (autoindex)
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8090/js/
```
Getting files onto a WSL/Ubuntu box from a Windows drive:
```bash
ls /mnt/f
sudo mkdir -p /var/www/portfolio
sudo unzip "/mnt/f/path/to/site.zip" -d /var/www/portfolio
sudo apt install unzip # if not already present
```

### 4.2 - Custom Error Pages

Error-page config for the portfolio site (`nginx-error-pages.conf`, included inside the `server {}` block, `root` pointing at the folder holding `index.html` and these pages):
```nginx
error_page 404 /404.html;
error_page 403 /403.html;
error_page 500 502 503 504 /50x.html;

# serve the error pages only as error responses, never as direct URLs
location = /404.html { internal; }
location = /403.html { internal; }
location = /50x.html { internal; }
```
```bash
curl -s http://127.0.0.1:8090/nope.html
curl -s -o /dev/null -w "status: %{http_code}\n" http://127.0.0.1:8090/nope.html
curl -s -o /dev/null -w "GET /404.html directly -> %{http_code}\n" http://127.0.0.1:8090/404.html
curl -s -o /dev/null -w "GET /403.html directly -> %{http_code}\n" http://127.0.0.1:8090/403.html
curl -s http://127.0.0.1:8091/api/
curl -sI http://127.0.0.1:8090/nope.html | grep -i '^server'
```
Key directives: `error_page` (404, 403, and the 500/502/503/504 group), `internal`, `server_tokens off;`

### 4.3 - Redirects & Rewrites

```bash
curl -sI http://127.0.0.1:8092/old-page
curl -sI http://127.0.0.1:8093/search?q=nginx # $uri vs $request_uri query-string test
sudo tail -1 /var/log/nginx/error.log # "rewrite or internal redirection cycle"
```
Key directives: `return 301/302`, `rewrite ... last|break|permanent`.

---

## Section 5 - Reverse Proxy & Load Balancing

### 5.1 - Reverse Proxy Configuration

```bash
curl -s -H "Host: app.local" http://127.0.0.1:8096/hello

# X-Forwarded-For chaining test
curl -H "X-Forwarded-For: 203.0.113.9" http://127.0.0.1:8096/hello

# timeouts / slow backend
curl -o /dev/null -w "%{http_code} in %{time_total}s\n" http://127.0.0.1:8096/slow

# kill the backend, observe 502
curl -s -o /dev/null -w "status: %{http_code}\n" http://127.0.0.1:8096/hello
```
Key directives: `proxy_pass`, `proxy_set_header Host/X-Real-IP/X-Forwarded-For/X-Forwarded-Proto`, `proxy_connect_timeout`/`proxy_send_timeout`/`proxy_read_timeout`, `proxy_buffer_size`/`proxy_buffers`/`proxy_busy_buffers_size`, `proxy_buffering off`.

### 5.3 - Proxying to Node.js Apps

systemd service for a Node app (`/etc/systemd/system/myapp.service`):
```ini
[Service]
ExecStart=/usr/bin/node /var/www/myapp/server.js
Restart=always
User=www-data
Environment=PORT=3000 NODE_ENV=production

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl enable --now myapp
```

---

## Section 6 - Security & SSL

### 6.1 - SSL/TLS on Localhost with OpenSSL

```bash
sudo mkdir -p /etc/nginx/ssl
cd /etc/nginx/ssl

sudo openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout localhost.key -out localhost.crt -days 365 \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

openssl x509 -in localhost.crt -noout -subject -issuer -dates -ext subjectAltName

sudo nginx -t && sudo systemctl reload nginx
curl -I "http://localhost/pricing?plan=pro"

curl https://localhost
openssl s_client -connect localhost:443 -servername localhost

curl --cacert localhost.crt https://localhost # trust exactly this cert
curl -k https://localhost # skip the check entirely (dev only)

# verify a cert/key pair actually match
openssl x509 -noout -modulus -in localhost.crt | openssl sha256
openssl rsa -noout -modulus -in localhost.key | openssl sha256

# prove old TLS is refused
openssl s_client -connect localhost:443 -tls1_1
```
Key directives: `listen 443 ssl;`, `ssl_certificate`, `ssl_certificate_key`, `ssl_protocols TLSv1.2 TLSv1.3;`, `return 301 https://$host$request_uri;` (HTTP->HTTPS redirect).
> Windows/Git Bash note: `-subj "/CN=localhost"` can get mangled by Git Bash's path conversion - record on Ubuntu/WSL, or use `-subj "//CN=localhost"`.

### 6.2 - Security Headers & Best Practices

```bash
curl -sI https://example.com/
```
Config block:
```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Content-Security-Policy "frame-ancestors 'none'" always;
add_header Permissions-Policy "geolocation=(), camera=(), microphone=()" always;
```
Testing headers survive error responses:
```bash
for path in / /nope /boom/; do
  code=$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:$PORT$path)
  curl -sI http://127.0.0.1:$PORT$path | grep -ci '^x-content-type-options\|^x-frame-options\|^referrer-policy'
done
```
Delete (deprecated, can introduce the bug it claims to prevent): `add_header X-XSS-Protection "1; mode=block";`

### 6.3 - Rate Limiting & Access Control

```nginx
limit_req_zone $binary_remote_addr zone=basic:10m rate=2r/s;
limit_req zone=basic burst=5 nodelay;

set_real_ip_from 10.0.0.0/8;
real_ip_header X-Forwarded-For;

location /admin/ { allow 192.0.2.0/24; deny all; }
location /private/ { auth_basic "Restricted"; auth_basic_user_file /etc/nginx/.htpasswd; }
```
```bash
# hammer an endpoint to see 503s from limit_req
for i in $(seq 1 10); do curl -s -o /dev/null -w "%{http_code} " http://127.0.0.1:8130/ ; done

# create basic-auth credentials
sudo apt install apache2-utils # Ubuntu - provides htpasswd
sudo htpasswd -c /etc/nginx/.htpasswd admin

# simulate several visitors behind one proxy IP
for u in 203.0.113.10 203.0.113.20 203.0.113.30; do
  curl -s -o /dev/null -w '%{http_code} ' -H "X-Forwarded-For: $u" http://127.0.0.1:$PORT/
done
```
> macOS equivalent for `htpasswd`: it ships with Apache tools via Homebrew (`brew install httpd`) or use `openssl passwd` to generate a hash manually.

---

## Section 7 - Performance Optimization

### 7.1 - Gzip Compression

```nginx
gzip on;
gzip_vary on;
gzip_min_length 256;
gzip_comp_level 6;
gzip_proxied any;
gzip_types text/plain text/css application/json application/javascript
           text/xml application/xml image/svg+xml;
```
```bash
curl -sI -H 'Accept-Encoding: gzip' https://example.com/style.css

# prove gzip_proxied matters once a CDN/LB is in front
curl -s -o /dev/null -w '%{size_download}\n' -H 'Accept-Encoding: gzip' -H 'Via: 1.1 cloudflare' https://example.com/style.css
```

### 7.2 - Caching Static Assets

```nginx
location ~* \.(css|js|woff2|png|jpg|svg)$ {
    add_header Cache-Control "public, max-age=31536000, immutable" always;
    access_log off;
}
location = /index.html {
    add_header Cache-Control "no-cache" always;
}
```
```bash
curl -sI http://example.com/assets/app.a1b2c3d4.css

# the ETag / 304 handshake
ETAG=$(curl -sI http://example.com/assets/app.css | grep -i '^etag' | sed 's/^[Ee][Tt]ag: //' | tr -d '\r\n')
curl -s -o /dev/null -w '%{http_code} %{size_download}\n' -H "If-None-Match: $ETAG" http://example.com/assets/app.css
```

---

## Section 8 - Logging & Monitoring

### 8.1 - Access & Error Logs

```nginx
log_format timed '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" "$http_user_agent" '
                  'rt=$request_time urt=$upstream_response_time';
access_log /var/log/nginx/access.log timed;

# behind a proxy, log the REAL client address instead of the proxy's
log_format whoami '$http_x_forwarded_for - "$request" $status';
```
Reading logs like a professional - one-liners:
```bash
awk '{print $9}' access.log | sort | uniq -c | sort -rn # status-code counts
awk '$9==404 {print $7}' access.log | sort | uniq -c | sort -rn # what's 404-ing
awk '{print $7}' access.log | sort | uniq -c | sort -rn | head # top pages
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head # busiest IPs
```

### 8.2 - Log Rotation & Monitoring

```bash
cat /etc/logrotate.d/nginx
mv rotate-demo.log rotate-demo.log.1
ls -l /proc/$(pgrep -f 'nginx: worker')/fd | grep rotate-demo
kill -USR1 $(cat /run/nginx.pid) # tell nginx to reopen its log files post-rotation
systemctl list-timers certbot # example of a systemd timer worth knowing
df -h /var
du -sh /var/log/nginx
awk '$9>=500' access.log | wc -l
sudo tail -f /var/log/nginx/error.log
```

---

## Appendix - macOS path cheat sheet

| Item | Apple Silicon (Homebrew) | Intel (Homebrew) |
|---|---|---|
| Prefix | `/opt/homebrew` | `/usr/local` |
| Config | `/opt/homebrew/etc/nginx/nginx.conf` | `/usr/local/etc/nginx/nginx.conf` |
| Logs | `/opt/homebrew/var/log/nginx/` | `/usr/local/var/log/nginx/` |
| Default document root | `/opt/homebrew/var/www` | `/usr/local/var/www` |
| Run `brew --prefix` to confirm which one applies on a given machine. | | |

macOS has no `systemctl`/`ufw` - service control is `brew services ...` or the `nginx -s <signal>` commands (`reload`, `quit`, `stop`), and firewall rules are managed through System Settings -> Network -> Firewall or `pfctl`, not shown elsewhere in this course.
