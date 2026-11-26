# Vérification et Diagnostic — Lab 42 (Kubernetes)

## 1. État du déploiement et des services
- **Namespace :** `quote-lab`
- **Image importée dans k3s :** `docker.io/library/quote-app:local`
- **Réplicas :** 3/3 Pods `Ready` (`deployment/quote-app`)
- **Service :** `quote-app` (`ClusterIP`, port `80` -> `targetPort: 3000`)

## 2. Validation des chemins réseau (`/health`)
- **Via port-forward (VM -> Service) :**
  - Commande : `curl http://127.0.0.1:8080/health`
  - Résultat attendu : `{"ok":true}`
- **Via le chemin interne au cluster (DNS Kubernetes) :**
  - Commande : `kubectl exec service-check -- wget -qO- http://quote-app/health`
  - Résultat attendu : `{"ok":true}`

## 3. Résilience et réconciliation (Remplacement de Pod)
- **Action :** Suppression manuelle d'un Pod `quote-app` (`kubectl delete pod <ANCIEN_POD>`).
- **Observation :** Le `ReplicaSet` a immédiatement recréé un nouveau Pod avec un nom différent pour maintenir l'état désiré de 3 réplicas.

## 4. Diagnostic de panne volontaire (Readiness Probe)
- **Symptôme :** Lors du déploiement, les nouveaux Pods restent à l'état `Running` mais `0/1 Ready`, et le `rollout` finit en timeout. Les anciens Pods sains continuent de servir le trafic.
- **Cause :** La sonde de disponibilité (`readinessProbe`) a été modifiée vers un chemin inexistant (`/does-not-exist`), provoquant un événement `Unhealthy` avec une erreur **HTTP 404**. Comme il s'agit d'une *readinessProbe* (et non d'une *livenessProbe*), Kubernetes retire le Pod des endpoints du Service sans redémarrer le conteneur.
- **Correction (Fix) :** Restauration du chemin `/health` dans `labs/42-kubernetes/deployment.yaml` puis réapplication (`kubectl apply -f deployment.yaml`). Le rollout s'est terminé avec 3/3 Pods `Ready`.

## 5. Extension validée
- **Lane choisie :** Base de données PostgreSQL avec stockage persistant (PVC).
- **Preuves détaillées :** Voir [les notes d'architecture et tests de persistance du Lab 52](../52-architecture/architecture-notes.md).
