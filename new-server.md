# New server Setup

## SSH

### Before initialising the server

#### On own device (not server)
Ideally always set a passphrase. The exceptions can be for automated services, but these should run on accounts with reduced permissions only.
```bash
ssh-keygen -t ed25519 -a 128 -C "<name>@<device>" -f ~/.ssh/<key file name>
```
> - `-C "<text>"` allows a comment, here it is suggested to list what device the key belongs to
> - `-a <integer>` sets the number of rounds to take with the password in the key-derivation function, higher means slightly slower unlocking but more security. The default is 16
> - `-f <path>` sets where to save the key pair 

#### Add the server to `~/.ssh/config`
```
Host <desired name>
  HostName <ip-address>
  User <account on server>
  IdentityFile ~/.ssh/<private key>
```

If this is a completely new setup, flashing a raspberry pi, setting up a web server, proxmox VM, etc. it should always be possible to set a main ssh key during initial setup. This should be the public key from the key generation step above. `~/.ssh/<key name>.pub`

### After initialising the server

#### First SSH

> #### Sudo Access
> Note that sometimes sudo may not already be installed. If this is the case, after ssh connection is established run the following:
> ```bash
> su -    # Switch to root
> apt install sudo
> usermod -aG sudo <user>
> ```
> After this logout (Ctrl+d) and log in again.

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
AllowUsers <username>
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
```

> An explanation of the settings:
> - `PermitRootLogin no`: It is impossible to login directly as root
> - `PasswordAuthentication no`: You can only log in with an ssh key over ssh, not with a password
> - `KbdInteractiveAuthentication no`: Further preventing passwords via ssh in some edge cases
> - `PubkeyAuthentication yes`: Explicitly allow ssh key login, in case accidentally disabled elsewhere
> - `AllowUsers <user>`: Which users may login via SSH. Names should be separated by spaces
> - `MaxAuthTries 3`: The maximum tries allowed per connection, after which the server will disconnect
> - `LoginGraceTime 30`: If a connection does not fully log in within 30 seconds it is disconnected (down from a default of 2 minutes) _Note_: If you use a long password, or paste it from a manager and feel that this process could take 30 seconds, use a longer grace period.
> - `X11Forwarding no`: Graphical windows cannot be sent over SSH. Remove this line if that is desired

Then apply the changes:
```bash
sudo sshd -t && sudo systemctl reload ssh
sudo sshd -T | grep -E 'permitrootlogin|passwordauthentication|kbdinteractive|allowusers'
```

**Important**

Before closing the current session, try to log in with ssh via a new terminal, to ensure you do not get locked out

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
sudo rkhunter --check --sk    # May contain some harmless warnings
sudo debsums -sc             # Should return nothing
```
