# Lab 4.2 : Maîtriser les ressources Kubernetes (K3s)

## 📖 Description du Lab
Ce laboratoire explore la gestion des ressources (CPU et Mémoire) dans Kubernetes. L'objectif est de comprendre la différence entre les `requests` (réservations pour le scheduler) et les `limits` (plafonds d'exécution), de diagnostiquer des états de panne (`Pending`, `OOMKilled`, `Throttling`), et de sécuriser un namespace avec des `ResourceQuota` et des `LimitRange`.

## 📂 Fichiers du projet
- `app.yaml` : Déploiement Nginx avec requests et limits déclarées.
- `gourmand.yaml` : Déploiement demandant 8Gi de RAM (provoque un état `Pending`).
- `oom.yaml` : Pod consommant plus que sa limite mémoire (provoque un `OOMKilled`).
- `cpu.yaml` : Deux Pods avec boucle infinie (démontre le CPU Throttling).
- `quota.yaml` : ResourceQuota pour limiter un namespace.
- `limitrange.yaml` : LimitRange pour injecter des valeurs par défaut.

---

## 🧠 Questions de réflexion (Réponses)

**1. Pourquoi un Pod avec `requests: memory: 8Gi` reste-t-il `Pending` alors que le conteneur ne consommerait pas autant ?**
> Le scheduler (planificateur) de Kubernetes se base sur les `requests` (la réservation), pas sur la consommation réelle. Aucun des nœuds du cluster ne possède 8 Gi de mémoire non réservée, donc le Pod n'est placé sur aucun nœud et reste en attente (`Pending`).

**2. Quelle différence de comportement entre le dépassement d'une limite de CPU et celui d'une limite de mémoire ?**
> Le CPU est une ressource "compressible" : si un conteneur dépasse sa limite CPU, il est ralenti (throttling) mais continue de tourner. La mémoire n'est pas compressible : si un conteneur dépasse sa limite mémoire, le noyau Linux le tue immédiatement (OOMKilled), puis Kubernetes le redémarre.

**3. Où lire la cause d'un OOMKilled ?**
> Dans la sortie de `kubectl describe pod`, section `Last State`. On y lit `Reason: OOMKilled` et `Exit Code: 137`. Les logs (`kubectl logs`) ne sont d'aucune aide ici, car le conteneur est tué brutalement avant de pouvoir écrire la cause de son crash.

**4. Pourquoi le Pod test a-t-il été refusé avant l'ajout du LimitRange ?**
> Parce qu'un `ResourceQuota` portant sur le CPU et la mémoire exige que *chaque* Pod déclare explicitement ses `requests` et `limits`. Le `LimitRange` résout ce problème en fournissant automatiquement ces valeurs par défaut si elles sont absentes du manifeste.

**5. Pourquoi le Deployment `charge` a-t-il été accepté alors que ses Pods ne le sont pas ?**
> Le Deployment ne crée pas les Pods lui-même : c'est son ReplicaSet qui s'en charge. Le quota refuse la création des Pods au niveau du ReplicaSet, d'où l'erreur visible dans les événements du ReplicaSet (`kubectl describe replicaset`) et non lors de la création du Deployment.

---

## 🚀 Commandes clés et manipulations

### 1. Diagnostiquer un Pod Pending (Manque de RAM)
```bash
kubectl apply -f gourmand.yaml
kubectl get pods -l app=gourmand
# Lire l'erreur "Insufficient memory" :
kubectl describe pod -l app=gourmand | tail -n 6
2. Provoquer et diagnostiquer un OOMKilled (Mémoire dépassée)
bash
kubectl apply -f oom.yaml
kubectl get pod oom -w
# Chercher la cause du crash :
kubectl describe pod oom | grep -A6 "Last State"
3. Observer le CPU Throttling
bash
kubectl apply -f cpu.yaml
sleep 60
kubectl top pods
# Observer que cpu-limite reste à ~100m alors que cpu-libre monte à ~1000m
kubectl delete -f cpu.yaml
4. Tester les Quotas et LimitRanges
bash
# Créer le namespace
kubectl create namespace tp-quota
kubectl config set-context --current --namespace=tp-quota

# Appliquer le Quota
kubectl apply -f quota.yaml

# Tenter de créer un pod sans limites (doit échouer)
kubectl run test --image=nginx:1.26-alpine

# Appliquer le LimitRange pour injecter les valeurs par défaut
kubectl apply -f limitrange.yaml
kubectl run test --image=nginx:1.26-alpine

# Vérifier l'injection automatique
kubectl get pod test -o jsonpath='{.spec.containers[0].resources}'; echo
5. Dépasser le quota
bash
kubectl create deployment charge --image=nginx:1.26-alpine --replicas=8
kubectl describe replicaset -l app=charge | tail -n 8
# Lire l'erreur "exceeded quota"
🧹 Nettoyage
bash
kubectl delete namespace tp-k8s
kubectl delete namespace tp-quota
kubectl config set-context --current --namespace=default
