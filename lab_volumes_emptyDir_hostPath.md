
# LAB : Exploration des Volumes `emptyDir` et `hostPath`

### Objectifs pédagogiques du TP

1. **Comprendre l'isolation du système de fichiers de conteneur** (disparition des données à la recréation du conteneur).
2. **Mettre en œuvre `emptyDir`** pour illustrer le partage de données entre deux conteneurs d'un même Pod (pattern Sidecar).
3. **Mettre en œuvre `hostPath`** pour comprendre l'ancrage du stockage à un nœud physique de l'infrastructure et ses limites en termes de portabilité.

---

## Partie 1 : Le système de fichiers de conteneur (Isolant et Éphémère)

Dans cette première partie, nous allons observer que la couche de stockage inscriptible d'un conteneur est liée à son cycle de vie.

1. Créez un pod simple avec un conteneur Nginx :
```bash
kubectl run test-container --image=nginx:alpine

```


2. Créez un fichier de test directement à l'intérieur du système de fichiers du conteneur :
```bash
kubectl exec -it test-container -- touch /usr/share/nginx/html/mon-fichier.txt
kubectl exec -it test-container -- sh -c "echo 'Donnees importantes' > /usr/share/nginx/html/mon-fichier.txt"

```


*Verification:* Affichez le contenu pour valider :
```bash
kubectl exec -it test-container -- cat /usr/share/nginx/html/mon-fichier.txt

```


3. **Simulez la destruction et la recréation du conteneur :**
Supprimez le pod et recréez-le à l'identique :
```bash
kubectl delete pod test-container
kubectl run test-container --image=nginx:alpine

```


4. **Testez l'accès au fichier :**
```bash
kubectl exec -it test-container -- cat /usr/share/nginx/html/mon-fichier.txt

```


* **Résultat attendu :** Une erreur `No such file or directory`. Le fichier a disparu car le système de fichiers inscriptible du conteneur est éphémère.



Nettoyez le pod :

```bash
kubectl delete pod test-container

```

---

## Partie 2 : Le volume `emptyDir` (Partage inter-conteneurs et cycle de vie du Pod)

Imaginons un cas d'usage où un conteneur génère des logs ou des fichiers, et un second conteneur (un sidecar) doit les lire ou les traiter.

1. Créez un fichier manifeste nommé `pod-emptydir.yaml` :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-emptydir
spec:
  containers:
  - name: ecrivain
    image: busybox:latest
    command: ["sh", "-c", "while true; do date >> /data/date.txt; sleep 5; done"]
    volumeMounts:
    - mountPath: /data
      name: stockage-partage
  - name: lecteur
    image: busybox:latest
    command: ["sh", "-c", "tail -f /logs/date.txt"]
    volumeMounts:
    - mountPath: /logs
      name: stockage-partage
  volumes:
  - name: stockage-partage
    emptyDir: {}

```


2. Appliquez le manifeste :
```bash
kubectl apply -f pod-emptydir.yaml

```


3. **Vérifiez le partage :**
Consultez les logs du conteneur `lecteur`. Il lit en temps réel les données écrites par le conteneur `ecrivain` via le volume partagé `emptyDir` :
```bash
kubectl logs demo-emptydir -c lecteur

```


*Verification:* Vous devez voir défiler les horodatages générés toutes les 5 secondes.
4. **Testez la résistance au redémarrage de conteneur :**
Supprimez l'un des deux conteneurs (par exemple en le tuant) :
```bash
kubectl exec demo-emptydir -c ecrivain -- kill 1

```


* **Résultat :** Le conteneur redémarre, et le fichier `/data/date.txt` est toujours accessible car le volume `emptyDir` survit au redémarrage des conteneurs *tant que le Pod lui-même existe*.



Nettoyez le pod :

```bash
kubectl delete -f pod-emptydir.yaml

```

---

## Partie 3 : Le volume `hostPath` (Ancrage à l'infrastructure)

Le volume `hostPath` monte un fichier ou un répertoire du système de fichiers du nœud hôte (Worker Node) directement à l'intérieur du Pod.

1. Créez un fichier manifeste nommé `pod-hostpath.yaml` :
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-hostpath
spec:
  containers:
  - name: worker
    image: busybox:latest
    command: ["sh", "-c", "echo 'Stocké sur le nœud physique' > /host-data/test-noeud.txt; sleep 3600"]
    volumeMounts:
    - mountPath: /host-data
      name: stock-noeud
  volumes:
  - name: stock-noeud
    hostPath:
      path: /tmp/kubernetes-hostpath  # Dossier créé sur le nœud hôte
      type: DirectoryOrCreate

```


2. Appliquez le manifeste :
```bash
kubectl apply -f pod-hostpath.yaml

```


3. **Vérification (Le piège de l'infrastructure) :**
* Si vous êtes sur un cluster KinD, vous pouvez entrer directement dans le conteneur Docker du nœud Worker (`docker exec -it <nom-du-noeud-kind> sh`) pour constater que le fichier `/tmp/kubernetes-hostpath/test-noeud.txt` a bien été écrit physiquement sur le disque de la machine hôte.
* *Que se passerait-il si ce Pod tombait en panne et était redémarré par Kubernetes sur un autre nœud du cluster ?* Le fichier ne serait plus accessible, car il est physiquement bloqué sur le disque du premier nœud. C'est l'anti-pattern par excellence pour les applications distribuées, mais c'est très utile pour des agents d'infrastructure (comme Prometheus Node Exporter ou des collecteurs de logs basés sur les fichiers `/var/log` de l'hôte).



Nettoyez le pod :

```bash
kubectl delete -f pod-hostpath.yaml

```

---