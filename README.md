# Homelab Infrastructure, Virtualization & Home Automation

![Proxmox VE](https://img.shields.io/badge/Proxmox_VE-Cluster-E57000?style=flat-square&logo=proxmox&logoColor=white)
![Unraid](https://img.shields.io/badge/Unraid-Storage_&_Docker-F15A2C?style=flat-square&logo=unraid&logoColor=white)
![Home Assistant](https://img.shields.io/badge/Home_Assistant-OS-41BDF5?style=flat-square&logo=homeassistant&logoColor=white)
![TP-Link Omada](https://img.shields.io/badge/TP--Link_Omada-SDN-005564?style=flat-square)
![Frigate NVR](https://img.shields.io/badge/Frigate-AI_NVR-3B82F6?style=flat-square)

Comprehensive documentation, architecture diagrams, and operational runbooks for my self-hosted multi-node virtualization cluster, software-defined network (SDN), AI-powered surveillance pipeline, and IoT automation ecosystem.

---

## Architecture Overview

This infrastructure separates **high-availability core services** from **high-throughput storage and GPU workloads**:
1. **Compute & High Availability (Proxmox VE Cluster):** A 3-node mini-PC cluster housed in a custom **FDM 3D-printed 10-inch rack chassis** running ZFS storage replication and automatic High Availability (HA) failover for critical services (Home Assistant OS, reverse proxy, and DNS).
2. **Bulk Storage & Hardware Acceleration (Unraid Tower):** A dedicated tower server handling NAS storage, Docker workloads, and GPU-accelerated video decoding/AI inference for Frigate NVR on an isolated surveillance drive.
3. **Software-Defined Networking & Zero-Trust Edge:** Managed centrally via TP-Link Omada SDN with local AdGuard DNS sinkholing, split-horizon DNS routing, and encrypted Cloudflare Tunnels for external access without exposing public ports.

### Network & Service Topology

```mermaid
graph TD
    Internet((Public Internet)) -->|Encrypted Tunnel| CF[Cloudflare Edge / Tunnel]
    
    subgraph LAN ["Local Network (192.168.0.0/24) - TP-Link Omada SDN"]
        GW[Omada Router / Gateway<br/>192.168.0.1]
        AP[Omada EAP610 Access Points<br/>2.4GHz / 5GHz Wi-Fi & PoE]
        DNS[AdGuard Home DNS<br/>192.168.0.26 / .27<br/>Split-Horizon Rewrites]
        NPM[Nginx Proxy Manager LXC<br/>192.168.0.22]
        
        CF -->|Internal Routing| NPM
        GW --- AP
        GW --- DNS
        NPM --> ProxieCluster
        NPM --> Unraid
    end

    subgraph ProxieCluster ["3-Node Proxmox VE Cluster (10-Inch 3D Printed Rack)"]
        P1[ProxieOne<br/>i5 8500T | 16GB RAM | ZFS]
        P2[ProxieTwo<br/>Dell Wyse 5070 | ZFS]
        P3[ProxieThree<br/>Dell Wyse 5070 | ZFS]
        
        HAOS[Home Assistant OS VM<br/>HA Failover & ZFS Replication]
        MQTT[Mosquitto MQTT Broker<br/>192.168.0.27:1883]
        
        P1 <-->|ZFS Sync & Quorum| P2
        P2 <-->|ZFS Sync & Quorum| P3
        P1 <-->|ZFS Sync & Quorum| P3
        P1 --- HAOS
        HAOS --- MQTT
    end

    subgraph Unraid ["Unraid Storage & Compute Server (192.168.0.8)"]
        CPU[Ryzen 5 5600X | 32GB DDR4]
        GPU[NVIDIA GTX 1060<br/>NVDEC Hardware Decoding]
        Frigate[Frigate NVR Docker<br/>Port 5000 API / 8971 WebUI]
        WDPurple[(WD Purple Surveillance HDD<br/>Unassigned Device - XFS)]
        
        GPU --> Frigate
        Frigate -->|Direct Event Writes| WDPurple
    end

    subgraph IoT ["IoT, Security & Embedded Devices"]
        Cams[Reolink IP Cameras<br/>PoE & Mains Wi-Fi RTSP]
        Zigbee[SMLIGHT SLZB-07<br/>Zigbee Coordinator]
        ESP[ESP32 + W5500 Ethernet<br/>LD2410C / LD2450 mmWave]
        WLED[ESP8266/ESP32 WLED<br/>192.168.0.180]
    end

    Cams -->|RTSP Streams| Frigate
    Frigate <-->|Detection Events & Telemetry| MQTT
    Frigate -->|Port 5000 Local API| HAOS
    Zigbee -->|Zigbee Mesh| HAOS
    ESP -->|ESPHome Native API / IPv4| HAOS
    WLED -->|Wi-Fi| AP
