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
timeout 5 nc -zvu HOSTNAME:PORT

# Check router port forwarding rules
# Expected: UDP PORT → Server IP
```

### 2. Check DNS Resolution
```bash
# Verify dynamic DNS resolves correctly
nslookup HOSTNAME DNS_SERVER

# Compare to server's actual public IP
ssh USER@SERVER "curl -4 -s ifconfig.me"
```

### 3. Inspect WireGuard Server Status
```bash
# List active peers on server
ssh USER@SERVER "sudo wg show INTERFACE peers"

# Check latest handshakes
ssh USER@SERVER "sudo wg show INTERFACE latest-handshakes"

# Expected: All configured peers should appear
```

### 4. Verify Client Configuration
Key fields in client config:
```ini
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = VPN_IP/32
DNS = DNS_SERVER_1, DNS_SERVER_2

[Peer]
PublicKey = SERVER_PUBLIC_KEY  # Must match server's public key
Endpoint = HOSTNAME:PORT
AllowedIPs = 0.0.0.0/0  # Full tunnel
PersistentKeepalive = 25
```

**Common mistake**: Client's Peer section contains client's own public key instead of server's public key.

### 5. Check Server Configuration File
```bash
ssh USER@SERVER "sudo cat /etc/wireguard/INTERFACE.conf | grep -A 5 '\[Peer\]'"
```

Expected peer sections:
```ini
[Peer]
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = VPN_IP/32
PersistentKeepalive = 25
```

## The Fix

### Issue Identified
Server configuration file had Android peer defined, but running WireGuard interface did not have it loaded.

### Solution
Add peer to running interface without restarting:
```bash
ssh USER@SERVER "sudo wg set INTERFACE peer CLIENT_PUBLIC_KEY \
  allowed-ips VPN_IP/32 \
  persistent-keepalive 25"
```

### Verification
```bash
# Confirm peer is now active
ssh USER@SERVER "sudo wg show INTERFACE peers"

# Check for successful handshake
ssh USER@SERVER "sudo wg show INTERFACE | grep -A 5 'CLIENT_PUBLIC_KEY'"

# Test connectivity from client
ping -c 4 VPN_SERVER_IP
```

## Success Criteria
- Latest handshake timestamp shows recent connection
- Transfer statistics show bidirectional data (received AND sent)
- Client can ping server's VPN IP
- Endpoint shows client's public IP and port

Example output:
```
peer: CLIENT_PUBLIC_KEY
  endpoint: CLIENT_PUBLIC_IP:RANDOM_PORT
  allowed ips: VPN_IP/32
  latest handshake: 45 seconds ago
  transfer: XX.XX KiB received, XX.XX KiB sent
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
- Must forward **UDP** (not TCP) on the WireGuard port
- Verify with external connectivity test
- Some ISPs block VPN protocols

### 4. Firewall Issues
- Ensure iptables/ufw allows UDP on the WireGuard port
- Check with: `sudo iptables -L -n -v | grep PORT`

### 5. DNS Resolution on Mobile Networks
- Some carriers use DNS hijacking or proxies
- May cause SSL/TLS warnings on legitimate sites
- Doesn't affect WireGuard connectivity (uses IP directly)

## Making Configuration Persistent
After confirming live changes work:
```bash
# Update config file
ssh USER@SERVER "sudo nano /etc/wireguard/INTERFACE.conf"

# Restart to verify persistence
ssh USER@SERVER "sudo systemctl restart wg-quick@INTERFACE"
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
# Test connectivity to VPN server
ping -c 4 VPN_SERVER_IP

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
- Dynamic DNS services
