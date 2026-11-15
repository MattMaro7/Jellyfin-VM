```mermaid

graph TD
    %% Styling
    classDef host fill:#2b2b2b,stroke:#4f4f4f,stroke-width:2px,color:#fff;
    classDef ts fill:#ff7e33,stroke:#d95d16,stroke-width:2px,color:#fff;
    classDef filter fill:#b3261e,stroke:#8c1d18,stroke-width:2px,color:#fff;
    classDef vm fill:#1f6feb,stroke:#0d57c6,stroke-width:2px,color:#fff;
    classDef app fill:#8e44ad,stroke:#732d91,stroke-width:2px,color:#fff;
    classDef disk fill:#238636,stroke:#1a6528,stroke-width:2px,color:#fff;
    classDef port fill:#d4af37,stroke:#aa841c,stroke-width:2px,color:#000;

    %% Network path
    Host["Windows host<br/>(on the tailnet)"]:::host
    TS(("Tailscale<br/>tailnet")):::ts
    ACL{"Tailscale ACL<br/>tag:server rules"}:::filter

    subgraph Debian["Debian 13 VM (headless, no desktop)"]
        FW{"ufw<br/>default deny inbound"}:::filter

        subgraph Ports["Allowed on the tailscale0 interface only"]
            Port22["Port 22<br/>SSH"]:::port
            Port8096["Port 8096<br/>Jellyfin"]:::port
        end

        VM["OS environment"]:::vm
        JF["Jellyfin service"]:::app
    end

    subgraph Storage["Virtual disks"]
        OSDisk[("OS disk, about 20 GB<br/>/ , /var , swap")]:::disk
        DataDisk[("Data disk, about 196 GB<br/>mounted at /mnt/mediadisk")]:::disk
    end

    %% Connections
    Host -->|"Encrypted tunnel"| TS
    TS --> ACL
    ACL --> FW
    FW --> Port22
    FW --> Port8096
    Port22 -->|"Remote management"| VM
    Port8096 -->|"Web UI and streaming"| JF

    %% Storage mapping
    OSDisk -.->|"System files"| VM
    DataDisk -.->|"Bind mounts:<br/>/var/lib/jellyfin<br/>/var/cache/jellyfin<br/>/srv/media"| JF

