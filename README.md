# My-homelab
My personal homelab setup for learning Linux administration with Ubuntu Server, Nginx, and WireGuard VPN with Wake-on-LAN.
------------------------------------------
The lab is isolated from the primary residential network using a dedicated Layer 3 switch, balancing continuous availability with power efficiency.

**Hardware Architecture**
Node 1: Low-Power (24/7 Node)

  CPU: Intel Celeron N3160
  GPU: Intel Integrated Graphics
  RAM: 4GB DDR3L
  Power Draw: 10W - 15W (Idle)
  Role: Continuous routing, VPN gateway, and lightweight web hosting. 

Node 2: High-Power (On-Demand Compute Node)

  CPU: Intel Xeon E3-1231 v3 (3.4 GHz)
  GPU: Matrox G200
  RAM: 16GB DDR3 ECC
  Storage: 2 x 3TB Hard Disk Drives
  Power Draw: 35W - 50W (Idle)
  Role: Handling heavier compute for future projects like web scraping, data analysis, and storage.

Network Switch
  Model: Cisco SG300-28 (L3 mode)
  Role: Handles inter-VLAN routing between nodes, isolates lab traffic from the residential environment, and optimizes power usage by disabling unused ports.
  Firmware: Manually updated to the final stable legacy firmware release (v1.4.11.5) to ensure platform stability.

------------------------------------------
**Core Infrastructure Implementation Details**

1. **Operating System Configuration**
   Distribution: Ubuntu Server (LTS, 64-bit) deployed via minimal installation on both nodes to avoid desktop environment overhead.
   Access Control: Headless configuration using a non-root system user with administrative privileges managed via explicitly audited sudo permissions.

2. **Local Networking**
   Interface: Wired Ethernet connection on enp2s0.
   IP Allocation: Assigned static internal IP address 192.168.1.200 to prevent DHCP lease rotation from disrupting server accessibility.

3. **SSH Hardening**

   To prevent automated brute-force attacks from the public internet,
   the OpenSSH server was hardened using the following steps:
   * Disabled password-based authentication completely (PasswordAuthentication no).
   * Enforced cryptographic public-key authentication only.
   * Configured local firewall exceptions exclusively for authorized subnets.

4. **Firewall (UFW)**

  The Uncomplicated Firewall (UFW) enforces a strict default-deny policy
  for incoming traffic and standard allowances for outgoing traffic.

   Port / Protocol     Service          Description
   -------------------------------------------------------------------------
   TCP 80              HTTP             Internal Nginx web routing
   TCP 8080            HTTP-Alt         External public web traffic entry point
   UDP 51820           WireGuard        Inbound VPN tunnel handshake

5. **Web Server Configuration**

  Nginx handles ingress HTTP requests. It is configured to run automatically at system boot, serving a baseline status website.
  Internal Test Endpoints: http://localhost and http://192.168.1.200
  Binding: Configured to bind directly to port 80

6. **Edge Routing and Port Forwarding**
  Edge traffic routing is managed via a Huawei DN8245V-70 router utilizing Network Address Translation (NAT) 
  port mapping to route traffic from the public WAN interface directly to the internal host (192.168.1.200).

  Public WAN Interface        
     Port 8080 (TCP)  ---> Forward to 192.168.1.200:80 (Nginx)
     Port 2222 (TCP)  ---> Forward to 192.168.1.200:2222 (SSH)
     Port 51820 (UDP) ---> Forward to 192.168.1.200:51820 (WireGuard)
  Note: DMZ hosting was explicitly tested and deactivated to minimize potential network exposure.

7. **Public DNS Infrastructure**

  Domain Registrar: Managed via external service.
  Records: Configured a fundamental DNS A Record mapping the domain directly to the public IPv4 address fetched via curl -4 ifconfig.me.
  Network Constraints: Due to local ISP blocks on standard residential ports 80 (HTTP) and 443 (HTTPS), 
  external traffic must utilize port 8080 to reach the web server interface (http://public_ip:8080).
  
8. **Secure Remote Access (WireGuard VPN)**
  To allow secure access to local management tools without exposing internal services to the public internet, a WireGuard peer-to-peer VPN tunnel is established.

  [Remote Device] -> [WireGuard Tunnel (UDP 51820)] -> [Node 1 (Gateway)] -> [Internal Subnet (192.168.1.X)]

  Interface: wg0
  VPN Subnet: 10.0.0.1
  Client Configurations: Dedicated peer endpoints configured for mobile devices (e.g., personal phone assigned to 10.0.0.2).
  Keepalive: Persistent keepalives enabled on the mobile clients to maintain active state across dynamic cellular handovers.

  To ensure VPN clients can communicate beyond the VPN interface and reach devices across the wider home local area network, IP forwarding and iptables rules are configured.

  Enable packet forwarding in the kernel configuration (/etc/sysctl.conf):
  net.ipv4.ip_forward = 1
  Apply NAT masquerading to translate private VPN addresses to the server's local physical interface:
  iptables -t nat -A POSTROUTING -o enp2s0 -j MASQUERADE

  These network translation rules are configured to persist automatically across server power cycles.

10. **Power Management via Wake-on-LAN (WOL)**

    To save operational electrical costs, Server 2 is managed remotely using Wake-on-LAN commands sent through the 24/7 VPN tunnel.
    Verification: Validated that the network interface card supports magic packet interpretation using ethtool.
    State: Configured to wake up on magic packets (Wake-on: g).
    Workflow: Admin connects to the home lab network securely via WireGuard VPN from anywhere in the world, issues a magic packet to boot Server 2,
    executes resource-heavy data analysis/scraping tasks, and powers the machine down when completed.

11. **Current Setup and Future Goals of Project**
    
    The two nodes communicate with each other, safe remote access is available, and security was tightened as much as possible without sacrificing features.
    Communication is handled appropriately from the switch with unused ports and features disabled for power usage. Currently, the two nodes are not powered.
    Once projects (besides the current welcome page hosted at the registered domain) are ready, they will work as intended.

    Future possible projects:
    0) Update this GitHub repo with uploads of configuration templates stripped of personal information.
    1) Keep a database of home prices in my city and scrape websites to update prices, making sure to manage defensive scraping mechanics and not overstress the websites,
       with the goal of making an analysis of the data.
    3) Experiment with machine learning with the constraints of legacy devices and lower power usage in mind to produce some proof-of-concept service
       or a small research project from live data taken from open-source registries and public environmental sensors.






  
