# ☸️ Kubernetes GitOps Infrastructure

This repository contains the declarative GitOps state of bare-metal Kubernetes cluster running on a dedicated server (Hetzner) virtualized via Proxmox. 

The entire infrastructure and all applications are managed via **ArgoCD** using the "App of Apps" pattern, ensuring that the cluster state always strictly mirrors this Git repository.

## 🛠️ Core Tech Stack

* **OS & Provisioning:** [Talos Linux](https://www.talos.dev/)
* **GitOps Controller:** [ArgoCD](https://argoproj.github.io/cd/)
* **Ingress & Networking:** NGINX Ingress Controller, MetalLB, Cert-Manager
* **Storage & Databases:** Longhorn (Distributed Block Storage), CloudNativePG
* **Secret Management:** Bitnami Sealed Secrets

---

## 🗺️ Network Topology

The architecture is designed to be **GitOps ready** and utilizes a strict **Double-NAT** topology for maximum isolation between public traffic and the internal network. 

Proxmox acts solely as an L1/L2 switch and a "dumb" NAT gateway. All L3 routing, firewalling, DHCP, and VPN services are handled by a virtualized **OPNsense** appliance.

```mermaid
graph TD
    Internet((Internet)) -->|Public IP| Proxmox[Proxmox Host<br>eno1]
    
    subgraph "Hetzner Server"
        Proxmox -->|NAT 1:1<br>WAN Link /30| OPNsense[OPNsense VM<br>Firewall & Router]
        
        subgraph "Internal LAN (10.10.10.0/24)"
            OPNsense -->|Gateway .1| Switch[vmbr1<br>Virtual Switch]
            Switch -->|Mgmt .2| PVE_Mgmt[Proxmox Mgmt]
            Switch -->|.10| TalosVIP[Talos API VIP]
            Switch -->|.11| Node1[Talos Node 1]
            Switch -->|.12| Node2[Talos Node 2]
            Switch -->|.13| Node3[Talos Node 3]
            Switch -->|.200| Ingress[Ingress VIP]
        end
    end
```

### Network Interfaces (Proxmox Bridges)

| Interface | Type | IP Address / CIDR | Description & Purpose |
| :--- | :--- | :--- | :--- |
| **eno1** | Physical | *Public IP* | Physical ingress. Performs DNAT forwarding to OPNsense. |
| **vmbr0** | Bridge | `192.168.100.1/30` | **WAN Link.** Private point-to-point link between Proxmox and OPNsense. |
| **vmbr1** | Bridge | `10.10.10.2/24` | **LAN Switch.** Main internal network for the Cluster, Management, and Apps. |

---

## 📍 IP Address Plan (LAN)

* **Subnet:** `10.10.10.0/24`
* **Gateway:** `10.10.10.1` (OPNsense)
* **DNS:** `10.10.10.1` (OPNsense Unbound DNS)

| IP Address                   | Hostname        | Role            | Description                                                 |
| :--------------------------- | :-------------- | :-------------- | :---------------------------------------------------------- |
| **Infrastructure** |                 |                 |                                                             |
| `10.10.10.1`                 | **OPNsense** | **Gateway** | Main router, DHCP, Tailscale Exit Node.                     |
| `10.10.10.2`                 | **Proxmox** | Management      | Proxmox internal interface (SSH, Web GUI).                  |
| `10.10.10.3` - `.9`          | *Reserved* | -               | Reserved for core network devices.                          |
| **Kubernetes Control Plane** |                 |                 |                                                             |
| `10.10.10.10`                | **Talos VIP** | **Cluster VIP** | **Virtual IP for API Server.** Target for `talosctl`/ArgoCD.|
| `10.10.10.11`                | `talos-node-1`  | CP              | Control Plane Node 1.                                       |
| `10.10.10.12`                | `talos-node-2`  | CP              | Control Plane Node 2.                                       |
| `10.10.10.13`                | `talos-node-3`  | CP              | Control Plane Node 3.                                       |
| **Applications (MetalLB)** |                 |                 |                                                             |
| `10.10.10.200`               | **Ingress VIP** | **Ingress** | Main ingress for HTTP/HTTPS (Websites, Webhooks).           |
| `10.10.10.201`               | *DB Access* | VIP             | Direct TCP access to the HA database (currently not configured).            |
| `10.10.10.202` - `.250`      | *Pool* | LoadBalancer    | Dynamic pool for standard LoadBalancer services.            |
| `10.10.10.100` - `.199`      | *DHCP* | Clients         | DHCP range for temporary VMs and VPN clients.               |

---

## 🛡️ Remote Access (VPN / Tailscale)

All management access (SSH, Proxmox GUI, Talos API) is strictly hidden behind the firewall and is **not exposed** to the public internet. Secure access is provided via **Tailscale**.

* **Location:** The `os-tailscale` plugin running directly on the OPNsense router.
* **Configuration:**
    * **Subnet Router:** Enabled (`--advertise-routes=10.10.10.0/24`).
    * **Exit Node:** Optionally enabled (for secure browsing tunneling through the server).
* **Access Points (via VPN):**
    * **Proxmox GUI:** `https://10.10.10.2:8006`
    * **Talos API:** `https://10.10.10.10:6443`
    * **OPNsense GUI:** `https://10.10.10.1`

---

## ⚙️ Host Configuration (Proxmox)

The `/etc/network/interfaces` file handles routing between the physical network and the OPNsense VM.

**Key Security Concepts:**
* The `vmbr0` network is strictly a `/30` (allowing only 2 hosts), minimizing the attack surface.
* Proxmox holds the `.2` IP on `vmbr1` but **does not** have a gateway configured there (it uses `eno1` as its default route).

```bash
auto lo
iface lo inet loopback

iface lo inet6 loopback

auto eno1
iface eno1 inet static
    address <PUBLIC_IP>/26
    gateway <PUBLIC_GATEWAY>

# --- WAN Link (Proxmox <-> OPNsense connection) ---
# Network: 192.168.100.0/30
# .1 = Proxmox Host
# .2 = OPNsense WAN
auto vmbr0
iface vmbr0 inet static
    address 192.168.100.1/30
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    
    # Enable IP forwarding in the kernel
    post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
    
    # 1. OUTBOUND NAT (Masquerade)
    # Everything leaving OPNsense (.100.2) is masqueraded behind the server's public IP
    post-up   iptables -t nat -A POSTROUTING -s '192.168.100.0/30' -o eno1 -j MASQUERADE
    post-down iptables -t nat -D POSTROUTING -s '192.168.100.0/30' -o eno1 -j MASQUERADE
    
    # 2. INBOUND PORT FORWARDING (Internet traffic)
    # HTTP (80) -> OPNsense
    post-up   iptables -t nat -A PREROUTING -i eno1 -p tcp --dport 80 -j DNAT --to 192.168.100.2:80
    post-down iptables -t nat -D PREROUTING -i eno1 -p tcp --dport 80 -j DNAT --to 192.168.100.2:80
    
    # HTTPS (443) -> OPNsense
    post-up   iptables -t nat -A PREROUTING -i eno1 -p tcp --dport 443 -j DNAT --to 192.168.100.2:443
    post-down iptables -t nat -D PREROUTING -i eno1 -p tcp --dport 443 -j DNAT --to 192.168.100.2:443

# --- LAN Switch (Internal Network) ---
auto vmbr1
iface vmbr1 inet static
    address 10.10.10.2/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    # Gateway is intentionally omitted here. Proxmox uses eno1 as the default gateway.
    # The 10.10.10.2 IP is strictly for management accessibility from the LAN/VPN.
```

## 🚦 Packet Flow

### 1. Outbound Traffic
*Example: A Talos Node pulling a Docker Image from Docker Hub.*

1. **Talos Node** (`.11`) sends a request to its default gateway → **OPNsense** (`.1`).
2. **OPNsense** performs NAT and routes the request to its gateway → **Proxmox** (`.100.1`).
3. **Proxmox** performs a second NAT (Masquerade) and sends the request out to the **Internet**.

### 2. Inbound Traffic (Webhooks)
*Example: An external service calling `https://webhook.mracek.dev`.*

1. The **Internet** hits the server's Public IP on port 443.
2. **Proxmox** (`eno1`) intercepts the packet and `iptables` forwards it via DNAT to the **OPNsense WAN** (`192.168.100.2`).
3. **OPNsense** receives the packet and applies a Port Forward rule, routing it to the internal **Ingress VIP** (`10.10.10.200`).
4. **MetalLB** inside the cluster (via ARP) receives the packet on the node currently running the Ingress Controller speaker.
5. **NGINX Ingress** terminates the TLS connection and forwards the request to the `n8n` service pod.
