# New server Setup

## SSH

### Before initialising the server

#### On own device (not server)
Ideally always set a password. The exceptions can be for automated services, but these should run on accounts with reduced permissions only.
```bash
ssh-keygen -t ed25519 -C "<name>@<device>"
```

#### Add the server to `~/.ssh/config`
```bash
Host <desired name>
  HostName <ip-address>
  User <account on server>
  IdentityFile ~/.ssh/<private key>
```

If this is a completely new setup, flashing a raspberry pi, setting up a web server, proxmox VM, etc. it should always be possible to set a main ssh key during initial setup. This should be the public key from the key generation step above. `~/.ssh/<key name>.pub`

### Afer initialising the server

#### First SSH

```bash
ssh <desired name from config>

# Ensure that it is up to date
sudo apt update && sudo apt full-upgrade
sudo hostnamectl set-hostname <desired server name>
sudo timedatectl set-timezone <desired timezone>   # E.g. Europe/Zurich

# Set account password for VNC if accessing via host
sudo passwd <user>
```

#### SSH Hardening

Create a new file to contain the new ssh rules replace `vim` with `nano` if preferred, or `vi` if vim is not installed. 
```bash
sudo vim /etc/ssh/sshd_config.d/10-hardening.conf
```

Add the following:
```
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowUsers debian
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
```

Then apply the changes:
```bash
sudo sshd -t && sudo systemctl reload ssh
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|kbdinteractive|allowusers'
```

#### Automatic Updates

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

#### Reduce brute force impact

```bash
sudo apt install fail2ban
sudo tee /etc/fail2ban/jail.local >/dev/null <<'EOF'
[sshd]
enabled = true
backend = systemd
EOF
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

#### Record a clean baseline 

```bash
sudo apt install rkhunter debsums
sudo rkhunter --propupd
sudo rkhunter --check -sk    # May contain some harmless warnings
sudo debsums -sc             # Should return nothing
```
