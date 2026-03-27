📌 Project Overview
Successfully deployed a Cloudflare Zero Trust Tunnel within a restricted Unprivileged LXC Container on Proxmox VE. This project enables secure, identity-based remote access to internal services (Proxmox UI) without opening inbound firewall ports or using a Static IP.

🛠️ Technical Challenges & Solutions
1. LXC Permission Bottlenecks
Problem: Unprivileged containers lack access to the host kernel's TUN device, preventing the tunnel from initializing.
Solution: Implemented Device Passthrough on the Proxmox Host by modifying the container configuration (/etc/pve/lxc/ID.conf) to allow cgroup2 device access and mounting /dev/net/tun.

2. SSL/TLS Handshake Failures (Error 0A000410)
Problem: Persistent 0A000410 errors during the SSL handshake with Cloudflare's Edge.
Solution: Identified a time-sync mismatch. Installed and configured Chrony (NTP) to synchronize the system clock and updated the CA-Certificates trust store to ensure a valid TLS session.

3. ISP/Network Protocol Optimization
Problem: Local ISP/Network interference caused UDP-based (QUIC) connection timeouts.
Solution: Forced a protocol fallback to HTTP2 (TCP) using the --protocol http2 flag in the systemd service unit, ensuring a stable connection through the local gateway.

🚀 Deployment Script
Bash
# Force HTTP2 and No-Autoupdate for stability in restricted environments
ExecStart=/usr/bin/cloudflared tunnel --no-autoupdate run --token <TOKEN> --protocol http2
🌐 Architecture
External: Cloudflare DNS + WAF

Transport: Encrypted Tunnel (TCP/443)

Internal: Proxmox LXC (Ubuntu/Debian)

SSL Policy: SSL Termination with "No TLS Verify" for internal self-signed certificates.

🛠️ Diagnostic & Troubleshooting Log
To resolve the deployment issues, I utilized the following Linux CLI toolset to isolate failures at the System, Network, and Application layers:

1. Service Layer Analysis
Used journalctl to identify the specific failure point in the cloudflared daemon.

Bash
# Checked the last 20 lines of the service log
journalctl -u cloudflared --no-pager -n 20
Insight: Discovered failed to run the datagram handler, indicating a UDP/QUIC protocol blockage by the local ISP/Firewall.

2. Network Path & SSL Verification
Used curl with verbose flags to test the handshake between the LXC container and the Cloudflare Edge.

Bash
# Tested the TLS handshake and forced a maximum protocol version
curl -Iv --tls-max 1.2 https://region1.v2.argotunnel.com
Insight: Identified sslv3 alert handshake failure, which led to the discovery of a de-synchronized system clock (NTP issue) and the need for updated CA-Certificates.

3. Permission & Kernel Validation
Validated the visibility of the Network Tunnel device within the unprivileged container.

Bash
# Verified if the TUN device was successfully passed through from the Proxmox host
ls -l /dev/net/tun
Insight: Confirmed that the Proxmox .conf modifications were active, allowing the container to create the necessary network interface.

4. Application Ingress Debugging
Managed the transition from Local to Remote (Dashboard) configuration to resolve "Public Hostname" visibility issues.

Bash
# Uninstalled local-managed service to migrate to Dashboard-managed token
cloudflared service uninstall
cloudflared service install <NEW_REMOTE_TOKEN>
💡 Key Takeaways for Cloud Engineering
Protocol Flexibility: Sometimes "Modern" (UDP/QUIC) isn't "Reliable." Switching to HTTP2/TCP ensured 100% uptime in a restricted network.

Container Security: Navigating the boundaries of Unprivileged LXC requires a deep understanding of Linux device mapping and cgroups.

Zero Trust Principles: Successfully removed the need for Port Forwarding, reducing the attack surface of the home lab to zero public-facing ports.
