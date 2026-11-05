
# LAB : Persistent Volumes – Provisionnement manuel

### Objectifs du Lab

1. Créer un PersistentVolume (PV) et un PersistentVolumeClaim (PVC) en mode **manuel strict** (sans passer par une StorageClass automatique).
2. Lier directement un PVC à un PV ciblé en utilisant `volumeName`.
3. Comparer les deux politiques de récupération : **`Delete`** (suppression automatique) et **`Retain`** (conservation).
4. Apprendre à réexploiter proprement un volume en état `Released` en purgeant son champ `claimRef`, puis en le liant à un nouveau PVC (`pvc22`) et un nouveau Pod (`pod22`) pour vérifier la persistance des données.

---

## Partie 1 : La politique de suppression automatique (`Delete`)

Dans cette première partie, nous utilisons la politique `Delete`. Lorsque le PVC est supprimé, le PV associé est automatiquement supprimé par Kubernetes.

### 1. Création du PersistentVolume (`pv1`)

Créez un fichier manifeste `pv1.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete  # <-- Supprime pv1 lorsque pvc1 est supprimé
  hostPath:
    path: /tmp/data-delete-1

```

Appliquez le PV et vérifiez qu'il est `Available` :

```bash
kubectl apply -f pv1.yaml
kubectl get pv

```

### 2. Création du PersistentVolumeClaim (`pvc1`) avec liaison directe

Pour désactiver le provisionnement automatique de la StorageClass par défaut, on fixe `storageClassName: ""` et on cible `pv1` via `volumeName`.

Créez le fichier `pvc1.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""
  volumeName: pv1

```

Appliquez le PVC et vérifiez la liaison (`Bound`) :

```bash
kubectl apply -f pvc1.yaml
kubectl get pv,pvc

```

### 3. Déploiement du Pod (`pod1`) et test de suppression

Créez le fichier `pod1.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: app
    image: busybox:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - mountPath: /data
      name: stock
  volumes:
  - name: stock
    persistentVolumeClaim:
      claimName: pvc1

```

Appliquez le Pod :

```bash
kubectl apply -f pod1.yaml

```

* **Test de la politique `Delete` :**
1. Supprimez d'abord le Pod, puis le PVC :
```bash
kubectl delete pod pod1
kubectl delete pvc pvc1

```


2. **Vérification :**
```bash
kubectl get pv

```


*Résultat :* `pv1` a été automatiquement supprimé du cluster par Kubernetes.



---

## Partie 2 : La politique de rétention (`Retain`)

Dans cette seconde partie, nous utilisons la politique `Retain` pour conserver les données et l'objet PV même après la suppression du PVC.

### 1. Création du PersistentVolume (`pv2`) et du PVC (`pvc2`)

Créez un fichier `pv2.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv2
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain  # <-- Conserve pv2 et les données à la libération
  hostPath:
    path: /tmp/data-retain-2

```

Appliquez `pv2.yaml` :

```bash
kubectl apply -f pv2.yaml

```

Créez le fichier de réclamation `pvc2.yaml` :

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc2
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""
  volumeName: pv2

```

Appliquez `pvc2.yaml` et vérifiez qu'ils sont liés (`Bound`) :

```bash
kubectl apply -f pvc2.yaml
kubectl get pv,pvc

```

### 2. Écriture de données via le pod `pod2`

Créez un fichier `pod2.yaml` pour monter `pvc2` et écrire des données persistantes :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod2
spec:
  containers:
  - name: app
    image: busybox:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - mountPath: /data
      name: stock
  volumes:
  - name: stock
    persistentVolumeClaim:
      claimName: pvc2

```

Appliquez le pod et écrivez le fichier de test :

```bash
kubectl apply -f pod2.yaml
kubectl exec -it pod2 -- sh -c "echo 'Donnees critiques a conserver' > /data/info.txt"

```

### 3. Suppression du Pod et du PVC, puis observation de l'état `Released`

Supprimez `pod2` et `pvc2` :

```bash
kubectl delete pod pod2
kubectl delete pvc pvc2

```

Vérifiez l'état du PV :

```bash
kubectl get pv

```

*Résultat :* `pv2` n'est pas supprimé, mais il passe à l'état **`Released`** car il conserve l'ancien lien (`claimRef`).

---

## Partie 3 : Réexploitation d'un PV en état `Released` et vérification des données

Pour réutiliser ce volume et prouver que les données n'ont pas été perdues :

1. **Supprimer l'ancien `claimRef` :**
Modifiez le PV en direct :
```bash
kubectl edit pv pv2

```


Dans l'éditeur, recherchez la section `spec:` et **supprimez entièrement le sous-bloc `claimRef:**`. Sauvegardez et quittez.
2. **Vérification du statut :**
```bash
kubectl get pv

```


*Résultat :* `pv2` repasse à l'état **`Available`**.
3. **Liaison à un nouveau PVC (`pvc22`) :**
Créez un nouveau fichier `pvc22.yaml` pointant vers ce même PV :
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc22
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ""
  volumeName: pv2

```


Appliquez-le :
```bash
kubectl apply -f pvc22.yaml

```


4. **Déploiement de `pod22` et récupération des données :**
Créez un fichier `pod22.yaml` utilisant ce nouveau `pvc22` :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod22
spec:
  containers:
  - name: app
    image: busybox:latest
    command: ["sleep", "3600"]
    volumeMounts:
    - mountPath: /data
      name: stock
  volumes:
  - name: stock
    persistentVolumeClaim:
      claimName: pvc22

```


Appliquez `pod22.yaml` :
```bash
kubectl apply -f pod22.yaml

```


Vérifiez que le fichier initial est bien présent et intact :
```bash
kubectl exec -it pod22 -- cat /data/info.txt

```


*Résultat affiché :* `Donnees critiques a conserver` (la preuve que la réexploitation et la persistance fonctionnent parfaitement).

---