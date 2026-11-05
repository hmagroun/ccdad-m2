
# LAB : Persistent Volumes – Provisionnement Dynamique

### Objectifs du Lab

1. Explorer les StorageClasses (`SC`) existantes dans le cluster et identifier celle configurée par défaut (`standard` / `local`).
2. Comprendre comment le provisionnement dynamique élimine le besoin de créer manuellement un PersistentVolume (PV).
3. Créer un PersistentVolumeClaim (PVC) sans préciser de PV cible, et observer la création automatique du PV par la StorageClass.
4. Déployer un Pod consommateur et vérifier le bon fonctionnement de l'approvisionnement automatique.
5. Nettoyer l'environnement en supprimant proprement les ressources créées.

---

## Partie 1 : Exploration des StorageClasses (SC)

Avant de créer des volumes, examinons ce que le cluster met à notre disposition par défaut.

### 1. Lister les StorageClasses du cluster

Exécutez la commande suivante :

```bash
kubectl get storageclass

```

* **Résultat observé sur KinD :** Vous devriez voir une StorageClass (généralement nommée `standard`) marquée par une mention `(default)`. C'est elle qui intercepte toutes les demandes de PVC qui ne spécifient pas de classe particulière.

### 2. Inspecter les détails de la StorageClass par défaut

Pour comprendre comment elle fonctionne, affichez sa configuration YAML :

```bash
kubectl describe storageclass standard

```

* **Points clés à observer :**
* **Provisioner :** Indique le composant logiciel responsable de créer le stockage (sur KinD, c'est `rancher.io/local-path`).
* **ReclaimPolicy :** Généralement fixée à `Delete` par défaut (le PV et le stockage physique sont supprimés lorsque le PVC est supprimé).
* **VolumeBindingMode :** Souvent réglé sur `WaitForFirstConsumer` (le volume n'est réellement créé/lié que lorsqu'un Pod a réellement besoin de l'utiliser).



---

## Partie 2 : Création d'un PVC dynamique (sans PV préalable)

Contrairement au provisionnement manuel, **nous ne créons aucun objet `PersistentVolume` (PV)**. C'est la magie du provisionnement dynamique : le PVC suffit.

### 1. Création du fichier `pvc-dynamique.yaml`

Créez un manifeste pour votre réclamation de stockage :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dyn
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  # Remarque : On ne précise ni storageClassName ni volumeName. 
  # Kubernetes utilise automatiquement la StorageClass par défaut (standard).

```

### 2. Application du PVC et observation immédiate

Appliquez le PVC :

```bash
kubectl apply -f pvc-dynamique.yaml

```

Vérifiez l'état de vos PVC et PV :

```bash
kubectl get pvc,pv

```

* **Ce qu'il faut observer :** Le PVC est créé, mais en mode `WaitForFirstConsumer`, il reste en attente (`Pending`) tant qu'aucun Pod ne l'utilise.

---

## Partie 3 : Déploiement du Pod et vérification de l'approvisionnement

Déployons maintenant un Pod qui utilise ce `pvc-dyn` pour déclencher l'approvisionnement automatique.

### 1. Création du fichier `pod-dynamique.yaml`

Créez le manifeste du Pod :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-dyn
spec:
  containers:
  - name: app
    image: busybox:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - mountPath: /data
      name: stock-dynamique
  volumes:
  - name: stock-dynamique
    persistentVolumeClaim:
      claimName: pvc-dyn

```

### 2. Déploiement et vérification magique

Appliquez le Pod :

```bash
kubectl apply -f pod-dynamique.yaml

```

Immédiatement après, observez le résultat :

```bash
kubectl get pvc,pv

```

* **Résultat magique :**
1. Le PVC passe à l'état **`Bound`**.
2. Un **`PersistentVolume` (PV)** a été **généré automatiquement** en arrière-plan par la StorageClass.



### 3. Test d'écriture des données

Vérifiez que le stockage fonctionne correctement :

```bash
kubectl exec -it pod-dyn -- sh -c "echo 'Test de provisionnement dynamique OK' > /data/dyn.txt && cat /data/dyn.txt"

```

---

## Partie 4 : Nettoyage de l'environnement

Pour terminer proprement le lab et libérer les ressources du cluster, supprimons le Pod et le PVC.

1. **Supprimez le Pod :**
```bash
kubectl delete pod pod-dyn

```


2. **Supprimez le PVC :**
```bash
kubectl delete pvc pvc-dyn

```


3. **Vérification du nettoyage :**
```bash
kubectl get pvc,pv

```


*Résultat :* Grâce à la politique `Delete` par défaut de la StorageClass, le PVC, le PV généré automatiquement et l'espace de stockage physique ont tous été supprimés proprement.

---