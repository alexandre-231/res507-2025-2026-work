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

 Stratégie de mise à jour `Recreate` | L'ancien Pod est détruit avant la création du nouveau, entraînant une coupure de service obligatoire à chaque déploiement. | 
 


# Production Architecture

 un pod pour API et un pod pour la base de donnée 
 realiser une sonde de préparation et une sonde de vitalité
 Assurer la scalabilité et la haute disponibilité : Déployer plusieurs réplicas de l'API (ex. 3 réplicas) derrière le Service Kubernetes pour absorber la charge et tolérer la perte d'un Pod.


Garantir la persistance des données : Attacher un PersistentVolumeClaim (PVC) au Pod PostgreSQL pour sortir les données de la couche éphémère du conteneur.


 Encadrer les ressources : Définir des requests et limits (CPU et RAM) sur chaque conteneur pour empêcher une fuite mémoire ou un pic de trafic de faire tomber les autres composants.

 Sécuriser les mises à jour : Remplacer la stratégie Recreate par RollingUpdate pour deployer une mise a jour sans tout couper 

Protéger les identifiants : Supprimer les mots de passe en clair des fichiers YAML et référencer un Secret Kubernetes (secretRef / secretKeyRef) provisionné hors de Git

realiser des back up de la base 

### Schéma de l'architecture proposée

```mermaid
flowchart TD
    Users((Utilisateurs)) -->|HTTPS| Ingress[Ingress / Load Balancer - Proposé]
    Ingress --> SvcApp(Service: quote-app - Vérifié Lab 52)

    subgraph ClusterK8s [Cluster Kubernetes Multi-Nœuds - Proposé]
        subgraph Node1 [Nœud 1 - VM / Serveur]
            PodApp1(Pod: quote-app #1<br/>Probes + Limits)
            PodApp2(Pod: quote-app #2<br/>Probes + Limits)
        end

        subgraph Node2 [Nœud 2 - VM / Serveur]
            PodApp3(Pod: quote-app #3<br/>Probes + Limits)
            SvcDB(Service: quote-db - Vérifié Lab 52)
            PodDB[(Pod: postgres:18<br/>Vérifié Lab 52)]
        end

        Secret[[Secret: quote-db-secret<br/>Provisionné hors Git]]
        PVC[[PVC: quote-db-data<br/>Vérifié Lab 52]]
    end

    SvcApp -->|Load Balancing| PodApp1
    SvcApp -->|Load Balancing| PodApp2
    SvcApp -->|Load Balancing| PodApp3

    PodApp1 -->|DATABASE_URL| SvcDB
    PodApp2 -->|DATABASE_URL| SvcDB
    PodApp3 -->|DATABASE_URL| SvcDB

    SvcDB --> PodDB
    Secret -.->|Injection| PodApp1
    Secret -.->|Injection| PodDB

    PodDB -->|Volume Mount| PVC
    PVC -.->|Stockage Réseau / Répliqué - Proposé| StorageNet[(Stockage Externe)]
    StorageNet -.->|Sauvegardes automatiques testées - Proposé| BackupExt[(Stockage Backup Externe / S3)]