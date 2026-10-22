Ce tutoriel technique présente l'utilisation des **ConfigMaps**, des **Secrets** et des **Volumes** dans Kubernetes pour gérer la configuration.

---

## 1. Présentation des ressources de configuration

Dans Kubernetes, la philosophie Cloud Native exige de découpler le code de l'application de sa configuration. C'est le rôle des ConfigMaps et des Secrets.

### A. ConfigMap

Un **ConfigMap** est un objet API qui permet de stocker des paires clé-valeur (ou des fichiers de configuration complets) sous forme de données non sensibles.

* **Cas d'usage :** URL de bases de données, paramètres d'environnement, drapeaux fonctionnels (*feature flags*), ou fichiers de configuration (`nginx.conf`, `application.properties`).
* **Sécurité :** Stocké en clair dans etcd (sauf si un chiffrement au repos est activé au niveau du cluster).

### B. Secret

Un **Secret** remplit la même fonction qu'un ConfigMap, mais est spécifiquement conçu pour stocker des données sensibles.

* **Cas d'usage :** Mots de passe, tokens d'API, clés privées SSH, certificats TLS.
* **Sécurité :** Les valeurs des Secrets sont encodées en **Base64** dans les manifestes YAML (ce qui n'est **pas** un chiffrement, mais une protection contre la lecture accidentelle). Les clusters de production modernes activent systématiquement le chiffrement des Secrets dans etcd.

---

## 2. Le concept de Volume et son rôle dans la configuration

### A. Qu'est-ce qu'un volume Kubernetes ?

Dans Docker, un volume est un répertoire sur l'hôte ou dans un conteneur. Dans Kubernetes, le concept est plus large et **lié au cycle de vie du Pod**, et non à un conteneur individuel.

* Un volume existe aussi longtemps que le Pod existe. Si le Pod est supprimé, le volume éphémère disparaît (contrairement aux Persistent Volumes).
* Il peut être partagé simultanément entre plusieurs conteneurs d'un même Pod.

### B. Utilisations possibles des volumes d'une manière générale

Les volumes Kubernetes couvrent de nombreux cas d'usage :

1. **Partage de données temporaires :** `emptyDir` pour échanger des fichiers entre conteneurs d'un même Pod (ex: un conteneur de logs et un conteneur sidecar d'export).
2. **Accès au système de fichiers de l'hôte :** `hostPath` pour monter un répertoire du nœud worker (réservé à des usages spécifiques comme le monitoring de nœud).
3. **Stockage persistant :** `PersistentVolume` (PV) / `PersistentVolumeClaim` (PVC) pour l'attachement de stockage réseau (NFS, Ceph, EBS, etc.) garantissant la persistance des données d'une base de données par exemple.
4. **Injection de configuration :** Utilisation des volumes `configMap` et `secret` pour injecter des fichiers de configuration directement dans le système de fichiers du conteneur.

---

## 3. Les deux modes d'injection : Variables d'environnement vs Volumes

Pour exploiter ConfigMaps et Secrets dans un Pod, vous disposez de deux méthodes principales :

| Critère | Variables d'environnement (`env` / `envFrom`) | Montage en Volume (`volumeMounts`) |
| --- | --- | --- |
| **Format** | Clé-valeur injectée dans l'environnement du processus. | Fichiers physiques créés dans un répertoire du conteneur. |
| **Cas d'idéal** | Paramètres simples, chaînes de connexion courtes, flags. | Fichiers de configuration complexes (`.json`, `.yaml`, `.conf`) ou certificats. |
| **Rechargement à chaud** | **Non** (nécessite un redémarrage du Pod si la valeur change). | **Oui** (les fichiers montés se mettent à jour automatiquement lorsque le ConfigMap/Secret change, selon la mise en cache de l'application). |

---

## 4. Tutoriel Pratique : Mise en œuvre pas à pas

Ce laboratoire met en place un Pod Nginx qui consomme un `ConfigMap` (pour un fichier de configuration HTML/personnalisé) et un `Secret` (pour une clé API simulée).

### Étape 1 : Créer le ConfigMap et le Secret

Créez un fichier `configs-and-secrets.yaml` :

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
data:
  app_env: "production"
  welcome_message: "Bienvenue sur notre plateforme Cloud Native !"
  nginx.conf: |
    server {
        listen 80;
        server_name localhost;
        location / {
            root /usr/share/nginx/html;
            index index.html index.htm;
        }
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: web-app-secret
type: Opaque
stringData:
  api_token: "super-secret-token-12345"

```

Appliquez le manifeste :

```bash
kubectl apply -f configs-and-secrets.yaml

```

---

### Étape 2 : Créer le Pod consommant ces ressources

Ce manifeste de Pod illustre les **deux approches** :

1. Injection de variables d'environnement (`app_env` et `api_token`).
2. Montage d'un volume pour le fichier de configuration Nginx et les variables.

Créez le fichier `pod-app.yaml` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: demo-config-pod
spec:
  containers:
  - name: nginx-container
    image: nginx:alpine
    ports:
    - containerPort: 80
    
    # 1. Injection par Variables d'Environnement
    env:
    - name: ENVIRONMENT
      valueFrom:
        configMapKeyRef:
          name: web-app-config
          key: app_env
    - name: API_SECRET_KEY
      valueFrom:
        secretKeyRef:
          name: web-app-secret
          key: api_token

    # 2. Injection par Volume
    volumeMounts:
    - name: config-volume
      mountPath: /etc/custom-configs
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true

  volumes:
  - name: config-volume
    configMap:
      name: web-app-config
  - name: secret-volume
    secret:
      secretName: web-app-secret

```

Appliquez le Pod :

```bash
kubectl apply -f pod-app.yaml

```

---

### Étape 3 : Vérification et validation

Vérifiez que le Pod est en cours d'exécution :

```bash
kubectl get pods

```

#### A. Vérification des variables d'environnement

Exécutez la commande suivante pour inspecter l'environnement du conteneur :

```bash
kubectl exec -it demo-config-pod -- env | grep -E "ENVIRONMENT|API_SECRET_KEY"

```

*Résultat attendu :*

```text
ENVIRONMENT=production
API_SECRET_KEY=super-secret-token-12345

```

#### B. Vérification des volumes montés

Inspectez le contenu du répertoire de configuration monté :

```bash
kubectl exec -it demo-config-pod -- ls -la /etc/custom-configs

```

*Résultat attendu :* Vous y verrez les fichiers `app_env`, `welcome_message` et `nginx.conf` créés automatiquement par Kubernetes à partir des clés du ConfigMap.

Affichez le contenu du fichier de configuration :

```bash
kubectl exec -it demo-config-pod -- cat /etc/custom-configs/welcome_message

```
