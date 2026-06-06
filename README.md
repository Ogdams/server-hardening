# Hardening a Linux VPS with Fail2ban, CrowdSec, and SSH Key-Only Auth

If you just spun up a fresh VPS (any Debian/Ubuntu box really), the default install is wide open. SSH is exposed on port 22 with password auth. Bots find it in minutes. This guide walks through what I did to lock mine down — three layers that work together without stepping on each other.

## What you end up with

| Tool | Job |
|---|---|
| Fail2ban | Fast local bans on repeated failed SSH attempts |
| CrowdSec | Behavioral detection + shared blocklist from the community |
| iptables bouncer | The thing that actually drops the IP at the firewall level |
| SSH key auth | No passwords allowed, root can only log in with a key |

None of these conflict. Fail2ban catches the dumb brute force locally within seconds. CrowdSec watches for weirder patterns and also pulls in reputation data from other servers. The bouncer ties CrowdSec's decisions into iptables. And locking SSH to keys only means even if someone gets past both, they still need a key.

---

## 1. Update the server

SSH in and get everything current before touching any configs.

```bash
ssh root@your-server-ip
```

```bash
apt update && apt upgrade -y
```

Reboot if the kernel got upgraded. Not strictly required but I do it out of habit.

```bash
reboot
```

---

## 2. Install and configure Fail2ban

Fail2ban has been around forever and it's still the easiest first line of defense. It watches logs, counts failures, and bans the IP at the firewall level once a threshold is hit.

```bash
apt install fail2ban -y
systemctl enable fail2ban
systemctl start fail2ban
```

The default config file is `/etc/fail2ban/jail.conf` but you shouldn't edit that directly — package updates will overwrite it. Copy it to a `.local` file instead.

```bash
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Open the local copy and find the `[sshd]` section:

```bash
nano /etc/fail2ban/jail.local
```

Look for this block and set it accordingly:

```
[sshd]
enabled = true
port = ssh
maxretry = 5
bantime = 3600
findtime = 600
```

What these mean:
- `maxretry = 5` — ban after 5 failed attempts
- `findtime = 600` — those 5 attempts must happen within 10 minutes
- `bantime = 3600` — ban lasts 1 hour

Restart and check it's running:

```bash
systemctl restart fail2ban
fail2ban-client status sshd
```

You should see something like `Status: OK` with a counter for currently banned IPs.

---

## 3. Install CrowdSec

CrowdSec is basically Fail2ban's younger, smarter sibling. Same idea — detect attacks from logs — but it uses a behavior-based approach and shares ban decisions across a community network. If someone attacks 10 different servers, CrowdSec can propagate that reputation.

```bash
curl -s https://install.crowdsec.net | bash
apt install crowdsec -y
systemctl enable crowdsec
systemctl start crowdsec
```

By itself CrowdSec just parses logs and runs scenarios against them. It doesn't block anything yet — that's what the bouncer is for later.

---

## 4. Add SSH collection for CrowdSec

CrowdSec needs to know what logs to watch. The SSH collection tells it what to look for in auth logs.

```bash
cscli collections install crowdsecurity/sshd
systemctl restart crowdsec
```

You can check that it's picking up data:

```bash
cscli metrics
```

This shows you how many events have been parsed, what scenarios are active, and whether anything has triggered an alert.

---

## 5. Install the firewall bouncer

CrowdSec detects threats and writes them to its local database, but it doesn't actually block IPs by itself. A bouncer reads those decisions and applies them somewhere — iptables, nginx, cloudflare, whatever. The firewall bouncer does iptables.

```bash
apt install crowdsec-firewall-bouncer-iptables -y
systemctl enable crowdsec-firewall-bouncer
systemctl start crowdsec-firewall-bouncer
```

That's it. Now when CrowdSec makes a decision about an IP, the bouncer adds an iptables rule to drop traffic from it.

---

## 6. Verify both are working

```bash
cscli decisions list
```

Should be empty if no one's hit you yet, or show IPs if bans are active.

```bash
fail2ban-client status
```

Lists all jails. `sshd` should show as active.

---

## 7. Disable SSH password authentication

This is the one that actually matters most. Fail2ban and CrowdSec react to attacks. Disabling password auth prevents the attack entirely — if passwords don't work, bots can try all they want and get nowhere.

**Before doing this, make sure your SSH key is set up and you can log in with it.** If you lock yourself out, you're reinstalling the server.

Add your public key if you haven't already:

```bash
mkdir -p ~/.ssh
echo "your-public-key-here" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Test that key login works in a separate terminal before you disable passwords. I've made that mistake and it's annoying.

Then edit the SSH config:

```bash
nano /etc/ssh/sshd_config
```

Set these two lines:

```
PasswordAuthentication no
PermitRootLogin prohibit-password
```

`PermitRootLogin prohibit-password` means root can still log in, but only with a key. No password, no matter how strong.

Restart SSH:

```bash
systemctl restart ssh
```

Try logging in from another terminal. If it works without prompting for a password, you're good. If it asks for a password, something's wrong — check your key setup before closing the current session.

---

## How the layers fit together

There's no real overlap between these tools. They work at different stages of an attack:

1. SSH key auth stops anyone who doesn't have a valid key — period.
2. Fail2ban watches `/var/log/auth.log` and bans IPs that hit the retry limit within a time window. Local only, simple, fast.
3. CrowdSec parses the same logs but applies more nuanced scenarios. It also checks IPs against the community blocklist. Detections are written to its local database.
4. The bouncer polls CrowdSec and adds iptables rules for any IP that has an active decision against it.

If someone tries to brute force SSH:
- They can't get in because password auth is off.
- Fail2ban sees the failures and bans them at the firewall after 5 tries.
- CrowdSec also sees the failures and bans them at the firewall after its own threshold.
- If that IP has attacked other CrowdSec instances, it gets blocked immediately by reputation.

Not bad for a few minutes of setup.

---

## Useful commands

```bash
# Fail2ban
fail2ban-client status sshd         # Check SSH jail stats
fail2ban-client set sshd unbanip x.x.x.x   # Manually unban

# CrowdSec
cscli decisions list                # Show active bans
cscli metrics                       # Parsing and alert stats
cscli alerts list                   # Show triggered alerts
cscli decisions delete -i x.x.x.x   # Remove a ban

# Firewall
iptables -L -n                      # See current iptables rules
```

---

## Notes

- These instructions target Debian/Ubuntu. CrowdSec has packages for other distros but the install command is different.
- Some VPS providers ship their images with basic firewall rules already in place. These tools add to them, not replace them.
- CrowdSec can feel heavy compared to Fail2ban — it uses more memory and has a lot more moving parts. That's the tradeoff for the community intelligence features.
- If you only want one, pick Fail2ban. It's simpler, well tested, and sufficient for most use cases. CrowdSec adds value mainly if you want the network-level visibility or you manage multiple servers.
