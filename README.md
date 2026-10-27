Here's a checklist you can copy. Commands assume Debian; run them on each server.

## Server checklist

### Monthly

**1. Updates**
```bash
sudo apt update && sudo apt list --upgradable
sudo apt upgrade                     # if unattended-upgrades isn't installed
cd ~/apps/APP && docker compose pull && docker compose build --pull && docker compose up -d   # per Docker app
```

**2. What's reachable from the internet.** Only 22, 80 and 443 should be listed:
```bash
sudo ss -tlnp | grep -vE '127\.0\.0\.|\[::1\]|100\.97\.144\.31'
docker ps --format '{{.Names}}  {{.Ports}}' | grep '0.0.0.0'
```
Replace `100.97.144.31` with that server's Tailscale IP. Any `0.0.0.0:PORT` other than nginx's should be changed to `127.0.0.1:PORT:PORT`.

**3. Containers survive reboots.** Every container should show `unless-stopped`:
```bash
docker ps -a --format '{{.Names}}  {{.Status}}'
docker inspect -f '{{.Name}} {{.HostConfig.RestartPolicy.Name}}' $(docker ps -aq)
```

**4. Disk space and certificates**
```bash
df -h /
sudo certbot renew --dry-run         # only on servers with certbot
```

**5. Backups.** Check that the latest dump exists and isn't tiny:
```bash
ls -lh ~/*.sql* | tail -3
```

### Every few months

**6. SSH settings.** You want `no`, `no` and `no`:
```bash
sudo sshd -T | grep -E 'passwordauthentication|permitrootlogin|kbdinteractiveauthentication'
```

**7. Who has access.** Each key should be a device you recognise:
```bash
cat ~/.ssh/authorized_keys
sudo cat /root/.ssh/authorized_keys 2>/dev/null
```

**8. Scheduled jobs and temp folders.** You shouldn't see anything you didn't set up:
```bash
crontab -l; sudo crontab -l; ls /etc/cron.d/
systemctl list-timers --no-pager
ls -la /tmp /var/tmp /dev/shm
```

**9. Rootkit and integrity checks.** rkhunter should give only known warnings, and debsums no output:
```bash
sudo nice -n 19 ionice -c3 rkhunter --check --sk
sudo nice -n 19 ionice -c3 debsums -sc
```
Known harmless rkhunter warnings on cb-server are `/etc/.updated` and `/etc/.resolv.conf.systemd-resolved.bak`.

**10. Logs: did anything suspicious succeed?** These assume the new log format with the hostname first:
```bash
# sensitive paths that returned 200 (sizes 539/558/1288 = catch-all pages, fine)
sudo zcat -f /var/log/nginx/access.log* | awk '$10 == 200' \
  | grep -Ei '\.git|\.env|\.aws|\.bak|\.sql|backup' | awk '{print $1, $11, $8}' | sort | uniq -c | sort -rn | head

# server errors per site (should be near zero)
sudo zcat -f /var/log/nginx/access.log* | awk '$10 ~ /^5/ {print $1}' | sort | uniq -c

# SSH login attempts
sudo journalctl -u ssh --since "-30 days" | grep -c "Failed\|Invalid"
```

### From outside, every few months

**11. External port scan** from another server (cb-server scanning satori, and vice versa). Expected open ports: 22, 80, 443. Everything else should be `filtered` or `closed`:
```bash
sudo nmap -Pn --top-ports 1000 OTHER.SERVER.IP
```

**12. Panels and dashboards**
- Infomaniak firewall still allows only 22, 80, 443 and ICMP.
- Tailscale admin console: remove old devices.
- Uptime Kuma: all monitors green, including the contact form and visitor counts.

### Per server

| Check | cb-server | satori server |
|---|---|---|
| Tailscale IP for item 2 | 100.97.144.31 | its own |
| Backups (item 5) | annotato dump | satori dump |
| certbot (item 4) | yes | only if it has certificates |
| Log checks (item 10) | new format (`$10`) | check whether it has `$host` logging; if not, use `$9` / `$7` / `$1` |
