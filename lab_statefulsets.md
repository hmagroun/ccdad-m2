

# LAB : Déploiement (Stateless) vs StatefulSet (Stateful)

### Objectifs du Lab

1. **Partie 1 :** Mettre en évidence la différence d'identité, de nommage et d'ordre de démarrage/arrêt entre **app1** (Deployment / Stateless) et **app2** (StatefulSet / Stateful) avec Nginx et leurs services dédiés (**service1** et **service2**).
2. **Partie 2 :** Étudier la gestion du stockage persistant, illustrer la limite du partage d'un volume RWO sur un Deployment, et configurer le stockage dédié par réplique pour **app2** via les `volumeClaimTemplates`.
3. **Partie 3 :** Auto-inspection et découverte réseau : se connecter directement à l'intérieur de la première instance (`app2-0`) pour vérifier son identité et exécuter les commandes `nslookup` (via les FQDNs complets) vers le Headless Service et les autres pairs.

---

## Partie 1 : Ordre de démarrage/arrêt, identité et services réseau

### 1. Application Stateless (`app1`) avec son Service classique (`service1`)

Créez le fichier `app1.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service1
spec:
  ports:
  - port: 80
    name: web
  selector:
    app: app1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: nginx
        image: nginx:alpine

```

Appliquez et observez :

```bash
kubectl apply -f app1.yaml
kubectl get pods -l app=app1 -w

```

* **Observation :** Les pods démarrent en parallèle avec des noms aléatoires.

---

### 2. Application Stateful (`app2`) avec son Headless Service (`service2`)

Créez le fichier `app2.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service2
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None  # <-- Headless Service indispensable pour le StatefulSet
  selector:
    app: app2
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: app2
spec:
  serviceName: "service2"
  replicas: 3
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: nginx
        image: nginx:alpine

```

Appliquez et observez le démarrage séquentiel strict (`app2-0`, puis `app2-1`, puis `app2-2`) :

```bash
kubectl apply -f app2.yaml
kubectl get pods -l app=app2 -w

```

Nettoyez ces premiers manifestes avant de passer à la gestion du stockage :

```bash
kubectl delete -f app1.yaml
kubectl delete -f app2.yaml

```

---

## Partie 2 : Gestion du stockage persistant (Deployment vs StatefulSet)

### 1. La limite du Deployment avec un volume persistant partagé

Si vous associez un PVC classique en mode **`ReadWriteOnce` (RWO)** à un Deployment d'**app1** comportant plusieurs répliques (Nginx), vous faites face à une impasse technique :

* Soit les pods se bloquent avec une erreur **`Multi-Attach error`** car un stockage RWO ne peut pas être monté en écriture par plusieurs pods simultanément.
* Soit, si l'infrastructure le permettait, toutes les répliques pointeraient vers le même dossier et s'écraseraient mutuellement sans isolation. Un Deployment n'est pas conçu pour attribuer un disque indépendant par réplique.

---

### 2. Le StatefulSet et les `volumeClaimTemplates` pour `app2`

Avec le StatefulSet **app2**, nous utilisons les `volumeClaimTemplates`. Kubernetes génère automatiquement **un PVC individuel et dédié pour chaque réplique** (`data-app2-0`, `data-app2-1`, `data-app2-2`).

Créez le fichier `app2-storage.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: service2
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: app2
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: app2
spec:
  serviceName: "service2"
  replicas: 3
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts:
        - name: data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 100Mi

```

Appliquez le fichier :

```bash
kubectl apply -f app2-storage.yaml

```

Vérifiez les PVC générés automatiquement :

```bash
kubectl get pvc

```

* **Observation :** 3 PVC distincts sont créés (`data-app2-0`, `data-app2-1`, `data-app2-2`).

Vous pouvez personnaliser la page web lue par chaque réplique Nginx pour prouver leur isolation :

```bash
kubectl exec app2-0 -- sh -c "echo 'Bienvenue sur App2 Instance 0' > /usr/share/nginx/html/index.html"
kubectl exec app2-1 -- sh -c "echo 'Bienvenue sur App2 Instance 1' > /usr/share/nginx/html/index.html"
kubectl exec app2-2 -- sh -c "echo 'Bienvenue sur App2 Instance 2' > /usr/share/nginx/html/index.html"

```

---

## Partie 3 : Auto-inspection et découverte des pairs depuis l'intérieur d'une instance

Connectons-nous directement à la première instance de l'application (`app2-0`) pour inspecter son identité et exécuter les commandes de résolution réseau (via les **FQDNs complets**).

### 1. Se connecter au premier pod du StatefulSet

```bash
kubectl exec -it app2-0 -- sh

```

### 2. Reconnaître son propre nom d'hôte (`hostname`)

Une fois à l'intérieur du conteneur, vérifiez l'identité de l'instance :

```bash
hostname

```

* **Résultat :** Affiche `app2-0`, prouvant que l'instance connaît son propre indice de manière statique et prévisible.

### 3. Interroger le Headless Service et les pairs avec `nslookup` (via FQDN)

Depuis l'intérieur de ce même pod, testez la résolution globale du Headless Service :

```bash
nslookup service2.default.svc.cluster.local

```

*(Cette commande renvoie la liste complète des adresses IP de toutes les répliques associées).*

Testez ensuite la résolution individuelle de chaque pair en utilisant son FQDN complet :

```bash
nslookup app2-0.service2.default.svc.cluster.local
nslookup app2-1.service2.default.svc.cluster.local
nslookup app2-2.service2.default.svc.cluster.local

```

* **Résultat :** Chaque réplique résout correctement sa propre adresse IP ainsi que celles de ses congénères.

Quittez le conteneur :

```bash
exit

```

---

## Nettoyage de l'environnement

```bash
kubectl delete -f app2-storage.yaml
kubectl delete pvc -l app=app2

```