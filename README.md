# SSH Remote Server Setup

## Project Overview

This project demonstrates how to provision a remote **Linux server on Amazon Web Services (AWS), configure Secure Shell (SSH)** access, and authenticate using two independent **SSH key pairs**.

The project focuses on the fundamentals of remote server administration, public-key authentication, cloud networking, and basic SSH security hardening.

The primary objective is to establish secure remote access to an Ubuntu Linux server hosted on an AWS EC2 (Elastic Compute Cloud) instance. Both SSH key pairs must independently authenticate to the same remote user account.

As an additional convenience, the project explores SSH client configuration through ~/.ssh/config, which allows connections using a memorable alias instead of repeatedly specifying the server's IP address, username, and private key path.

The stretch goal introduces Fail2ban, a security tool that monitors authentication failures and can temporarily block IP addresses exhibiting suspicious activity.

---

## Table of Contents

- [Project Brief](#project-brief)
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step 1: Prepare Your Local Machine](#step-1-prepare-your-local-machine)
- [Step 2: Provision the Server on AWS](#step-2-provision-the-server-on-aws)
- [Step 3: Connect to the Server](#step-3-connect-to-the-server)
- [Step 4: Add the Second Key Pair](#step-4-add-the-second-key-pair)
- [Step 5: Stretch Goal, Install fail2ban](#step-5-stretch-goal-install-fail2ban)
- [Step 6: Configure the SSH Jail](#step-6-configure-the-ssh-jail)
- [Verification Checklist](#verification-checklist)
- [Another Approach: SSH Config Aliases](#another-approach-ssh-config-aliases)
- [Security Notes](#security-notes)
- [Author](#author)

---

## Project Brief/ Deliverables

**Goal:** set up a basic remote Linux server and configure it to accept SSH connections.

**Requirements from the brief:**

1. Register and set up a remote Linux server on any provider (DigitalOcean, AWS, etc.).
2. Create **two** new SSH key pairs and add them to the server.
3. Connect to the server with **both** keys using:

   ```bash
   ssh -i <path-to-private-key> user@server-ip
   ```

4. Look into `~/.ssh/config` so the server can be reached with a short alias: `ssh <alias>`.
5. **Stretch goal:** install and configure `fail2ban` to prevent brute-force attacks.

The only outcome that matters is being able to SSH into the server with both keys. Everything else builds toward that or hardens it.

> **Submission note:** never push a private key to a public repository. Only this README (the steps) is submitted.

---

## Overview

- #!Shebang Everyone, So we are required to launch a Linux Virtual Server 
- Generate SSH key pairs (a public and a private key) with `ssh-keygen`.
- Configure the instance to authenticate me using those keys instead of a password.
- Create local shortcuts: a `~/.ssh/config` file that maps server details to a short alias, so `ssh <alias>` just works.
- Install and configure **fail2ban** on the server so it automatically detects and blocks brute-force login attempts.

**My approach:** Create an AWS EC2 Instance, Must use an Ubuntu Image,  Generated keys locally with `ssh-keygen`, first key imported through the AWS console, second key added directly on the server.

---

## Prerequisites

- An AWS account (the Free Tier is enough).
- A Linux machine, or Ubuntu running in Oracle VirtualBox if you don't have one.
- A terminal.

---

## Step 1: Prepare Your Local Machine

### 1.1 Install and verify OpenSSH

**SSH (Secure Shell)** is a protocol for logging into and controlling a remote machine over an encrypted connection. It has two sides: the **client** (on your machine, runs the `ssh` command) and the **server** (on the remote machine, listens for connections).

```bash
sudo apt update
sudo apt install openssh-server -y
dpkg -s openssh-server
```

- `apt update` refreshes the package index so you install the latest versions.
- `apt install openssh-server -y` installs the SSH server package; `-y` auto-confirms the prompt.
- `dpkg -s openssh-server` shows the package status. `Status: install ok installed` means it worked.

Also confirm the client is available, since that is what runs the `ssh` command:

```bash
ssh -V
```

### 1.2 Generate the first SSH key pair

SSH key authentication uses an asymmetric **key pair**: a **private key** that never leaves your machine and a **public key** that you place on the server. The server uses the public key to challenge you, and only the matching private key can answer, so no password ever crosses the network.

```bash
ssh-keygen -t rsa -b 4096
```

- `ssh-keygen` generates the key pair.
- `-t rsa` selects the RSA algorithm.
- `-b 4096` sets the key length to 4096 bits.

Accept the default location and set a passphrase when asked. You now have two files in `~/.ssh/`:

| File | What it is | Share it? |
|---|---|---|
| `id_rsa` | Private key | **Never** |
| `id_rsa.pub` | Public key | Yes, this goes on the server |

### 1.3 Save the public key to a file

```bash
cat ~/.ssh/id_rsa.pub > key-pair.txt
```

`cat` prints the public key and `>` redirects it into `key-pair.txt`, which we will import into AWS next. Double-check that you are exporting the `.pub` file and not the private key.

---

## Step 2: Provision the Server on AWS

### 2.1 Import the public key into AWS

1. Sign in to the **AWS Management Console** (or create a Free Tier account).
2. Search for **Key Pairs** and open it in a new tab.
3. Click **Import key pair** and fill in:

   ```text
   name:            devops-key-pair
   key pair file:   key-pair.txt   (browse and upload, or paste the contents)
   tags:            (optional, left blank)
   ```

4. Click **Import key pair**.

AWS stores only the public key. When the instance launches, AWS places it in the server's `~/.ssh/authorized_keys` file, the list of public keys allowed to log in.

### 2.2 Launch the EC2 instance

Back in the console, search for **EC2** and click **Launch instance**. Since we want a Linux server, the machine image must be Linux. Use these settings:

```text
name:            devops-vm
image:           Ubuntu Server 26.04 LTS
instance type:   t3.micro
key pair:        devops-key-pair
security group:  create new (or select existing)
                 -> Allow SSH (port 22) from 0.0.0.0/0 (anywhere)
storage:         defaults, or adjust to your needs
```

- **Security group:** a virtual firewall around the instance. SSH listens on **port 22**, so we need an inbound rule allowing it, otherwise connections are silently dropped.
- **Source `0.0.0.0/0`:** allows any IP to *attempt* a connection. I chose this for convenience, but restricting the source to **your own IP** is the better practice (see [Security Notes](#security-notes)).

Click **Launch instance** and wait until its state shows **Running**.

---

## Step 3: Connect to the Server

Copy the instance's **Public IPv4 address** from the EC2 console, then lock down your private key's permissions. SSH refuses to use a private key that other users can read:

```bash
chmod 400 ~/.ssh/id_rsa
```

Now connect:

```bash
ssh -i ~/.ssh/id_rsa ubuntu@<server-ip>
```

- `ssh` starts the connection.
- `-i ~/.ssh/id_rsa` is the **identity file** flag. It tells SSH exactly which private key to authenticate with. The usual `ssh user@server-ip` works when you have a single default key (SSH tries `~/.ssh/id_rsa` automatically), but once you have **multiple keys**, as this project requires, `-i` lets you pick which one to use explicitly.
- `ubuntu@<server-ip>` is `user@host`. `ubuntu` is the default login user on Ubuntu AMIs (other images use `ec2-user`, `admin`, and so on).

The first time, SSH asks you to trust the server's fingerprint. Type `yes` to continue. If you get a shell prompt on the server, the core of the project is working.

---

## Step 4: Add the Second Key Pair

The brief requires **two** key pairs. AWS attached only the first one at launch, so we add the second ourselves.

**On your local machine**, generate a second key with a different filename so it does not overwrite the first:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/devops_key_2 -C "devops-second-key"
```

- `-f` sets the output file path, which is what keeps the pair separate from `id_rsa`.
- `-C` adds a comment label to the public key so you can identify it later.

Append the new public key to the server's `authorized_keys`, authenticating with the first key:

```bash
cat ~/.ssh/devops_key_2.pub | ssh -i ~/.ssh/id_rsa ubuntu@<server-ip> "cat >> ~/.ssh/authorized_keys"
```

This pipes the public key over the encrypted connection and **appends** (`>>`) it. A single `>` would overwrite the file and lock out your first key.

Test **both** keys:

```bash
ssh -i ~/.ssh/id_rsa ubuntu@<server-ip>
ssh -i ~/.ssh/devops_key_2 ubuntu@<server-ip>
```

Both should log you in. **The core requirement is complete.**

---

## Step 5: Stretch Goal, Install fail2ban

Any server with a public IP and an open port 22 is constantly probed by bots guessing credentials. **fail2ban** reads authentication logs, detects repeated failed logins, and temporarily bans the offending IP at the firewall level.

On the server:

```bash
sudo apt update
sudo apt install fail2ban -y
```

Start it now and enable it on every boot:

```bash
sudo systemctl start fail2ban
sudo systemctl enable fail2ban
```

- `start` runs the service immediately.
- `enable` makes it start automatically after a reboot.

---

## Step 6: Configure the SSH Jail

A **jail** is a fail2ban rule set that ties a service (such as SSH) to a ban policy.

### 6.1 Create a local override file

```bash
cd /etc/fail2ban
sudo cp jail.conf jail.local
```

Never edit `jail.conf` directly. Package updates can overwrite it. fail2ban reads `jail.local` afterwards and its values take precedence, so your changes are safe.

### 6.2 Enable the sshd jail

```bash
sudo vi /etc/fail2ban/jail.local
```

Find the `[sshd]` section and make sure it is enabled:

```ini
[sshd]
enabled = true
```

### 6.3 Whitelist your own IP

Find `ignoreip` and add your public IP, space-separated, so you never ban yourself:

```ini
ignoreip = 127.0.0.1/8 YOUR.OWN.PUBLIC.IP
```

If your ISP gives you a dynamic IP, this entry can go stale, so recheck it if you get locked out.

### 6.4 Tune the ban parameters

The defaults work (10 minutes), but you can adjust them:

| Parameter | Meaning |
|---|---|
| `bantime` | How long an offending IP stays banned |
| `findtime` | The sliding time window in which failed attempts are counted |
| `maxretry` | Failures allowed within `findtime` before a ban triggers |

For example, `maxretry = 5` with `findtime = 10m` bans an IP that fails 5 logins within 10 minutes for the duration of `bantime`.

### 6.5 Apply and verify

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

The status output should show the `sshd` jail active, with counters for currently failed and currently banned IPs.

---

## Verification Checklist

- [ ] Can log in with key 1: `ssh -i ~/.ssh/id_rsa ubuntu@<server-ip>`
- [ ] Can log in with key 2: `ssh -i ~/.ssh/devops_key_2 ubuntu@<server-ip>`
- [ ] `sudo systemctl status fail2ban` shows **active (running)**
- [ ] `sudo fail2ban-client status sshd` shows the jail is active

---

## Another Approach: SSH Config Aliases

I connected with the full `ssh -i ... user@ip` command throughout this project. The brief suggests a cleaner and more efficient method for day-to-day use: the **SSH client config file**, `~/.ssh/config`. It stores connection details under a short name (an **alias**), so you stop retyping the key path, user and IP.

Create or edit `~/.ssh/config` on your **local machine**:

```text
Host devops-key1
    HostName <server-ip>
    User ubuntu
    IdentityFile ~/.ssh/id_rsa

Host devops-key2
    HostName <server-ip>
    User ubuntu
    IdentityFile ~/.ssh/devops_key_2
```

Set safe permissions:

```bash
chmod 600 ~/.ssh/config
```

Now connecting is just:

```bash
ssh devops-key1
ssh devops-key2
```

`Host` is the alias, `HostName` the server address, `User` the login user, and `IdentityFile` the private key (the config equivalent of `-i`). If the server is stopped and restarted, its public IP may change, and then you only update `HostName` in one place.

---

## Security Notes

- **Never commit private keys** (`id_rsa`, `devops_key_2`) to Git. Only `.pub` files are safe to share.
- **Restrict the security group.** Replace `0.0.0.0/0` with your own IP (`x.x.x.x/32`) when you can. fail2ban is a second layer of defense, not a substitute.
- **Use passphrases** on private keys so a stolen key file is not immediately usable.
- **Keep both keys backed up securely.** If you lose every private key, you are locked out of the instance.
- **Terminate or stop the instance** when you finish to avoid unexpected AWS charges.

## AUTHOR

**OPSBYNYIKO**