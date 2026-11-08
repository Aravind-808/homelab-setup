# My Homelab: a Server Made From a Laptop

this is my very first attempt at a homelab !! i've always found the idea of setting up my own server pretty interesting, and this felt like the perfect way to get my hands dirty and learn
a lot about how comp networks work under the hood.

---

## Contents

1. [What I wanted](#what-i-wanted)
2. [The hardware](#the-hardware)
3. [Phase 1: SSH, firewall, Wireshark](#phase-1-ssh-firewall-wireshark)
4. [Phase 2: the IP problem (skipped)](#phase-2-the-ip-problem-skipped)
5. [Phase 3: desktop mode vs server mode](#phase-3-desktop-mode-vs-server-mode)
6. [Phase 4: file server with Samba](#phase-4-file-server-with-samba)
7. [Phase 7: hosting a website with nginx](#phase-7-hosting-a-website-with-nginx)
8. [Putting the site on the internet](#putting-the-site-on-the-internet)
9. [After a reboot: the handbook](#after-a-reboot-the-handbook)

---

## What I wanted

- Learn networking by watching real traffic: TCP handshakes, SSH key exchange, SMB, HTTP.
- Keep everything local and self-hosted. No cloud storage or hosted databases.
- Stay light on resources. No Kubernetes, and no Docker until I actually need it.
- Build it one piece at a time and understand each piece before moving on.

## The hardware

| Thing | What I've got |
|---|---|
| Server | Lenovo ThinkPad E15 Gen 4, dual boot (Ubuntu 22.04 and Windows) |
| CPU | Intel Core i5-1235U |
| RAM | 15 GiB |
| GPU | NVIDIA MX550 (about 2 GB VRAM, not useful for running models) |
| Disk | 477 GB NVMe. `/` is 64 GB, `/home` is 28 GB, Windows is about 350 GB |
| Network | Wi-Fi through a range extender. The router (`192.168.0.1`) is a TP-Link from my ISP |
| Client | My Windows laptop on the same Wi-Fi |

At rest the server (along with other stuff) uses about 1.5 GiB of RAM in desktop mode, so there's plenty of room.

## Phase 1: SSH, firewall, Wireshark

SSH comes first because it's the remote control for everything else. Once it works I can do the rest from my Windows laptop.

SSH, as I understand, is a protocol that is used between 2 systems on a network to communicate with each other in a secure way, in a not-so-secure
network.

### 1. Install the SSH server

On the server:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y openssh-server ufw
systemctl status ssh --no-pager | head -5
ss -tlnp | grep :22
```

- `apt update` refreshes the list of available packages, and `apt upgrade` actually installs the newer ones. I do this first because a machine on a network shouldn't be running old buggy software.
- `ss -tlnp` shows listening TCP sockets (`t`=TCP, `l`=listening, `n`=numbers not names, `p`=which process). Seeing `0.0.0.0:22` means sshd is waiting for connections on port 22 on every interface.
- The idea to remember: **an IP address finds the machine, and a port finds the program on it.**
- The `|` is a pipe. It sends the output of the first command into the second one, so `ss -tlnp | grep :22` means "list everything, but only show me the lines with `:22`".

### 2. Test a normal password login first

From Windows:

```powershell
ssh sec@<SERVER_IP>
```

the connection, during the first time, displays the host system's fingerprint and asks if you are sure that is the system you want to connect to. when you type `yes`, this information is saved somewhere so that the next time you connect, your system knows it can trust this host. this is to make sure that the host is actually the host and not someone pretending to be one.

once normal password login works, we can move on to the key-pair method.

### 3. Make a key pair and put the public half on the server

key pairs are amazing. they enable you to login to the host without using a password. it is much more safer than using the password because this way, someone pretending to be you cant connect to the host by guessing your password.

how it works it, your client generates a pair of keys - a private and public key, and sends this public key over to the server. so anytime the client connects to the server, the server can 
verify that it is infact the client using said public key.

client signs in with private key, server checks signature using public key, that is all. you need to set up a passphrase that you will use instead of the password, its use to encrypt the private key file on the client side.

On Windows:

```powershell
ssh-keygen -t ed25519 -C "homelab"
```

Then copy the **public** key over. Heads up, the command is different in cmd vs PowerShell because the variable syntax differs:

```cmd
type %USERPROFILE%\.ssh\id_ed25519.pub | ssh sec@<SERVER_IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh sec@<SERVER_IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

to check if the key was actually used:

```cmd
ssh -v sec@<SERVER_IP> exit 2>&1 | findstr /i Authenticated
```

```powershell
cmd /c "ssh -v sec@<SERVER_IP> exit 2>&1 | findstr /i Authenticated"
```
You want to see `using "publickey"`.

### 4. Turn off password login

only do this after key login works, and **keep one SSH session open** while you do it, in case you break something.

```bash
sudo tee /etc/ssh/sshd_config.d/10-homelab.conf <<'EOF'
PasswordAuthentication no
PermitRootLogin no
EOF
sudo sshd -t && sudo systemctl restart ssh
sudo sshd -T | grep -iE 'passwordauthentication|permitrootlogin'
```

- A file in `sshd_config.d/` overrides the defaults without editing Ubuntu's main config.
- `sshd -t` checks the syntax first, and `&&` means it only restarts if the check passed.
- Test from a second window: `ssh -o PubkeyAuthentication=no sec@<SERVER_IP>` should say `Permission denied (publickey)` and never ask for a password.
- `sudo` still asks for my Ubuntu password. That's a separate check on the machine itself, not remote login.

### 5. Firewall 

we will now configure the firewall of our server by using uncomplicated firewall (ufw). 

generally, when we want to configure the firewall of out linux system, iptables is what comes in handy. but the problem is that the syntax of iptables is notoriously hard to understand.
ufw makes it easier for us to configure stuff, as you will see below.

in this order, send these commands:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.0.0/24 to any port 22 proto tcp
sudo ufw show added       
sudo ufw enable
sudo ufw status verbose
```

| Command | What it does |
|---|---|
| `default deny incoming` | drop anything I haven't allowed. Nothing is enforced yet. |
| `default allow outgoing` | let the laptop reach out (updates, browsing). replies come back through connection tracking. |
| `allow from 192.168.0.0/24 to any port 22 proto tcp` | allow TCP port 22 from my home network only. `/24` means the first 24 bits (`192.168.0`) have to match. |
| `show added` | safety check. never run `enable` until the SSH rule shows up here. |
| `enable` | start enforcing now, and on every boot. |

prove it from windows:

```powershell
Test-NetConnection <SERVER_IP> -Port 22      # True
Test-NetConnection <SERVER_IP> -Port 8080    # False
```

```bash
sudo grep 'DPT=8080' /var/log/ufw.log
```

Two different failures to not mix up:

| Error | Meaning |
|---|---|
| Connection timed out | Packets got dropped (firewall), or the IP is wrong |
| Connection refused | A machine answered with a reset: it's reachable, but nothing is listening on that port |

### 6. Wireshark lab: watching my own SSH connection

1. download and install wireshark and also download npcap. npcap allows you to monitor packets via wireshark.
2. close any open ssh sessions, and go to wireshark to start capturing the wifi (click on the wifi option). starting a new ssh then lets you see the cool 3 way handshake.
3. display filter: `tcp.port == 22 && ip.addr == <SERVER_IP>`. you only wanna see packets pertaining to your specific server ip and the port in that ip.
4. run `ssh sec@<SERVER_IP>`, type `ls`, then `exit`. then go to wireshark and see the packets. press ctrl + home to go to the first packet
5. youll see the first packet with SYN, then the server responding with SYN, ACK, then the 3rd packet with ACK.

---

## Phase 2: the IP problem (skipped)

dhcp addresses are dynamic. they dont stay the same. mine changed from xyz to abc after reconnecting. to make this permanent, i saw somewhere that you have to go to your home router settings and change up some stuff but i didnt wanna mess with the router for now so i skipped it. you can too, but the price to pay is noting down your ip everytime you start the session. 

### Finding the server's IP

```bash
ip -br addr show wlp0s20f3
```

that gives the address to SSH to (`sec@<that IP>`). `homelabctl status` also prints it.

because the IP can change, my firewall rules use the **subnet** (`192.168.0.0/24`) instead of one fixed address, so a new IP doesn't break them.

how you can fix it if you find this process annoying:

- DHCP reservation on the router
- Static IP on the server with `nmcli`
- mDNS names like `<hostname>.local` (Avahi is already running)
- Proper local DNS with dnsmasq (that's Phase 8)

---

## Phase 3: desktop mode vs server mode

BEFORE you read this, i had to generate most of these scripts with the help of claude as im still finding my way around linux, ive understood what these scripts do, and i suggest you do the same for this or any other scripts you might write in the future. Ive explained the code in my own words.

- **desktop mode:** just normal Ubuntu. No homelab services running.
- **server mode:** `homelab.target` is active, services are running, and the laptop won't suspend.
- sshd stays **always on** in both modes. it's already locked down (keys only, home network only) and it costs basically nothing. it also means i can log in whenever.

### The target and the sleep lock

```bash
sudo tee /etc/systemd/system/homelab.target <<'EOF'
[Unit]
Description=Homelab server mode
Wants=homelab-awake.service
EOF

sudo tee /etc/systemd/system/homelab-awake.service <<'EOF'
[Unit]
Description=Keep laptop awake in server mode
PartOf=homelab.target

[Service]
ExecStart=/usr/bin/systemd-inhibit --what=sleep:handle-lid-switch --who=homelab --why="Server mode" --mode=block /usr/bin/sleep infinity
EOF

sudo systemctl daemon-reload
```

- `Wants=` means "when this target starts, start that too". its a soft dependency.
- `PartOf=` means "when the target stops or restarts, stop me too". between the two, one name starts and stops everything.
- `systemd-inhibit ... sleep infinity` grabs a **block** lock on sleep and lid-close, then just sits there holding it. the service *is* the lock, so stopping it releases the lock and I never have to remember to undo anything.

check it with `systemd-inhibit --list --no-pager`. While the target is up there should be a `homelab` row with `sleep:handle-lid-switch` in `block` mode.

i tested it with the lid closed and SSH kept working, so the lock does its job.

### homelabctl

```bash
sudo tee /usr/local/bin/homelabctl <<'EOF'
#!/bin/bash
TARGET=homelab.target
IFACE=wlp0s20f3

status() {
  if systemctl is-active --quiet "$TARGET"; then
    echo "Mode:  SERVER"
  else
    echo "Mode:  DESKTOP"
  fi
  ip=$(ip -4 -br addr show "$IFACE" | awk '{print $3}' | cut -d/ -f1)
  echo "IP:    ${ip:-none}"
  free -h | awk '/Mem/ {print "RAM:   " $3 " used / " $2}'
  echo "---"
  printf "%-16s %s\n" ssh "$(systemctl is-active ssh)"
  printf "%-16s %s\n" homelab-awake "$(systemctl is-active homelab-awake)"
}

case "$1" in
  start)   sudo systemctl start "$TARGET" && echo "Server mode ON"; status ;;
  stop)    sudo systemctl stop "$TARGET" && echo "Back to desktop mode"; status ;;
  restart) sudo systemctl restart "$TARGET" && echo "Server mode restarted"; status ;;
  status)  status ;;
  *)       echo "usage: homelabctl {start|stop|restart|status}"; exit 1 ;;
esac
EOF
sudo chmod +x /usr/local/bin/homelabctl
```

`/usr/local/bin` is already on the PATH, so `homelabctl` works from anywhere. the wi-fi interface name is hard-coded in `IFACE`, so change it if you switch to ethernet.

```bash
homelabctl status
homelabctl start
homelabctl stop
```

(later I added `public` and `private` commands for the tunnel, see [One command for the public link](#one-command-for-the-public-link). that version replaces this script.)

things to know:

- `start` and `stop` use `sudo`, so running them non-interactively over SSH needs a password prompt. I could add a sudoers rule for just this script later.
- Keep the laptop plugged in and on a hard surface with some airflow while it's in server mode.

### Adding any new service to server mode

Every service gets hooked up the same way (you'll see it for Samba and nginx below):

1. Turn it off at boot: `sudo systemctl disable --now <service>`
2. A drop-in on `homelab.target` with `Wants=<service>`
3. A drop-in on the service with `PartOf=homelab.target`

---

## Phase 4: file server with Samba

**SMB** is the file sharing protocol Windows speaks natively, and it runs over TCP port 445. **Samba** is the Linux server for it. I picked SMB over NFS because my client is Windows.

### 1. Install it and hook it to server mode

```bash
sudo apt install -y samba
sudo systemctl disable --now smbd nmbd

sudo mkdir -p /etc/systemd/system/homelab.target.d /etc/systemd/system/smbd.service.d

sudo tee /etc/systemd/system/homelab.target.d/samba.conf <<'EOF'
[Unit]
Wants=smbd.service
EOF

sudo tee /etc/systemd/system/smbd.service.d/homelab.conf <<'EOF'
[Unit]
PartOf=homelab.target
EOF

sudo systemctl daemon-reload
```

`smbd` now starts and stops with server mode. `nmbd` (old NetBIOS name stuff) stays off.

### 2. The share account and folder

```bash
sudo useradd -M -s /usr/sbin/nologin nasuser
sudo smbpasswd -a nasuser
sudo mkdir -p /srv/nas
sudo chown nasuser:nasuser /srv/nas
sudo chmod 770 /srv/nas
```

- `nasuser` has no home folder (`-M`) and can't log in (`nologin`). it only exists to own files. If someone steals the share password they get one folder, not the whole machine. That's the principle of **least privilege**. i think of it as a maid who only has the key to one room.
- Samba keeps its own password list, so the `smbpasswd` password is separate from any Linux password.
- `770` means owner and group get full access, and everyone else gets nothing.

### 3. Define the share

```bash
sudo tee -a /etc/samba/smb.conf <<'EOF'

[nas]
   path = /srv/nas
   valid users = nasuser
   read only = no
   create mask = 0660
   directory mask = 0770
EOF
testparm -s
```

use `tee -a` (append). without `-a` it would wipe the whole Samba config. `testparm` should say `Loaded services file OK.`

### 4. Open port 445 for my home network only

```bash
sudo ufw allow from 192.168.0.0/24 to any port 445 proto tcp
sudo ufw status
```

### 5. Connect from Windows

```powershell
Test-NetConnection <SERVER_IP> -Port 445
net use Z: \\<SERVER_IP>\nas /user:nasuser
```

| Part | Meaning |
|---|---|
| `Z:` | the drive letter to use |
| `\\<SERVER_IP>` | how Windows writes "a machine on the network" |
| `\nas` | the share name from `smb.conf` |
| `/user:nasuser` | log in as the Samba account, with the Samba password |

```powershell
net use                    # list connections
net use Z: /delete /y      # disconnect
```

### How the permissions work

a file I copy over shows up on the server like this:

```
-rw-rw---- 1 nasuser nasuser 10 Nov  8 12:19 test.txt
```

| Part | Meaning |
|---|---|
| `-` | regular file (`d` would be a folder) |
| `rw-` | owner can read and write |
| `rw-` | group can read and write |
| `---` | everyone else gets nothing |

Read = 4, write = 2, execute = 1, so `rw-rw----` is `660`, which matches `create mask`.

### Storage warning

`/srv/nas` is on the 64 GB root partition (about 44 GB free when I checked), so the share and the OS fight over the same space. If root fills up, Ubuntu starts acting weird. Long term I want to move it to its own partition or an external ext4 drive mounted at `/srv/nas`, and Samba and Windows won't notice the change.

---

## Phase 7: hosting a website with nginx

Skipped ahead to this one because I wanted to host my portfolio. nginx is a web server: it listens on **TCP port 80** and answers requests like `GET /index.html`. It idles at a few MB of RAM.

### 1. Install nginx and hook it to server mode

```bash
sudo apt install -y nginx
sudo systemctl disable --now nginx

sudo tee /etc/systemd/system/homelab.target.d/nginx.conf <<'EOF'
[Unit]
Wants=nginx.service
EOF

sudo mkdir -p /etc/systemd/system/nginx.service.d
sudo tee /etc/systemd/system/nginx.service.d/homelab.conf <<'EOF'
[Unit]
PartOf=homelab.target
EOF

sudo systemctl daemon-reload
```

ubuntu starts nginx the second it installs, so the `disable --now` turns that off. Then it's the same two drop-ins as Samba.

### 2. Make the folder for the site

```bash
sudo mkdir -p /var/www/portfolio
sudo chown $USER:$USER /var/www/portfolio
```

i own this folder, so I can edit the site without `sudo`. nginx runs as `www-data` and only needs to *read* the files.

### 3. Copy my site over from Windows with scp

My HTML file lives on my Windows laptop, and I run commands on the server over SSH. So I need to copy it across. `scp` sends files over SSH, so it uses the same key and port 22 and needs nothing new in the firewall.

**Run this in a normal local PowerShell window on Windows, not inside the SSH session:**

```powershell
scp "C:\path\to\my_portfolio.html" sec@<SERVER_IP>:/var/www/portfolio/index.html
```

- The first part is the file on Windows. The second is `user@server:/path`, and the colon splits the machine from the path.
- Naming it `index.html` matters because that's what nginx serves by default.
- It failed the first time with `No such file or directory`, because I hadn't made `/var/www/portfolio` yet. `scp` can create a file but not a missing folder.
- For a site with more files: `scp -r C:\path\to\site\* sec@<SERVER_IP>:/var/www/portfolio/`. The `\*` copies what's *inside* the folder, so it doesn't end up nested one level too deep.

Then on the server:

```bash
chmod a+r /var/www/portfolio/index.html
ls -l /var/www/portfolio
```

**About the image:** my page has a photo in it, but I didn't need to upload a separate image file. The `<img>` tag uses a **data URI** (`src="data:image/jpeg;base64,..."`), so the photo's bytes are stored right inside the HTML as text. The whole page is one self-contained 29 KB file, with no separate CSS, JS or image requests. If a page links to files like `/style.css` or `images/photo.png`, those have to be copied too, or you get an unstyled page and broken images.

### 4. Tell nginx about the site

```bash
sudo tee /etc/nginx/sites-available/portfolio <<'EOF'
server {
    listen 80;
    server_name _;
    root /var/www/portfolio;
    index index.html;
}
EOF
sudo ln -s /etc/nginx/sites-available/portfolio /etc/nginx/sites-enabled/portfolio
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
```

- `listen 80` means accept connections on port 80.
- `server_name _` is a catch-all, so it answers whatever name or IP the request used.
- `root` is the folder it serves files from.
- `sites-available` holds configs, and a symlink in `sites-enabled` turns one on. Removing the symlink turns it off without deleting anything.
- I removed the `default` site so Ubuntu's welcome page doesn't fight with mine.
- `nginx -t` should say `syntax is ok` and `test is successful`.

### 5. Open port 80 and start it

```bash
sudo ufw allow from 192.168.0.0/24 to any port 80 proto tcp
homelabctl start
systemctl is-active nginx
ss -tlnp | grep :80
```

Then open `http://<SERVER_IP>` in a browser, or:

```powershell
Test-NetConnection <SERVER_IP> -Port 80
```

## Putting the site on the internet

my server has a private address (`192.168.0.x`), so outsiders can't reach it directly. so i decided i would use a cloudflare tunnel for now, as it doesnt need any router mods
or cgnat requirements.

### Quick tunnel (free, no account)

**1. Install cloudflared** from Cloudflare's apt repo:

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update && sudo apt install -y cloudflared
cloudflared --version
```

**2. Make it a systemd service** so it doesn't die when a terminal closes:

```bash
sudo tee /etc/systemd/system/homelab-tunnel.service <<'EOF'
[Unit]
Description=Cloudflare quick tunnel to local nginx
PartOf=homelab.target
After=nginx.service network-online.target
Wants=network-online.target

[Service]
DynamicUser=yes
ExecStart=/usr/bin/cloudflared tunnel --no-autoupdate --url http://localhost:80
Restart=on-failure
EOF
sudo systemctl daemon-reload
```

- `PartOf=homelab.target` means `homelabctl stop` also kills the tunnel.
- there's **no** `Wants=` from the target on purpose, so server mode does *not* start the tunnel by itself. The site is only public when I choose to start it.
- `DynamicUser=yes` runs it as a throwaway unprivileged account.
- `Restart=on-failure` reconnects it if it drops.

**3. Start it and grab the link:**

```bash
homelabctl start
curl -I http://localhost                       # nginx should say 200 OK first
sudo systemctl start homelab-tunnel
sleep 10
sudo journalctl -u homelab-tunnel --no-pager | grep -o 'https://[a-z0-9-]*\.trycloudflare\.com' | tail -1
```

`grep -o` prints only the part that matches the pattern, so I get a clean URL.

**4. Test it from outside:** turn off Wi-Fi on my phone (so it uses mobile data) and open the link. If the portfolio loads, it works from anywhere.

**Turn it off:** `homelabctl private`, or `sudo systemctl stop homelab-tunnel` (the link dies instantly, but the site still works on my home network).

### Things to know about the quick tunnel

- The URL is **random and changes** every time the tunnel restarts (including after a reboot). I can't pick a custom name on `trycloudflare.com`.
- Cloudflare says quick tunnels are for dev and testing, with a limit of 200 concurrent requests. That's plenty for friends.
- Anyone with the link can open the page, so I treat the link as disposable.

## After a reboot: the handbook

what I do when I turn the laptop off and back on. Short version: **boot, find the IP, SSH in, `homelabctl start`, and `homelabctl public` if I want the site public.**

### Start-up routine

**0. Boot into Ubuntu.** my server lap is a dual boot one so i boot into ubuntu.

**1. Find the server's IP.** one of these from the server laptop:

- On the laptop screen: `homelabctl status`
- The connected-devices list on the router at `192.168.0.1`
- `ssh sec@<hostname>.local` (Avahi is running)

**2. SSH in from Windows:**

```powershell
ssh sec@<SERVER_IP>
```

**3. Turn on server mode:**

```bash
homelabctl start
homalabctl status
```

**4. Check site locally:**

```bash
curl -I http://localhost
```

**5. Start tunnel, get new url:**
```bash
sudo systemctl start homelab-tunnel
sleep 10
sudo journalctl -u homelab-tunnel --no-pager | grep -o 'https://[a-z0-9-]*\.trycloudflare\.com' | tail -1
```
### Before I shut down or reboot

Doing it in this order avoids the frozen File Explorer problem:

1. Windows: `net use Z: /delete /y`
2. Server: `homelabctl private` (stops the tunnel)
3. Server: `homelabctl stop`
4. Then shut down or reboot.

### Stuff I've learned so far

TCP handshake and teardown, ports and listening sockets, public-key auth, host keys and `known_hosts`, Diffie-Hellman key agreement, stateful firewalls and default-deny, dropped packets vs resets, TCP retransmission and backoff, Ethernet/IP/TCP headers, DHCP leases, subnet notation, systemd units, targets and drop-ins, sleep inhibitor locks, SMB and Samba, Linux permissions and groups, least privilege, HTTP requests and responses, reverse tunnels and why they work behind CGNAT.
