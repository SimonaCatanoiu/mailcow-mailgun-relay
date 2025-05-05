# MailCow with Mailgun as SMTP Relay

## 📦 Overview

This tutorial will guide you through the process of setting up **MailCow** on an Ubuntu VPS and configuring it to use **Mailgun** as an SMTP relay to handle outgoing email traffic. This approach avoids the need to open port 25 directly on your VPS, and instead, you will use Mailgun's SMTP servers to send emails, ensuring that they are delivered reliably.

---

## 📁 Repository Structure

```
.
├── README.md              # This documentation
├── install_mailcow.yml    # Ansible playbook to automate Mailcow setup
└── inventory.ini          # Ansible inventory with host and connection details
```
---

## ⚙️ Inventory Configuration

The `inventory.ini` file specifies the target VPS for Mailcow installation:

```ini
[mailcow]
mailserver ansible_host=<vps_ip> ansible_user=<vps_user>

[all:vars]
ansible_ssh_private_key_file=<vps_key>
```

Ensure the private key path is correct and accessible from your local machine.

---
## 🚀 Prerequisites

#### System Requirements:
- Ubuntu VPS (Hetzner or similar) with public IP (highly recommended to avoid blacklisting)
- **A Fully Qualified Domain Name (FQDN)** for the mail server (example: `mail.testmydom.org`)
- A **Mailgun**, **Namecheap** and **Cloudflare** account.
- Ansible installed on your local machine

#### Ports Required:
- 25 (Blocked by cloud providers for sending mail directly, but we will not need this)
- 80, 110, 143, 443, 465, 587, 993, 995, 4190 (Make sure these are open for MailCow to operate correctly)

#### Minimum VPS Specs:
| Component | Specification |
| --- | --- |
| CPU | 1 GHz |
| RAM | 6 GiB (recommended minimum) |
| Disk | 20 GiB (without emails) |


## 🌐 Domain Configuration

Before setting up MailCow, you'll need a registered domain. In this example, we'll use **Namecheap** for domain registration and **Cloudflare** for DNS management.

### 🪜 Steps:

1. **Add your domain to Cloudflare:**
   - Go to [Cloudflare](https://cloudflare.com/), create an account if you don't have one.
   - Add your domain to Cloudflare.
   - Cloudflare will automatically detect the DNS records from Namecheap.

2. **Update nameservers on Namecheap:**
   - In the **Namecheap Dashboard**, go to **Manage** for your domain, then navigate to **Domain List** → **Nameservers**.
   - Replace the default nameservers with the custom ones provided by Cloudflare. Example:
     - `amy.ns.cloudflare.com`
     - `brad.ns.cloudflare.com`

---

## 🛠️ VPS Setup

Now, let's prepare your VPS. We will use Hetzner in this configuration. For automating the installation, you can use the provided Ansible playbook. The playbook automates the process of installing dependencies and the Mailcow server:

#### 1. 📥 Clone This Repository

```bash
git clone 
cd mailcow-mailgun-relay
```

#### 2. ▶️ Run the Playbook

```bash
ansible-playbook -i inventory.ini install_mailcow.yml
```

#### 🧰 What the Playbook Does

- Installs required system packages
- Installs Docker and Docker Compose
- Clones the Mailcow repository
- Generates Mailcow configuration
- Configures Docker networking based on MTU detection
- Pulls and starts Mailcow containers

#### 🛠 Manual Installation Steps

1. **SSH into your Hetzner VPS:**

   ```bash
   ssh -i <vps_private_key> root@<public_ip>
   ```

2. **Install Docker:**

   ```bash
   curl -sSL https://get.docker.com/ | CHANNEL=stable sh
   systemctl enable --now docker
   ```

3. **Install Docker Compose:**

   ```bash
   apt update
   apt install docker-compose-plugin
   ```

4. **Install Mailcow:**

   ```bash
   umask 0022
   cd /opt
   git clone https://github.com/mailcow/mailcow-dockerized
   cd mailcow-dockerized
   ```

5. **Generate Mailcow Configuration:**

   ```bash
   ./generate_config.sh
   ```

   - Mail server hostname (FQDN): `mail.testmydom.org`
   - Timezone: `Europe/Bucharest`

6. **Check MTU (Maximum Transmission Unit):**

   If the MTU is not set to `1500`, you may need to adjust it. Use the following command to check:

   ```bash
   ip link
   ```

   If needed, modify `/opt/mailcow-dockerized/docker-compose.yml` to set:

   ```yaml
   networks:
     mailcow-network:
       driver_opts:
         com.docker.network.driver.mtu: 1450
   ```

7. **Start Mailcow:**

   ```bash
   docker compose pull
   docker compose up -d
   ```

---

## 📤 Mailgun Setup (SMTP Relay)

1. **Create a Mailgun account** if you don’t have one.

2. **Mailgun Setup**

   * Go to **Sending → Domains → Add Domain**: e.g. `testmydom.org`
   * Add Mailgun DNS records in Cloudflare (SPF, DKIM, MX, DMARC, A) -> copy the **Sending** and **Authentication** records into Cloudflare as following:

        **DNS Records to add (as provided by Mailgun)**:

   * **SPF (Sender Policy Framework):**

     ```
     Name: @
     Type: TXT
     Value: v=spf1 include:mailgun.org ~all
     ```

   * **DKIM (DomainKeys Identified Mail):**

     ```
     Name: mailo._domainkey.testmydom.org
     Type: TXT
     Value: k=rsa; p=MIGfMA0GCSqGSIb3DQE... (public key)
     ```

   * **DMARC:**

     ```
     Name: _dmarc
     Type: TXT
     Value: v=DMARC1; p=quarantine; rua=mailto:dmarc@testmydom.org
     ```

   * **MX Record:**

     ```
     Name: @
     Type: MX
     Value: mail.testmydom.org
     Priority: 10
     ```

   * **A Record:**

     ```
     Name: mail
     Type: A
     Value: <Mailcow IP>
     Proxy: DNS only
     ```

3. **Get SMTP Credentials**

   * Go to Mailgun → Domain → SMTP Credentials → Copy **Username + Password**

4. **Configure Mailcow to use Mailgun as SMTP Relay:**
   - Go to **System → Configuration → Routing**.
   - Add the following under **Sender-dependent transport**:
     - **Host**: `smtp.mailgun.org:587`
     - **Username**: `postmaster@testmydom.org`
     - **Password**: SMTP password from Mailgun

5. **Verify Mailgun SMTP Connection**
   - Use the **Test** button for `smtp.mailgun.org:587`.

---

## 🔐 Admin & User Access

1. **Admin Interface** (default credentials: `admin / moohoo`):

  ```bash
  https://mail.testmydom.org/admin
  ``` 

2. **User Interface**:

  ```bash
  https://mail.testmydom.org
  ```

3. **Domain Admin Interface**:

  ```bash
  https://mail.testmydom.org/domainadmin
  ```

4. **Add Domain in Mailcow**:
   - Go to **Email → Configuration → Add Domain**.
   - Enter domain (`testmydom.org`), click **Add Domain**.

5. **Add Email Address**:
   - Add address (e.g., `no-reply@testmydom.org`).

---

## 🪛 Debugging

If you're facing any issues with Mailcow, you can check the postfix logs with:

```bash
docker-compose logs -f /opt/mailcow-dockerized/postfix-mailcow
```

---

## 📝 Notes

- Inbound mail can only be received if port 25 is open.
- Alternative SMTP Relay services: **SendGrid**, **SMTP2GO**, etc.

---

## 🤝 Contributing

Feel free to submit issues, fork the project and create pull requests. Contributions are welcomed!

---

## 🔗 Useful Links

- [Mailcow Installation Guide](https://docs.mailcow.email/getstarted/install/#initialize-mailcow)
- [Mailcow Relayhosts](https://docs.mailcow.email/manual-guides/Postfix/u_e-postfix-relayhost/)
- [Domain Verification Setup Guide](https://help.mailgun.com/hc/en-us/articles/32884702360603-Domain-Verification-Setup-Guide)
- [Cloudflare DNS Setup Guide](https://help.mailgun.com/hc/en-us/articles/15585722150299-Cloudflare-DNS-Setup-Guide)