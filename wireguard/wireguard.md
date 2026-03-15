# wireguard-vpn

Absolutely! Here's a polished **README-style guide** with diagram, troubleshooting, and step-by-step clarity based on your setup. I’ve formatted it for easy reading and practical use.

---

# WireGuard VPN Gateway for Local Containers

**Hands-on guide to securely access local LXC/VM services via a VPS using WireGuard.**

This guide demonstrates:

* Setting up WireGuard on a minimal VPS
* Configuring a WireGuard client in a local LXC container
* Exposing local services (like Jellyfin, Odoo, Nextcloud) via the VPS
* Routing traffic through a single WireGuard client to multiple local containers

---

## **Scenario Overview**

| Component                                       | IP / Port       | Purpose                                  |
| ----------------------------------------------- | --------------- | ---------------------------------------- |
| VPS (WireGuard Server)                          | 140.245.195.196 | Public endpoint, VPN server, NAT gateway |
| WireGuard Interface on VPS                      | 10.10.10.1/24   | VPN subnet                               |
| LXC WireGuard Client (CT 100)                   | 10.10.10.2/24   | Routes traffic to other local containers |
| Local Service Container (e.g., Jellyfin CT 103) | 192.168.0.103   | Runs web service on 8096                 |
| WireGuard Port                                  | 51820 UDP       | VPN traffic                              |

---

## **Step 1: Prepare Minimal VPS**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wireguard iptables curl -y

# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## **Step 2: Configure WireGuard Server (VPS)**

1. Generate keys:

```bash
wg genkey | tee server_private.key | wg pubkey > server_public.key
```

2. Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.10.10.1/24
ListenPort = 51820
PrivateKey = <server_private.key>

PostUp = iptables -A INPUT -i wg0 -j ACCEPT
PostUp = iptables -A OUTPUT -o wg0 -j ACCEPT
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp = iptables -A FORWARD -o wg0 -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE

PostDown = iptables -D INPUT -i wg0 -j ACCEPT
PostDown = iptables -D OUTPUT -o wg0 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -D FORWARD -o wg0 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
```

3. Start server:

```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

4. Verify:

```bash
wg show
```

---

## **Step 3: Configure WireGuard Client (LXC CT 100)**

1. Install WireGuard:

```bash
sudo apt update && sudo apt install wireguard iptables -y
```

2. Generate keys:

```bash
wg genkey | tee client_private.key | wg pubkey > client_public.key
```

3. Create `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.10.10.2/24
PrivateKey = <client_private.key>

[Peer]
PublicKey = <server_public.key>
Endpoint = 140.245.195.196:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

4. Add client to VPS `/etc/wireguard/wg0.conf`:

```ini
[Peer]
PublicKey = <client_public.key>
AllowedIPs = 10.10.10.2/32
```

5. Start client:

```bash
sudo wg-quick up wg0
ping 10.10.10.1  # test connectivity to VPS
```

---

## **Step 4: Forward VPS Ports to Local Services**

### Example: Expose Jellyfin (192.168.0.103:8096)

**On VPS:**

```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 8096 -j DNAT --to-destination 10.10.10.2:8096
sudo iptables -A FORWARD -p tcp -d 10.10.10.2 --dport 8096 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -p tcp -d 10.10.10.2 --dport 8096 -j MASQUERADE
```

**On LXC WireGuard Client (CT 100):**

```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 8096 -j DNAT --to-destination 192.168.0.103:8096
sudo iptables -A FORWARD -p tcp -d 192.168.0.103 --dport 8096 -j ACCEPT
sudo iptables -t nat -A POSTROUTING -p tcp -d 192.168.0.103 --dport 8096 -j MASQUERADE
```

**Test:**

```bash
curl http://10.10.10.2:8096   # from VPS
http://140.245.195.196:8096   # from browser
```

---

## **Step 5: Traffic Flow Diagram**

```
 Internet
     │
     ▼
+----------------+
| VPS (140.245...)|
| WireGuard Server|
+--------+-------+
         |
         | VPN (10.10.10.0/24)
         ▼
+-----------------+
| LXC CT 100      |
| WireGuard Client|
+--------+--------+
         |
         | LAN forwarding
         ▼
+-----------------+
| Local Containers |
| (Jellyfin, etc.)|
+-----------------+
```

* **VPS:** Public endpoint, NAT router
* **LXC 100:** Single WireGuard client, gateway to local containers
* **Local containers:** Services accessible via port forwarding

---

## **Step 6: Extending to Multiple Services**

| Service   | Local IP      | Local Port | VPS Port | Forwarding Path             |
| --------- | ------------- | ---------- | -------- | --------------------------- |
| Jellyfin  | 192.168.0.103 | 8096       | 8096     | VPS → WG client → Jellyfin  |
| Odoo      | 192.168.0.104 | 8069       | 8069     | VPS → WG client → Odoo      |
| Nextcloud | 192.168.0.105 | 443        | 8443     | VPS → WG client → Nextcloud |

> Only **one WireGuard client** is required as long as it can route traffic to multiple containers.

---

## **Step 7: Troubleshooting**

| Issue                                       | Check / Fix                                                                          |
| ------------------------------------------- | ------------------------------------------------------------------------------------ |
| Cannot ping VPS from client                 | Check `wg show` on both sides, confirm `AllowedIPs` includes subnet                  |
| Service accessible from VPS but not browser | Verify port forwarding rules on VPS and client, open firewall/security group         |
| No traffic appears on `tcpdump`             | Confirm WireGuard is `up` and client is connected, check `iptables -t nat -L -n`     |
| 100% packet loss on ping                    | Check VPS IP forwarding (`sysctl net.ipv4.ip_forward`), ensure NAT rules are correct |
| Connection times out externally             | Check cloud provider ingress rules, firewall, VPS listening ports (`ss -ulnp`)       |

---

## **Step 8: Best Practices**

* Use **HTTPS** and reverse proxies for secure access.
* Restrict public access to known IPs if possible.
* Use a **single WireGuard client** to reduce complexity.
* Keep WireGuard configs **consistent and versioned**.

---

This setup allows you to **expose local apps securely** via a single VPS without exposing your entire LAN. You can replicate the port forwarding logic to host **multiple web apps**, **media servers**, or **internal dashboards**.

---

If you want, I can also create a **ready-to-use iptables + WireGuard script** for VPS and LXC to quickly replicate this setup without typing every command manually.

Do you want me to prepare that script?
