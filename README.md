# AJ's Docker Home Server (Raspberry Pi)
A lightweight, containerized home server running on a Raspberry Pi with remote access via encrypted VPN tunnel.

---

## Overview
Building a self-hosted home server infrastructure using Docker containers on a Raspberry Pi, accessible both locally and remotely. The goal was to understand containerization, service deployment, and secure remote access without opening ports or exposing the server to the public internet.

---

## Environment
- **Hardware:** Raspberry Pi (physical, wired ethernet)
- **OS:** Raspberry Pi OS Lite (no desktop environment)
- **Host IP:** 10.0.0.2 (static, assigned via NetworkManager)
- **Router:** Rogers Ignite gateway (10.0.0.1)

| Container | Image | Port |
|---|---|---|
| Portainer CE | portainer/portainer-ce:latest | 8000, 9443 |
| Heimdall | lscr.io/linuxserver/heimdall:latest | 8888 |
| Pi-hole | pihole/pihole:latest | host networking |

---

## Setup

### 1. Base OS & SSH
Flash Raspberry Pi OS Lite onto an SD card using Raspberry Pi Imager. During setup, enable SSH and set your username and password. Boot the Pi, find its IP from your router, and SSH in:

```bash
ssh username@<PI-IP>
```

Update the system before doing anything else:

```bash
sudo apt update && sudo apt full-upgrade -y && sudo reboot
```

---

### 2. Static IP
A static IP is required so your services are always reachable at the same address. Since Raspberry Pi OS Lite uses NetworkManager, configure it with nmcli:

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.addresses 10.0.0.2/24
sudo nmcli connection modify "Wired connection 1" ipv4.gateway 10.0.0.1
sudo nmcli connection modify "Wired connection 1" ipv4.dns 10.0.0.1
sudo nmcli connection modify "Wired connection 1" ipv4.method manual
sudo nmcli connection down "Wired connection 1" && sudo nmcli connection up "Wired connection 1"
```

> **NOTE:** This will drop your SSH session briefly. Reconnect on the new static IP (10.0.0.2).

Disable WiFi since the Pi should run wired only:

```bash
sudo rfkill block wifi
echo "dtoverlay=disable-wifi" | sudo tee -a /boot/firmware/config.txt
```

---

### 3. Docker
Do not install apps directly on the Pi. Use Docker to keep everything containerized, isolated, and easy to manage:

```bash
curl -sSL https://get.docker.com | sh
```

Add your user to the Docker group to avoid permission issues:

```bash
sudo usermod -aG docker $USER
```

Exit and SSH back in, then confirm:

```bash
groups
```

You should see `docker` listed among your groups.

---

### 4. Portainer
Portainer is a web GUI for managing all your Docker containers without needing to memorize commands:

```bash
docker volume create pdat

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v pdat:/data \
  portainer/portainer-ce:latest
```

Access Portainer at `https://10.0.0.2:9443`. Click through the certificate warning — this is expected. Create your admin account on first launch and save the password somewhere safe.

---

### 5. Heimdall
Heimdall is a dashboard that gives you a single launchpad for all your services. Run it on port 8888 — port 80 is reserved for Pi-hole:

```bash
docker run -d \
  --name heimdall \
  --restart=always \
  -p 8888:80 \
  -p 4443:443 \
  -v heimdall_data:/config \
  lscr.io/linuxserver/heimdall:latest
```

Access it at `http://10.0.0.2:8888` and add your services as app tiles.

---

### 6. Pi-hole
Pi-hole handles DNS for your network and blocks ads at the DNS level. It requires host networking and the NET_ADMIN capability for DHCP to work properly:

```bash
docker run -d \
  --name pihole \
  --restart=always \
  --network host \
  --cap-add NET_ADMIN \
  -e TZ="America/Edmonton" \
  -e WEBPASSWORD="yourpassword" \
  -v pihole_data:/etc/pihole \
  -v dnsmasq_data:/etc/dnsmasq.d \
  --dns=127.0.0.1 \
  --dns=1.1.1.1 \
  pihole/pihole:latest
```

> **NOTE:** Because Pi-hole uses host networking, it shares the host's ports directly. Port 80 must be free on the host — do not map Heimdall or any other container to port 80.

Enable DHCP inside the container:

```bash
docker exec -it pihole bash
sed -i 's/active = false/active = true/' /etc/pihole/pihole.toml
pihole-FTL --config dhcp.start 10.0.0.50
pihole-FTL --config dhcp.end 10.0.0.200
pihole-FTL --config dhcp.router 10.0.0.1
pihole-FTL --config dhcp.netmask 255.255.255.0
exit
docker restart pihole
```

> **TIP:** Rogers Ignite gateways do not allow you to change DNS or disable their DHCP server. Because of this, Pi-hole's DHCP will conflict with Rogers'. The workaround is to manually set DNS to `10.0.0.2` on each device individually rather than pushing it network-wide.
>
> On iPhone, Tailscale overrides manually set DNS. Fix this by opening Tailscale → DNS → add `10.0.0.2` as a custom nameserver and enable **Override local DNS**.

Access Pi-hole at `http://10.0.0.2/admin`.

---

### 7. Tailscale
Tailscale creates an encrypted mesh VPN between your devices so you can access your home server remotely without opening any ports. Install it directly on the Pi — not in Docker:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Follow the URL it provides to log in and authorize the Pi. Install the Tailscale app on your other devices and sign in with the same account. Confirm the connection:

```bash
tailscale ip
```

Your Pi will be reachable at its Tailscale IP (`100.x.x.x`) from anywhere, even on mobile data.

---

## Remote Access

Once Tailscale is connected, access everything via the Tailscale IP from anywhere:

| Service | Local URL | Remote URL |
|---|---|---|
| Heimdall | http://10.0.0.2:8888 | http://100.x.x.x:8888 |
| Pi-hole | http://10.0.0.2/admin | http://100.x.x.x/admin |
| Portainer | https://10.0.0.2:9443 | https://100.x.x.x:9443 |

Bookmark the Heimdall URL on your phone for a single entry point to all your services.
