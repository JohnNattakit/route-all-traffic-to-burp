Step-by-step guide to route **all traffic from your phone (connected via Wi-Fi hotspot on Kali)** through **Burp Suite** using `iptables`. This approach uses **Kali as a transparent proxy**.

------

## ✅ Overview:

1. **eth0** = Internet
2. **wlan0** = Wi-Fi hotspot (your phone connects to this)
3. **Burp Suite** = Listening on port `8080` on Kali
4. Use `iptables` to **redirect HTTP/S traffic** to Burp

------

## 🛠 Prerequisites:

- **Burp Suite is running** with listener on `8080`, **invisible mode** enabled
- **IP forwarding** is enabled
- Your **phone trusts Burp CA certificate**

------

## 🔧 Step-by-Step:

### 1. Enable IP forwarding

```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

(Optional: make it permanent in `/etc/sysctl.conf` by setting `net.ipv4.ip_forward=1`)

------

### 2. Configure Burp:

- Go to **Proxy > Options > Proxy Listeners**
- Add listener on **all interfaces** (`0.0.0.0`) or just on `wlan0`
- Enable **"Support invisible proxying"**

------

### 3. Set up `iptables` NAT to redirect traffic to Burp

Replace `<YOUR_KALI_IP>` with the IP of your `wlan0` (run `ip a | grep wlan0` to find it)

```
t# Clear old rules
iptables -F
iptables -t nat -F

# Allow traffic from wlan0 to internet (NAT)
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Redirect HTTP traffic from phone to Burp
iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 80 -j REDIRECT --to-port 8080
iptables -t nat -A PREROUTING -i wlan0 -p tcp --dport 443 -j REDIRECT --to-port 8080

# Allow forwarding
iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o wlan0 -m state --state ESTABLISHED,RELATED -j ACCEPT
```

------

### 4. Trust Burp CA on your Phone

- Export CA from Burp: `Proxy > Options > Import / Export > Export Certificate`
- Transfer `.cer` file to your phone
- Install it as **trusted user certificate** (network > VPN & apps > install certificate)

> For HTTPS to work, **you must install Burp CA on the phone**. Otherwise, you'll get SSL errors.

------

## ✅ Done!

Now, when your phone browses any site:

- HTTP/HTTPS is transparently intercepted
- Burp Suite will see the raw traffic

------

### 📌 Optional: Persist iptables rules

You can save the config using:

```
iptables-save > /etc/iptables/rules.v4
```

And restore using `iptables-restore`.








### Single Bash script that will completely reset iptables to default (allow all, no NAT, no forwarding) and disable IP forwarding:
```
#!/bin/bash

echo "[+] Flushing all iptables rules..."
iptables -F
iptables -t nat -F
iptables -t mangle -F

echo "[+] Deleting all user-defined chains..."
iptables -X
iptables -t nat -X
iptables -t mangle -X

echo "[+] Setting default policies to ACCEPT..."
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

echo "[+] Disabling IP forwarding..."
echo 0 > /proc/sys/net/ipv4/ip_forward

echo "[✓] iptables reset complete."
```
