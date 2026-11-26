# Current System Problems
Tout tourne sur un POD ce qui est déconseillé, si le POD plante ou une mise a jour est réaliser tout le service sont bloqué

on ne pourrait pas monter en charge en collant l'API et la base de donnée dans le même POD car lorsque on voudrais faire un nouveau pods il faudra redupliquer la base de donnée 

il y aura de la perte de donnée car le POD ecrit dans la couche éphémère du conteneur, tout disparait a chaque recréation ou redemarage du POD

il n'y a pas de ressource qui sont limité dans le conteneur qui a l'API et la Base de Donnée. Si il y a un pic de trafic ou que la mémoire de l'API fuite, le conteneur peut etre tuer et donc faire tomber la base de donnée avec lui. 

pas de sonde de préparation: sans ceci même si il y est plantée il peut qand même recevoir des donnée rien est bloquée, de plus il ne le redemare pas automatiquement

sans la sonde de vitalité même si le POD reste bloquer sur Running sans jamais etre redemarée

Aucune procédure testée de sauvegarde/restauration : Sans sauvegarde stockée à l'extérieur du nœud (par exemple sur un stockage objet type S3 ou un autre serveur), impossible de récupérer les données après un crash disque, un bug applicatif ou une erreur humaine . une sauvegarde qu'on n'a jamais essayé de restaurer peut être corrompue ou inutilisable le jour de l'incident.

Les identifiants de base de données sont des valeurs d’environnement littérales engagées dans le manifeste : Le probleme est que les mot de passe sont ecrit en claire dans le fichier YAML, cela expose des donnée 

tout tourne sur une VM sans sauvegarde sur d'autre support 

Stratégie de mise à jour `Recreate` : L'ancien Pod est détruit avant la création du nouveau, entraînant une coupure de service obligatoire à chaque déploiement.

---

# Production Architecture

### Toutes les solutions appliquées
1. **Séparer l'API et la base de données :** Découpler les deux composants en créant un `Deployment` dédié pour l'API (`quote-app`) et un `Deployment` (ou `StatefulSet`) séparé pour PostgreSQL (`quote-db`), chacun exposé par son propre `Service`.
2. **Assurer la scalabilité et la haute disponibilité (Multiples réplicas) :** Déployer 3 réplicas de l'API `quote-app` derrière le `Service` Kubernetes pour absorber la charge et tolérer la perte d'un Pod.
3. **Ajouter des sondes de préparation et de vitalité (`readinessProbe` & `livenessProbe`) :** Configurer une `readinessProbe` (`/health` pour l'API, `pg_isready` pour PostgreSQL) afin de bloquer le trafic vers un Pod non prêt, et une `livenessProbe` pour redémarrer automatiquement un conteneur bloqué.
4. **Encadrer les ressources (`requests` et `limits`) :** Définir des limites CPU et RAM sur chaque conteneur pour empêcher une fuite mémoire ou un pic de trafic de faire tomber les autres composants.
5. **Garantir la persistance des données (PVC) :** Attacher un `PersistentVolumeClaim` (`quote-db-data`) au Pod PostgreSQL pour sortir les données de la couche éphémère du conteneur.
6. **Sécuriser les mises à jour (`RollingUpdate`) :** Remplacer la stratégie `Recreate` par `RollingUpdate` sur le `Deployment` de l'API pour déployer les nouvelles versions progressivement sans coupure de service.
7. **Protéger les identifiants (`Secret` hors Git) :** Supprimer les mots de passe en clair des fichiers YAML et référencer un `Secret` Kubernetes (`secretKeyRef`) provisionné hors de Git (manuellement en lab, ou via un gestionnaire de secrets type Vault en production).
8. **Sécuriser l'exposition externe (`Ingress` HTTPS) :** Remplacer le simple `NodePort` par un `Ingress` / `LoadBalancer` en HTTPS devant le `Service` `quote-app`.
9. **Éliminer le point de défaillance unique du nœud (Cluster Multi-Nœuds) :** Déployer Kubernetes sur plusieurs nœuds (VMs/serveurs) afin que l'application continue de tourner si une machine tombe en panne.
10. **Découpler le stockage et automatiser les sauvegardes externes :** Utiliser un stockage réseau/répliqué indépendant du disque local de la VM et mettre en place des sauvegardes externes automatisées (`pg_dump` ou snapshots vers S3/serveur distant) avec des tests de restauration réguliers.

### Schéma de l'architecture proposée (avec toutes les solutions)

```mermaid
flowchart TD
    Users(("Utilisateurs")) -->|"HTTPS"| Ingress["Ingress / Load Balancer<br/>[Proposé Prod]"]
    Ingress --> SvcApp("Service: quote-app (ClusterIP)<br/>[Vérifié Lab 52]")

    subgraph ClusterK8s ["Cluster Kubernetes Multi-Nœuds [Proposé Prod - Fin du SPOF Nœud]"]
        subgraph Node1 ["Nœud 1 (VM / Serveur)"]
            PodApp1["Pod: quote-app #1 (API séparée)<br/>• RollingUpdate + Requests/Limits<br/>• Readiness (/health) + Liveness<br/>[Vérifié Lab 42/52 + Proposé]"]
            PodApp2["Pod: quote-app #2 (API séparée)<br/>• RollingUpdate + Requests/Limits<br/>• Readiness (/health) + Liveness<br/>[Vérifié Lab 42/52 + Proposé]"]
        end

        subgraph Node2 ["Nœud 2 (VM / Serveur)"]
            PodApp3["Pod: quote-app #3 (API séparée)<br/>• RollingUpdate + Requests/Limits<br/>• Readiness (/health) + Liveness<br/>[Vérifié Lab 42/52 + Proposé]"]
            SvcDB("Service: quote-db (ClusterIP)<br/>[Vérifié Lab 52]")
            PodDB[("Pod: PostgreSQL 18 (DB séparée)<br/>• Requests/Limits + pg_isready<br/>• Liveness Probe<br/>[Vérifié Lab 52 + Proposé]")]
        end

        Secret[["Secret: quote-db-secret<br/>Provisionné hors Git / Vault<br/>[Vérifié Lab 52 + Proposé]"]]
        PVC[["PVC: quote-db-data<br/>Hors couche éphémère<br/>[Vérifié Lab 52]"]]
    end

    SvcApp -->|"Load Balancing (3 Réplicas)"| PodApp1
    SvcApp -->|"Load Balancing (3 Réplicas)"| PodApp2
    SvcApp -->|"Load Balancing (3 Réplicas)"| PodApp3

    PodApp1 -->|"DATABASE_URL"| SvcDB
    PodApp2 -->|"DATABASE_URL"| SvcDB
    PodApp3 -->|"DATABASE_URL"| SvcDB

    SvcDB --> PodDB
    Secret -.->|"secretKeyRef"| PodApp1
    Secret -.->|"secretKeyRef"| PodApp2
    Secret -.->|"secretKeyRef"| PodApp3
    Secret -.->|"secretKeyRef"| PodDB

    PodDB -->|"Volume Mount"| PVC
    PVC -.->|"Découplé du disque local"| StorageNet[("Stockage Réseau / Répliqué<br/>[Proposé Prod]")]
    StorageNet -.->|"pg_dump / Snapshots testés"| BackupExt[("Backups Externes (S3 / Off-site)<br/>[Proposé Prod]")]