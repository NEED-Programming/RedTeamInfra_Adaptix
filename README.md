# Red Team Infrastructure Checklist — Adaptix C2

---

## 1. Pre-flight: OPSEC & Identity Separation

- [ ] Set up a dedicated privacy-focused email address
  - Use ProtonMail or similar — not tied to your real identity. Used for all accounts below.
- [ ] Create a single-use virtual card via Privacy.com
  - Free virtual cards, ties to a US bank account. Use this burner card for **all** purchases — DigitalOcean, GoDaddy, and any other services.
  - **Monero (XMR)** — best option for full separation, accepted by Njalla
  - **Bitcoin** — accepted by Njalla, less private than Monero but workable
- [ ] Generate SSH keypair locally before anything else
  - `ssh-keygen`
- [ ] Validate the public key is ready to paste into DigitalOcean
  - `type <keys.pub>`

---

## 2. Domain Acquisition

Choose a neutral-sounding domain name first
- e.g. `cdn-assets-[random].com` or `static-[random]-files.net` — avoid pentest, red, hack.
- [ ]**OR**
- [ ] Browse expireddomains.net for seasoned domains
  - Go to `Domain Lists → GoDaddy Closeout Domains` → click the Price filter to find the cheapest option.
- [ ] Check ABY (birth year) of the domain
  - Older domains have more trust — aged/seasoned domains raise less suspicion with blue teams.
- [ ] Click into the domain and review domain age details
  - Confirm it has a clean history and reasonable age.
- [ ] Check domain reputation via VirusTotal
  - Search the domain — confirm no security vendors have flagged it as malicious.
- [ ] Purchase the domain via GoDaddy
  - Pay with your Privacy.com burner card.

---

## 3. Droplet Provisioning

### Droplet 1 — Adaptix C2 (Team Server)

- [ ] Create DigitalOcean account using your privacy email
- [ ] Add your Privacy.com burner payment method to the account
- [ ] Create Droplet with these specs:
  - **Name:** `Adaptix`
  - **OS:** Ubuntu 24.04 LTS x64
  - **Spec:** 2 vCPU / 2 GB RAM — $18/mo
  - **Region:** NYC
- [ ] Paste your SSH public key during Droplet creation
- [ ] Disable password authentication at creation time

### Droplet 2 — Redirector (Public-Facing Proxy)

- [ ] Create second Droplet with these specs:
  - **Name:** `Redirector`
  - **OS:** Ubuntu 24.04 LTS x64
  - **Spec:** 1 vCPU / 1 GB RAM — $6/mo
  - **Region:** NYC
- [ ] Paste your SSH public key during Droplet creation
- [ ] Disable password authentication at creation time

### Post-Creation

- [ ] Validate both droplets are up and running in the DO console
- [ ] Note both Droplet IPs — needed for DNS records, firewall rules, and Nginx proxy config

---

## 4. Cloudflare & DNS

- [ ] Add site in Cloudflare
  - `Overview → Add a Site → Connect a domain`
- [ ] Enter your purchased domain name
- [ ] Change DNS to manual
  - Prevents Cloudflare from auto-scanning existing records.
- [ ] Create A record (proxied)
  - `Type=A, Name=@, IPv4=<redirector IP>, Proxy Status=Enabled`
- [ ] Locate the two Cloudflare nameservers assigned to your site
- [ ] Swap nameservers on GoDaddy
  - `GoDaddy → DNS → Name Servers → Change Name Servers` → paste the two Cloudflare NS records.
- [ ] Click Validate on the Cloudflare overview page
- [ ] Confirm it works — visit the domain, expect a Cloudflare "Web Server is down" page
  - This is expected — Nginx isn't running on the Redirector yet.

### TLS Settings

- [ ] Set SSL/TLS mode to Full (Strict)
  - `SSL/TLS → Configure → Full (Strict)` — end-to-end encryption with validated origin cert.
- [ ] Enable HSTS and enforce minimum TLS 1.2
  - `SSL/TLS → Edge Certificates → Enable HSTS → Minimum TLS Version: 1.2`

### API Token for Certbot

- [ ] Create API Token for DNS zone editing
  - `Search → API Tokens → Create API Token → Edit Zone DNS template → Zone Resources: Include, Specific Zone, <your domain> → Continue to Summary → Create Token`
- [ ] **Save this token** — needed for the Certbot Cloudflare DNS plugin on the Redirector

---

## 5. Hardening & Firewall

### Both Droplets

- [ ] Connect to each Droplet via Web Console
- [ ] Update and upgrade the Ubuntu image
  - `apt update && apt upgrade -y`
- [ ] Install and enable fail2ban
  - `apt install -y fail2ban && systemctl enable --now fail2ban`

### UFW — Adaptix C2 Droplet

- [ ] Configure firewall rules
  ```bash
  ufw default deny incoming
  ufw default allow outgoing
  ufw allow ssh
  ufw allow from <redirector-IP> to any port 443 proto tcp   # C2 HTTPS from redirector only
  ufw allow 1738/tcp                                          # Adaptix C2 operator port
  ufw allow in 53/udp from <redirector-IP>                    # DNS from redirector
  ufw allow in 53/tcp from <redirector-IP>                    # DNS from redirector
  ufw allow in <custom-port>/tcp from <redirector-IP>         # Gopher from redirector
  ufw enable
  ```
- [ ] Verify rules
  - `ufw status numbered`

### UFW — Redirector Droplet

- [ ] Configure firewall rules
  ```bash
  ufw default deny incoming
  ufw default allow outgoing
  ufw allow ssh
  ufw allow 80/tcp                                            # HTTP
  ufw allow 443/tcp                                           # HTTPS
  ufw allow out to <adaptix-IP> port 443 proto tcp            # Forward C2 to Adaptix
  ufw allow 53/udp                                            # DNS inbound
  ufw allow 53/tcp                                            # DNS inbound
  ufw allow out 53/udp to <adaptix-IP>                        # DNS forward to Adaptix
  ufw allow out 53/tcp to <adaptix-IP>                        # DNS forward to Adaptix
  ufw allow <custom-port>/tcp                                 # Gopher inbound
  ufw allow out <custom-port>/tcp to <adaptix-IP>             # Gopher forward to Adaptix
  ufw enable
  ```
- [ ] Verify rules
  - `ufw status numbered`

---

## 6. TLS & Nginx Redirector

> All steps below are run on the **Redirector** Droplet.

### Certbot via Cloudflare DNS Plugin

- [ ] Install required packages
  - `apt install certbot nginx python3-certbot-nginx python3-certbot-dns-cloudflare curl gnupg2 ca-certificates lsb-release`
- [ ] Create Cloudflare credentials file
  ```bash
  mkdir .secrets && touch .secrets/cloudflare.ini
  echo 'dns_cloudflare_api_token = "$CLOUDFLARETOKEN"' >> .secrets/cloudflare.ini
  cat .secrets/cloudflare.ini
  ```
- [ ] Issue cert via Cloudflare DNS challenge
  ```bash
  certbot certonly --dns-cloudflare \
    --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
    --dns-cloudflare-propagation-seconds 30 \
    -d <domain.com> -d <sub.domain.com>
  ```
  - Can also use `-d *.domain.com` for wildcard. All options produce the cert directory for `domain.com`.

### Nginx Redirector Config

- [ ] Create Nginx config file
  - `touch /etc/nginx/sites-available/redirector && nano /etc/nginx/sites-available/redirector`
- [ ] Write config:
  ```nginx
  # HTTP to HTTPS redirect
  server {
      listen 80;
      listen [::]:80;
      server_name <domain.com>;                   ### CHANGE THIS
      return 301 https://$host$request_uri;
  }

  # Main redirector server block
  server {
      listen 443 ssl http2;
      listen [::]:443 ssl http2;
      server_name <domain.com>;                   ### CHANGE THIS

      ssl_certificate     /etc/letsencrypt/live/<domain.com>/fullchain.pem;   ### CHANGE PATH
      ssl_certificate_key /etc/letsencrypt/live/<domain.com>/privkey.pem;     ### CHANGE PATH

      # Security Headers
      add_header X-Frame-Options "SAMEORIGIN" always;
      add_header X-XSS-Protection "1; mode=block" always;
      add_header X-Content-Type-Options "nosniff" always;
      add_header Referrer-Policy "no-referrer" always;
      server_tokens off;

      # === C2 TRAFFIC — /<URI> PATH ONLY ===
      location ~ ^/<URI>(/.*)?$ {                 ### Adaptix URI path
          proxy_pass              https://<AdaptixC2 Droplet>;   ### Adaptix VPS IP
          proxy_ssl_verify        off;
          proxy_set_header        Host $host;
          proxy_set_header        X-Real-IP $remote_addr;
          proxy_set_header        X-Forwarded-For $http_cf_connecting_ip;
          proxy_set_header        X-Forwarded-Proto $scheme;

          # WebSocket support
          proxy_http_version      1.1;
          proxy_set_header        Upgrade $http_upgrade;
          proxy_set_header        Connection "upgrade";
          proxy_read_timeout      86400;
          proxy_send_timeout      86400;
          proxy_buffering         off;
          proxy_cache_bypass      $http_upgrade;
      }

      # === EVERYTHING ELSE — DECOY ===
      location / {
          default_type text/plain;
          return 200 "1";
      }

      # Hidden files — decoy
      location ~ /\. {
          default_type text/plain;
          return 200 "1";
      }
  }
  ```
- [ ] Use nano search-and-replace to swap all `<domain.com>` placeholders
  - `Ctrl + \` in nano.
- [ ] Enable site and remove default
  ```bash
  ln -s /etc/nginx/sites-available/redirector /etc/nginx/sites-enabled
  rm /etc/nginx/sites-enabled/default
  ```
- [ ] Test config
  - `nginx -t` — confirm syntax is OK.
- [ ] Reload and verify
  - `systemctl reload nginx && systemctl status nginx`
- [ ] Validate decoy response
  - Visit the domain in a browser — you should receive a plain text `1`.

---

## 7. Adaptix C2 Deployment

> All steps below are run on the **Adaptix** Droplet unless noted.

### Server Build

- [ ] Clone the Adaptix C2 repository
  - `git clone https://github.com/Adaptix-Framework/AdaptixC2.git`
- [ ] Run pre-install script for server
  - `bash ./pre_install_linux_all.sh server`
- [ ] Build the server with extensions
  - `make server-ext`

### SSL Keys for C2 Comms

- [ ] Generate self-signed SSL certificate
  ```bash
  openssl req -x509 -nodes -newkey rsa:2048 \
    -keyout server.rsa.key -out server.rsa.crt -days 3650
  ```
- [ ] Copy SSL files to dist directory
  - `cp server.rsa.* ./dist`

### Profile Configuration

- [ ] Edit the Adaptix C2 profile
  - `cd dist && nano profile.yaml`
  - Change `Port` and `Server` to match your setup.

### Docker

- [ ] Install Docker
  - `curl -fsSL https://get.docker.com/ | sh`
- [ ] Initialize Docker
  ```bash
  newgrp docker
  systemctl start docker
  systemctl enable docker
  ```
- [ ] Verify Docker is working
  - `docker run hello-world`
- [ ] Clean up test container
  - `docker container ls -all && docker container stop <Container_ID>`

### Swap (Virtual Memory)

- [ ] Create 8 GB swap file
  ```bash
  sudo fallocate -l 8G /swapfile
  chmod 600 /swapfile
  mkswap /swapfile
  swapon /swapfile
  echo 'swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
  swapon --show
  ```

### Client Build

- [ ] Build the Adaptix client
  ```bash
  cd ~/AdaptixC2
  bash ./pre_install_linux_all.sh client
  make docker-build-client
  ```
- [ ] Note the client AppImage filename in `/AdaptixClient/client-dist`

### BOF Extension Kit

- [ ] Clone the Extension Kit
  - `git clone https://github.com/Adaptix-Framework/Extension-Kit.git`
- [ ] Install mingw cross-compiler
  - `apt install g++-mingw-w64-x86-64-posix gcc-mingw-w64-x86-64-posix mingw-w64-tools`

### Start the Team Server

- [ ] Open a new Web Console session
- [ ] Start the Adaptix server
  ```bash
  cd ~/AdaptixC2/dist
  sudo ./adaptixserver -profile profile.yaml
  ```

### Transfer Client to Local Machine

- [ ] SCP the client binary to your local machine
  - `scp -i <key> root@ip:/path/to/client_appimage /path/to/save/locally`
- [ ] Load BOF extensions in the client
  - `Extensions → Script Manager → Load New → /path/to/extension-kit.axs`

---

## 8. Listener, Beacon & Execution

> **Warning:** Confirm signed SOW and rules of engagement before proceeding.
> Running C2 against an unauthorized target is a serious crime regardless of what you find.

### Create HTTP Listener

- [ ] Open the Adaptix client and connect to the team server
- [ ] Create a new listener
  - `Listener → Right Click → Create`
- [ ] Configure beacon settings:
  - **Config:** `BeaconHTTP`
  - **Host Port & Bind:** `0.0.0.0:443`
  - **Callback Address:** `<domain.com>:port`
  - **URI:** Custom path — must match the Nginx `location` block
  - **Use SSL/TLS:** Enabled

### HTTP Headers

- [ ] Enable `Trust X-Forwarded-For`
  - Required since Cloudflare sits in front — the real client IP comes via this header.

### Generate Agent

- [ ] Generate the agent beacon from the listener config
- [ ] Adjust jitter, kill date, and other parameters to suit the engagement scope

### Execution

- [ ] Transfer the agent to the target and execute
- [ ] Confirm beacon check-in appears in the Adaptix client
- [ ] Kill the beacon when the engagement is complete

---

## 9. OPSEC Hygiene — Per Engagement

- [ ] Set beacon jitter appropriate to the engagement
  - Fast beacons are a reliable EDR trigger.
- [ ] Set a kill date on every agent
  - Agents should not outlive the engagement scope.
- [ ] Verify firewall rules before each engagement
  - `ufw status numbered` on both droplets — confirm C2 ports are still locked to redirector IP only.
- [ ] Use a separate agent per target and engagement
  - Never reuse agents across engagements.
- [ ] Customise all URI paths away from defaults
  - Pick paths that look like routine app traffic.
  - Update both the **Nginx location block** and the **Adaptix listener URI** to match — they must be identical.
- [ ] Rotate and purge logs after each engagement
  - `shred -u /var/log/nginx/*.log /var/log/auth.log`
- [ ] Clear shell history before teardown
  - `history -c && cat /dev/null > ~/.bash_history`
- [ ] Destroy **both** Droplets via DigitalOcean console after engagement
  - Ephemeral infra is the strongest OPSEC — don't reuse across engagements.
