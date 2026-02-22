# Home Server Deployment & Troubleshooting Guide

**Domain:** yourdomain.com\
**Server:** Ubuntu + Cloudflare Tunnel + SSH + MongoDB + K3s

------------------------------------------------------------------------

## 🌍 DOMAIN & CLOUDFLARE SETUP

### Move Domain to Cloudflare

1.  Create Cloudflare account\
2.  Add domain → Choose Free Plan\
3.  Copy Cloudflare nameservers\
4.  Update nameservers at registrar\
5.  Wait until domain becomes **Active**

Verify:

    nslookup yourdomain.com

Must return Cloudflare IPs (104.x.x.x / 172.x.x.x).

------------------------------------------------------------------------

## 🚇 CLOUDFLARE TUNNEL SETUP

### Install cloudflared

    wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
    sudo dpkg -i cloudflared-linux-amd64.deb

Login:

    cloudflared tunnel login

Create tunnel:

    cloudflared tunnel create myserver

------------------------------------------------------------------------

### Config File Location (IMPORTANT)

System services use:

    /etc/cloudflared/config.yml

Create/edit:

    sudo nano /etc/cloudflared/config.yml

Example:

``` yaml
tunnel: TUNNEL_ID
credentials-file: /home/USER/.cloudflared/TUNNEL_ID.json

ingress:
  - hostname: app.yourdomain.com
    service: http://localhost:8000

  - service: http_status:404
```

------------------------------------------------------------------------

### Route DNS

    cloudflared tunnel route dns myserver subdomain.yourdomain.com

------------------------------------------------------------------------

### Restart Tunnel

    sudo systemctl restart cloudflared
    sudo systemctl status cloudflared

Must show **active (running)**.

------------------------------------------------------------------------

## 🔐 SSH SETUP VIA CLOUDFLARE

### SSH Ingress Rule

Add to config:

``` yaml
- hostname: ssh.yourdomain.com
  service: ssh://localhost:22
```

Restart tunnel.

------------------------------------------------------------------------

### Connect from Mac

    ssh user@ssh.yourdomain.com   -o ProxyCommand="cloudflared access ssh --hostname %h"

------------------------------------------------------------------------

### Copy SSH Key

Correct order:

    ssh-copy-id -o ProxyCommand="cloudflared access ssh --hostname %h" user@ssh.yourdomain.com

------------------------------------------------------------------------

### Disable Password Login (Recommended)

On server:

    sudo nano /etc/ssh/sshd_config

Set:

    PasswordAuthentication no
    PermitRootLogin no

Restart:

    sudo systemctl restart ssh

------------------------------------------------------------------------

## 🗄 MONGODB SETUP (NO DOCKER)

### Start MongoDB

    sudo systemctl start mongod
    sudo systemctl enable mongod

Check:

    sudo systemctl status mongod

------------------------------------------------------------------------

### Mongo Shell Commands

    mongosh
    show dbs
    use mydb
    show collections
    db.collection.find().pretty()

------------------------------------------------------------------------

### SAFE BINDING (CRITICAL)

    bindIp: 127.0.0.1

Never expose MongoDB publicly.

------------------------------------------------------------------------

### SSH Tunnel from Mac

    ssh -fN -L 27018:localhost:27017 user@ssh.yourdomain.com   -o ProxyCommand="cloudflared access ssh --hostname %h"

Compass URI:

    mongodb://localhost:27018

------------------------------------------------------------------------

### ECONNREFUSED Errors

Cause:

✔ Tunnel not running\
✔ MongoDB not running

Fix:

    lsof -i :27018
    sudo systemctl restart mongod

------------------------------------------------------------------------

## ☸️ KUBERNETES (K3S) SETUP

### Install K3s

    curl -sfL https://get.k3s.io | sh -

Verify:

    sudo k3s kubectl get nodes

------------------------------------------------------------------------

### Fix Permission Errors

    sudo chmod 644 /etc/rancher/k3s/k3s.yaml
    mkdir -p ~/.kube
    sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
    sudo chown $USER:$USER ~/.kube/config

Test:

    kubectl get nodes

------------------------------------------------------------------------

### Restart K3s

    sudo systemctl restart k3s
    sudo systemctl status k3s

------------------------------------------------------------------------

### Deploy Test App

    kubectl create deployment nginx --image=nginx
    kubectl expose deployment nginx --port 80 --type ClusterIP

------------------------------------------------------------------------

### Expose via Domain (Ingress)

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx
spec:
  rules:
  - host: app.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```

Apply:

    kubectl apply -f ingress.yaml

Tunnel service:

    service: http://localhost:80

------------------------------------------------------------------------

## 🔁 COMMON RESTART COMMANDS

### Restart Everything

    sudo systemctl restart cloudflared
    sudo systemctl restart ssh
    sudo systemctl restart mongod
    sudo systemctl restart k3s

------------------------------------------------------------------------

## 🚨 TROUBLESHOOTING CHECKLIST

### Domain Not Resolving

    nslookup subdomain.yourdomain.com

If fails → DNS issue.

------------------------------------------------------------------------

### Tunnel Issues

    sudo systemctl status cloudflared
    journalctl -u cloudflared -f

------------------------------------------------------------------------

### Backend Not Working

Test locally:

    curl http://localhost:PORT

------------------------------------------------------------------------

### MongoDB Connection Refused

✔ SSH tunnel missing\
✔ MongoDB stopped

------------------------------------------------------------------------

### Kubernetes Issues

    kubectl get pods -A
    sudo systemctl status k3s

------------------------------------------------------------------------

## 🔄 MOVING TO ANOTHER DOMAIN

### Steps

1.  Add new domain to Cloudflare\
2.  Update nameservers\
3.  Update tunnel DNS routes:

```{=html}
<!-- -->
```
    cloudflared tunnel route dns myserver newsub.newdomain.com

4.  Edit config.yml hostnames\
5.  Restart tunnel

------------------------------------------------------------------------

## 🌐 SWITCHING TO IP-BASED ACCESS (LAN / VPN ONLY)

### Get Server IP

    ip a

Example:

    192.168.x.x

------------------------------------------------------------------------

### MongoDB Bind

    bindIp: 127.0.0.1,192.168.x.x

Restart MongoDB.

------------------------------------------------------------------------

### Compass URI

    mongodb://192.168.x.x:27017

⚠ Never use public IP without firewall & auth.

------------------------------------------------------------------------

## ✅ BEST PRACTICES SUMMARY

✔ Never expose database ports publicly\
✔ Always use SSH tunnels\
✔ Keep cloudflared as reverse proxy\
✔ Enable services on boot\
✔ Test locally before debugging network

------------------------------------------------------------------------

**Your Server Architecture**

Internet → Cloudflare → Tunnel → Local Services / K3s / SSH

Secure, stable, production-style setup 👌

--------------------------------------------------------------------------

## Made by @AryanAg08 with ❤️
