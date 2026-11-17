cat << 'EOF' > /home/alex/res507-2025-2026-work/labs/52-architecture/architecture-notes.md
# Architecture Notes - Lab 52 : Recovery & Persistent Data

## 1. Schéma de l'infrastructure (Actual System)

```mermaid
flowchart TD
    User((Navigateur PC)) -->|Tunnel / Port-forward 8080:80| SvcApp(Service: quote-app)
    
    subgraph Nœud Kubernetes [VM Ubuntu - Noyau Linux partagé]
        SvcApp -->|Load Balancing| PodApp1(Pod: quote-app)
        SvcApp -->|Load Balancing| PodApp2(Pod: quote-app)
        SvcApp -->|Load Balancing| PodApp3(Pod: quote-app)
        
        PodApp1 -->|DATABASE_URL| SvcDB(Service: quote-db)
        PodApp2 -->|DATABASE_URL| SvcDB(Service: quote-db)
        PodApp3 -->|DATABASE_URL| SvcDB(Service: quote-db)
        
        SvcDB --> PodDB[(Pod: postgres:18)]
        
        PodDB -->|Volume Mount| PVC[[PVC: quote-db-data]]
        PVC -.->|StorageClass: local-path| Disk[Disque dur de la VM]
    end
    
    Nœud Kubernetes --- Hypervisor[Hyperviseur]
    Hypervisor --- Hardware[Matériel Physique Hôte]