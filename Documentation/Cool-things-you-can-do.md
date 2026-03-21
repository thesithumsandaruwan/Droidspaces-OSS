# Cool Things You Can Do with Droidspaces

### Quick Navigation

- [1. Setting Up a Secure "Mobile Server" (Tailscale + UFW + Fail2Ban)](#1-setting-up-a-secure-mobile-server-tailscale--ufw--fail2ban)
    - [Prerequisites](#prerequisites)
    - [Step 1: Install Networking Tools & Compatibility Layer](#step-1-install-networking-tools--compatibility-layer)
    - [Step 2: Personal User Setup & SSH Hardening](#step-2-personal-user-setup--ssh-hardening)
    - [Step 3: Set Up Tailscale](#step-3-set-up-tailscale)
    - [Step 4: Secure the Container with UFW (Firewall)](#step-4-secure-the-container-with-ufw-firewall)
    - [Step 5: Add Brute-Force Protection with Fail2Ban](#step-5-add-brute-force-protection-with-fail2ban)

---

You can turn your Android device into a secure, accessible-from-anywhere Linux server by combining Droidspaces with Tailscale and standard Linux security tools.

### Prerequisites

- **Kernel Support**: This setup requires several Netfilter and IPSet modules. See [Additional Kernel Configuration for UFW/Fail2ban](./Kernel-Configuration.md#additional-kernel-configuration-for-ufwfail2ban) for the full list of required options.
- **LTS Distribution**: It is highly recommended to use a Long-Term Support (LTS) distribution like **Ubuntu 24.04 LTS** or **Debian 12** for the best stability and package support.
- **Root User**: All steps in this guide must be run as the **root** user inside the container.
- **Package Manager**: The commands below use `apt`, which is only available in Debian and Ubuntu-based distributions.
- **NAT Mode**: **Mandatory.** You must run your container in NAT mode (`--net=nat`). Using host networking while running a firewall like UFW can interfere with the Android host's connectivity, or it won't even work.

---

### Step 1: Install Networking Tools & Compatibility Layer

To handle firewall rules and network debugging, you first need to install the essential networking tools and ensure compatibility with the Android kernel.

1. **Install tools**:
   ```bash
   apt update && apt install -y net-tools iptables
   ```

2. **Switch to Legacy iptables**:
   Modern Ubuntu/Debian versions use the `nftables` backend by default, which often fails in Droidspaces containers on Android kernels. You **must** switch to the legacy `iptables` backend to ensure your firewall works:
   ```bash
   update-alternatives --set iptables /usr/sbin/iptables-legacy
   update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
   ```

---

### Step 2: Personal User Setup & SSH Hardening

To maintain a secure server, creating a dedicated user with `sudo` privileges and disabling direct root access over SSH is best practice.

1. **Reclaim UID 1000**: Linux distributions usually assign UID `1000` to the first non-root user (like `ubuntu`). To use this ID for your personal user, you should first detect and completely remove any existing UID 1000:
   ```bash
   # Identify and delete the default user associated with UID 1000
   DEFAULT_USER=$(getent passwd 1000 | cut -d: -f 1)
   userdel -r "$DEFAULT_USER"
   groupdel "$DEFAULT_USER" 2>/dev/null
   ```

2. **Create your personal user as UID 1000** (Replace `YOUR_USER` with your desired username):
   ```bash
   useradd -m -u 1000 -s /bin/bash YOUR_USER
   usermod -aG sudo YOUR_USER
   passwd YOUR_USER
   ```

2. **Install OpenSSH Server**:
   ```bash
   apt install -y openssh-server
   ```

3. **Disable Root Login**:
   Edit `/etc/ssh/sshd_config` to prevent direct root access:
   ```bash
   sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin no/' /etc/ssh/sshd_config
   sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
   systemctl restart ssh
   ```

---

### Step 3: Set Up Tailscale

Tailscale provides a secure P2P tunnel to your container, allowing you to access it from any device in your Tailnet without opening ports on your public router.

1. **Install Tailscale**:
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   ```

2. **Authenticate**:
   ```bash
   tailscale up
   ```

---

### Step 4: Secure the Container with UFW (Firewall)

Android kernels often have limited IPv6 support. Since Droidspaces NAT mode currently only supports IPv4, we should disable IPv6 in UFW to avoid initialization errors.

1. **Disable IPv6 in UFW**:
   ```bash
   sed -i 's/IPV6=yes/IPV6=no/' /etc/default/ufw
   ```

2. **Set Default Policies**:
   ```bash
   ufw default deny incoming
   ufw default allow outgoing
   ```

3. **Whitelist the Tailscale Interface**:
   Instead of whitelisting specific IP addresses, tell UFW to trust anything coming through your private Tailscale tunnel:
   ```bash
   ufw allow in on tailscale0
   ```

4. **Enable the Firewall**:
   ```bash
   ufw --force enable
   ```

---

### Step 5: Add Brute-Force Protection with Fail2Ban

Fail2Ban monitors your system logs and automatically blocks IP addresses that show malicious behavior.

1. **Install Fail2Ban**:
   ```bash
   apt install -y fail2ban
   ```

2. **Create a Local Configuration**:
   Create a persistent configuration file at `/etc/fail2ban/jail.local` to protect SSH and integrate it with UFW:

   ```ini
   [DEFAULT]
   # Ban for 1 hour after 5 failed attempts within 10 minutes
   bantime  = 1h
   findtime = 10m
   maxretry = 5

   # Use UFW as the banning action
   banaction = ufw
   
   # Whitelist your Tailscale CIDR range and specific IP to prevent lockouts
   ignoreip = 127.0.0.1/8 ::1 100.64.0.0/10 YOUR_TAILSCALE_IP

   [sshd]
   enabled = true
   port    = ssh
   backend = systemd
   ```

3. **Start and Verify**:
   ```bash
   systemctl restart fail2ban
   fail2ban-client status sshd
   ```

---

Your "Mobile Server" is now officially a hardened fortress! Anyone attempting to access it from the open internet will be blocked, while you maintain full access through your private Tailscale network. 🗿✨
