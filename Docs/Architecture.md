graph TD
    %% Styling
    classDef host fill:#2b2b2b,stroke:#4f4f4f,stroke-width:2px,color:#fff;
    classDef ts fill:#ff7e33,stroke:#d95d16,stroke-width:2px,color:#fff;
    classDef vm fill:#1f6feb,stroke:#0d57c6,stroke-width:2px,color:#fff;
    classDef app fill:#8e44ad,stroke:#732d91,stroke-width:2px,color:#fff;
    classDef disk fill:#238636,stroke:#1a6528,stroke-width:2px,color:#fff;
    classDef port fill:#d4af37,stroke:#aa841c,stroke-width:2px,color:#000;

    %% Nodes
    Host[Host Machine]:::host
    TS((Tailscale VPN)):::ts
    
    subgraph Debian [Debian VM - Non-Graphical]
        VM[OS Environment]:::vm
        
        subgraph Ports [Tailscale Interface Only]
            Port22[Port 22: SSH]:::port
            Port8096[Port 8096: Jellyfin]:::port
         Downs ~~~ Port22
        end
        
        JF[Jellyfin Native Service]:::app
    end
    
    subgraph Storage [Virtual Disks]
        DiskA[(Disk A: /dev/sda<br/>~20 GB)]:::disk
        DiskB[(Disk B: /dev/sdb<br/>196 GB)]:::disk
    end

    %% Connections
    Host -->|Encrypted Tunnel| TS
    TS --> Port22
    TS --> Port8096
    
    Port22 -->|Remote Management| VM
    Port8096 -->|Web UI & Streaming| JF
    
    %% Storage Mapping
    DiskA -.->|/ root<br/>/var logs<br/>/srv system data| VM
    DiskB -.->|/mnt/mediadisk<br/>Media Libraries| JF
