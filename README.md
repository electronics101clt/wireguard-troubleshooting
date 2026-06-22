# WireGuard VPN Troubleshooting Guide

## Problem Summary
Mobile client unable to connect to WireGuard VPN server from internet despite correct configuration file.

## Environment
- **Server**: Ubuntu server with WireGuard installed
- **Client**: Android device on mobile data (not local network)
- **Setup**: Port forwarding configured on router (UDP port 51820)

## Symptoms
- Client showed data transmission (tx) but no reception (rx: 0 B)
- VPN tunnel appeared "on" but no connectivity
- SSH connections to VPN subnet failed

## Root Cause
Server's running WireGuard interface was missing the Android peer configuration, despite the peer being present in `/etc/wireguard/wg0.conf`.

## Diagnostic Steps

### 1. Verify Port Forwarding
```bash
# Test external accessibility of WireGuard port
timeout 5 nc -zvu <domain>:51820

# Check router port forwarding rules
# Expected: UDP 51820 → Server IP
```

### 2. Check DNS Resolution
```bash
# Verify DuckDNS/dynamic DNS resolves correctly
nslookup <domain> 1.1.1.1

# Compare to server's actual public IP
ssh user@server "curl -4 -s ifconfig.me"
```

### 3. Inspect WireGuard Server Status
```bash
# List active peers on server
ssh user@server "sudo wg show wg0 peers"

# Check latest handshakes
ssh user@server "sudo wg show wg0 latest-handshakes"

# Expected: All configured peers should appear
```

### 4. Verify Client Configuration
Key fields in client config:
```ini
[Interface]
PrivateKey = <client_private_key>
Address = <vpn_ip>/32
DNS = 8.8.8.8, 8.8.4.4

[Peer]
PublicKey = <SERVER_public_key>  # Must match server's public key
Endpoint = <domain>:51820
AllowedIPs = 0.0.0.0/0  # Full tunnel
PersistentKeepalive = 25
```

**Common mistake**: Client's Peer section contains client's own public key instead of server's public key.

### 5. Check Server Configuration File
```bash
ssh user@server "sudo cat /etc/wireguard/wg0.conf | grep -A 5 '\[Peer\]'"
```

Expected peer sections:
```ini
[Peer]
PublicKey = <client_public_key>
AllowedIPs = <vpn_ip>/32
PersistentKeepalive = 25
```

## The Fix

### Issue Identified
Server configuration file had Android peer defined, but running WireGuard interface did not have it loaded.

### Solution
Add peer to running interface without restarting:
```bash
ssh user@server "sudo wg set wg0 peer <CLIENT_PUBLIC_KEY> \
  allowed-ips <VPN_IP>/32 \
  persistent-keepalive 25"
```

### Verification
```bash
# Confirm peer is now active
ssh user@server "sudo wg show wg0 peers"

# Check for successful handshake
ssh user@server "sudo wg show wg0 | grep -A 5 '<CLIENT_PUBLIC_KEY>'"

# Test connectivity from client
adb shell "ping -c 4 <VPN_SERVER_IP>"
```

## Success Criteria
- Latest handshake timestamp shows recent connection
- Transfer statistics show bidirectional data (received AND sent)
- Client can ping server's VPN IP
- Endpoint shows client's public IP and port

Example output:
```
peer: <client_public_key>
  endpoint: <client_public_ip>:<random_port>
  allowed ips: <vpn_ip>/32
  latest handshake: 45 seconds ago
  transfer: 82.32 KiB received, 96.15 KiB sent
  persistent keepalive: every 25 seconds
```

## Common Pitfalls

### 1. Public Key Mismatch
- **Client**: Peer section must contain SERVER's public key
- **Server**: Peer section must contain CLIENT's public key
- Cross-check with: `echo "PRIVATE_KEY" | wg pubkey`

### 2. Configuration File vs Running Interface
- Editing `/etc/wireguard/wg0.conf` doesn't affect running interface
- Use `sudo wg set wg0 ...` to modify live configuration
- OR restart interface: `sudo wg-quick down wg0 && sudo wg-quick up wg0`

### 3. Router Port Forwarding
- Must forward **UDP** (not TCP) port 51820
- Verify with external connectivity test
- Some ISPs block VPN protocols

### 4. Firewall Issues
- Ensure iptables/ufw allows UDP 51820
- Check with: `sudo iptables -L -n -v | grep 51820`

### 5. DNS Resolution on Mobile Networks
- Some carriers use DNS hijacking or proxies
- May cause SSL/TLS warnings on legitimate sites
- Doesn't affect WireGuard connectivity (uses IP directly)

## Making Configuration Persistent
After confirming live changes work:
```bash
# Update config file
ssh user@server "sudo nano /etc/wireguard/wg0.conf"

# Restart to verify persistence
ssh user@server "sudo systemctl restart wg-quick@wg0"
```

## Additional Tools

### Android Client Setup
1. Install WireGuard app from Play Store
2. Generate config file with correct keys
3. Transfer via:
   - ADB: `adb push config.conf /sdcard/Download/`
   - QR code generation
   - Manual entry

### Testing Client Connectivity
```bash
# Test from Android via ADB
adb shell "ping -c 4 <vpn_server_ip>"

# Check VPN interface on client
# WireGuard app shows tx/rx statistics
```

## Security Notes
- Never share private keys
- Store configuration files securely (chmod 600)
- Use strong, randomly generated keys (`wg genkey`)
- Regularly rotate keys for high-security environments
- Monitor handshake timestamps for unauthorized access

## References
- WireGuard documentation: https://www.wireguard.com/
- Port forwarding verification tools
- Dynamic DNS services (DuckDNS, No-IP, etc.)
