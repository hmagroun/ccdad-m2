# storage-lab — Lab de formation au stockage Kubernetes (KinD)

Lab destiné à des **étudiants et stagiaires qui découvrent Kubernetes** : manipuler PV, PVC et StorageClass, en provisionnement statique comme dynamique, sur deux types de stockage professionnels, en gardant la mécanique du stockage aussi discrète que possible.

| StorageClass | Type | Protocole | Mode d'accès | Driver CSI | Sur la VM |
|---|---|---|---|---|---|
| `nfs-shared` | **fichiers partagés** | NFS v4.1 | **RWX** | `csi-driver-nfs` (projet officiel Kubernetes) | `/dev/sdc` → ext4 → `/srv/nfs` |
| `nvme-block` | **bloc réseau** | NVMe/TCP | **RWO** | `democratic-csi` (ZFS + nvmet) | `/dev/sdb` → pool ZFS `tank-nvme` → zvols |

Point de départ : **une VM Ubuntu 24.04 fraîchement installée**. Point d'arrivée : un cluster KinD avec ses deux StorageClasses, des outils de provisionnement manuel et un **parcours de 6 labs**, des services **LoadBalancer** (cloud-provider-kind) et deux fichiers de cluster kind simples dans le répertoire personnel.

---

## 1. Architecture

```
┌──────────────────────── VM Ubuntu 24.04 = « serveur de stockage » ────────────────────────┐
│  /dev/sdb → ZFS tank-nvme → zvol (1 par volume) → cible NVMe/TCP (noyau, nvmet)  :4420    │
│  /dev/sdc → ext4 /srv/nfs → export NFS                                            :2049    │
│  sshd :22 ← le controller democratic-csi crée zvols et cibles (zfs, nvmetcli)              │
│                                                                                            │
│  ── réseau docker « kind » 172.18.0.0/16 — la VM y est 172.18.0.1 ──                       │
│  ┌───────────────┐  ┌──────────────────────┐  ┌──────────────────────┐                     │
│  │ control-plane │  │ worker               │  │ worker2              │  « serveurs         │
│  └───────────────┘  │ nvme-cli + nfs       │  │ nvme-cli + nfs       │   applicatifs »     │
│                     └──────────────────────┘  └──────────────────────┘                     │
└────────────────────────────────────────────────────────────────────────────────────────────┘
```

**NVMe/TCP en quelques mots** (c'est le successeur d'iSCSI, avec les mêmes concepts) :

| Concept | Terme NVMe/TCP | Équivalent iSCSI |
|---|---|---|
| Nom de la cible (le « disque réseau ») | **NQN** | IQN |
| Numéro du disque dans la cible | **namespace** (nsid) | LUN |
| Point d'accès réseau | **port :4420** | portail :3260 |
| Identité du client | **hostnqn** | IQN initiateur |
| Côté client | `nvme connect`, aucun démon | démon `iscsid` |

**Particularité de KinD :** les nœuds kind sont des conteneurs qui **partagent le noyau de la VM**. Les modules (`nvme-tcp`, `nfs`) sont donc chargés sur la VM, le `/dev` de la VM est monté dans les workers, et une connexion NVMe ouverte par un nœud est en réalité une connexion du noyau de la VM.

---

## 2. Préparer la VM VirtualBox

### 2.1 Créer la VM
| Paramètre | Valeur |
|---|---|
| Type | Linux, Ubuntu (64-bit) |
| Processeurs | 4 vCPU |
| Mémoire | 8 Go minimum (10 Go conseillés pour le lab 5, PostgreSQL) |
| Disque système | 40 Go (VDI, dynamique) |
| Réseau | NAT (suffit : seul l'accès Internet est nécessaire) |

### 2.2 Ajouter les deux disques du lab
*Configuration → Stockage → Contrôleur SATA → « Ajouter un disque dur » → « Créer »*, deux fois :

| Fichier | Taille | Port SATA | Vu dans Ubuntu comme | Rôle |
|---|---|---|---|---|
| `storage-nvme.vdi` | 20 Go, dynamique | 1 | `/dev/sdb` | bloc réseau (NVMe/TCP) |
| `storage-nfs.vdi` | 20 Go, dynamique | 2 | `/dev/sdc` | fichiers partagés (NFS) |

Le disque système reste sur le port 0 (`/dev/sda`).

### 2.3 Installer Ubuntu 24.04 Desktop
Installation standard (ou minimale). ⚠️ À l'étape « Type d'installation », installez Ubuntu **uniquement sur le disque de 40 Go** : les deux disques de 20 Go doivent rester vierges.

Après le premier démarrage, vérifiez :
```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINTS
# sdb  20G disk      ← aucune partition, aucun point de montage
# sdc  20G disk      ← idem
```
S'ils ont d'autres noms, vous les indiquerez dans `lab.env` (étape 3).

---

## 3. Récupérer le lab sur la VM

Copiez `storage-lab.tar.gz` dans la VM (téléchargement depuis le navigateur de la VM, ou dossier partagé VirtualBox), puis :
```bash
tar xzf ~/Téléchargements/storage-lab.tar.gz -C ~
cd ~/storage-lab
chmod +x *.sh tools/*.sh
```
Si vos disques ne s'appellent pas `/dev/sdb` et `/dev/sdc`, modifiez `NVME_DISK` et `NFS_DISK` dans `lab.env`. Aucune autre valeur n'a besoin d'être changée.

---

## 4. Mise en place, étape par étape

| Étape | Script | Durée indicative | Fréquence |
|---|---|---|---|
| 1 | `./00-install-tools.sh` | 5 min | une fois |
| — | **fermer la session Ubuntu et se reconnecter** | | une fois |
| 2 | `./05-storage-server.sh` | 3 min | une fois |
| 3 | `./10-build-image.sh` | 2 min | une fois par version de Kubernetes |
| 4 | `./20-create-cluster.sh` | 2 min | à chaque nouveau cluster |
| 5 | `./25-smoke-tests.sh` | 30 s | recommandé |
| 6 | `./30-install-csi.sh` | 3 min | à chaque nouveau cluster |
| 7 | `./90-test.sh` | 2 min | recommandé |
| 8 | `./40-labs.sh` | 5 s | à chaque nouveau cluster |

Tous les scripts sont **idempotents** : en cas d'erreur, corrigez puis relancez le même script.

### Étape 1 — Outils (`00-install-tools.sh`)
Met Ubuntu à jour et installe `docker` (paquet Ubuntu), `kind v0.31.0`, `kubectl v1.35.0` (somme de contrôle vérifiée), `helm 3`, `git`, `curl` et `gettext-base`. Ajoute votre utilisateur au groupe `docker`.

Dépose aussi dans le répertoire personnel **`~/cluster-1cp1w.yml`** (1 control-plane + 1 worker) et **`~/cluster-1cp2w.yml`** (1 control-plane + 2 workers) : des clusters kind « nus », pour d'autres exercices, avec les **mêmes versions** que le lab (image `kindest/node:v1.35.0`, fixée par son digest). `10-build-image.sh` conserve cette image sous un nom, pour qu'aucun nettoyage ne la supprime.

➡️ **Fermez ensuite la session Ubuntu et reconnectez-vous** (ou redémarrez) pour que le groupe `docker` soit pris en compte. Vérification :
```bash
docker run --rm hello-world
```
Si vous ne pouvez pas vous reconnecter tout de suite, préfixez les scripts suivants par `sg docker -c` (ex. `sg docker -c ./05-storage-server.sh`).

### Étape 2 — Le serveur de stockage (`05-storage-server.sh`)
La VM devient le serveur de stockage :
- **Paquets :** `zfsutils-linux`, `nfs-kernel-server`, `nvme-cli`, `openssh-server`, et **nvmetcli** (non packagé dans Ubuntu 24.04, installé depuis son dépôt officiel dans `/opt/nvmetcli`).
- **Modules noyau :** `nvmet nvmet-tcp nvme-tcp nfs nfsd` (installe `linux-modules-extra` si besoin), chargés aussi à chaque démarrage.
- **Bloc :**
  - pool ZFS `tank-nvme` sur `/dev/sdb`, avec les datasets `k8s/vols` (dynamique), `k8s/snaps` et `manual` (exercices) ;
  - port NVMe/TCP `:4420` ;
  - service `nvmet.service`, qui restaure la configuration de la cible à chaque démarrage (`/etc/nvmet/config.json`).
- **Fichiers :**
  - `/dev/sdc` formaté en ext4 (étiquette `storage-lab-nfs`), monté sur `/srv/nfs` (entrée dans `/etc/fstab`) ;
  - export NFS vers `172.18.0.0/16`, avec les dossiers `k8s/` (dynamique) et `manual/` (exercices).
- **SSH :** clé `~/.ssh/democratic-csi`, autorisée pour root **uniquement depuis le réseau kind**. Le driver CSI s'en sert pour créer les volumes.
- **Pare-feu :** ouvre 22, 4420 et 2049 pour le réseau kind si ufw est actif.

🛡️ Le script refuse un disque monté ou non vierge. Pour forcer l'effacement : `FORCE_WIPE=1 ./05-storage-server.sh`.

Vérification :
```bash
sudo zfs list                      # tank-nvme, tank-nvme/k8s/vols, ...
systemctl is-active nvmet          # active
sudo exportfs -v                   # /srv/nfs  172.18.0.0/16(...)
```

### Étape 3 — Image des nœuds (`10-build-image.sh`)
Construit `kind-node-storage:v1.35.0` à partir de l'image officielle `kindest/node`, en ajoutant `nvme-cli` et `nfs-common`. À chaque démarrage, le service `storage-lab-init` du nœud :
- génère son **identité NVMe (hostnqn) unique**. On n'utilise pas `nvme gen-hostnqn`, qui la dérive du matériel : tous les nœuds d'une même VM auraient la même ;
- rend **inscriptibles les seuls dossiers NVMe de `/sys`** (`/sys/devices/virtual/nvme-fabrics` et `nvme-subsystem`). kind monte `/sys` en lecture seule dans ses nœuds, ce qui empêcherait `nvme disconnect` (et donc le driver CSI) de fermer une connexion.

### Étape 4 — Cluster (`20-create-cluster.sh`)
- Crée le réseau docker `kind` avec des **adresses fixes** (la VM est toujours `172.18.0.1`).
- Démarre **cloud-provider-kind** (`v0.12.0`) : un conteneur `cloud-provider-kind` sur le réseau `kind`, relancé automatiquement par docker. Il surveille **tous** les clusters kind et donne à chaque `Service` de type `LoadBalancer` une IP en `172.18.x.x`, joignable depuis la VM.
- Crée le cluster `storage-lab` : 1 control-plane et 2 workers, qui montent `/dev`, `/lib/modules` et `/run/udev` (lecture seule) de la VM.
  `/run/udev` est indispensable : sans la base udev de la VM, `lsblk` ne voit pas le système de fichiers d'un disque NVMe, et le driver CSI le **reformaterait à chaque rattachement**.
- Vérifie sur chaque worker : hostnqn unique, `nvme`, `mount.nfs`, `/dev/nvme-fabrics`, la base udev et le `/sys` NVMe inscriptible.

Vérification :
```bash
kubectl get nodes
```

### Étape 5 — Tests sans Kubernetes (`25-smoke-tests.sh`)
Valide la chaîne réseau avant d'installer les drivers :
- **NVMe/TCP :** un zvol de 256 Mo est publié (le port :4420 n'est réellement ouvert par le noyau qu'à ce moment), un worker fait `nvme discover` puis `nvme connect`, et le disque `/dev/nvmeXnY` doit être visible et lisible ;
- **NFS :** un worker monte un dossier, écrit un fichier, et la VM le relit.

Tout est nettoyé automatiquement, même en cas d'échec.

### Étape 6 — Drivers CSI (`30-install-csi.sh`)
| Release Helm | Rôle | CSIDriver | StorageClass |
|---|---|---|---|
| `csi-nfs` (kube-system) | NFS dynamique et PV statiques NFS | `nfs.csi.k8s.io` | `nfs-shared` |
| `csi-nvme` (democratic-csi) | NVMe/TCP dynamique | `org.democratic-csi.nvmeof` | `nvme-block` |
| `csi-manual` (democratic-csi) | montage des disques NVMe créés à la main | `org.democratic-csi.node-manual` | — |

Les versions des charts et des images sont figées dans `lab.env`. La clé SSH est injectée par `helm --set-file` ; elle n'est jamais écrite dans un fichier généré.

Vérification :
```bash
kubectl get csidrivers
kubectl get storageclass           # nfs-shared, nvme-block, standard
```

### Étape 7 — Validation (`90-test.sh`)
- **NVMe :** un pod avec une PVC formatée et une PVC en **bloc brut** ;
- **NVMe, rattachement :** le pod est supprimé puis recréé ; les données écrites sur les deux volumes doivent être retrouvées ;
- **NFS :** 2 pods, sur 2 workers différents, écrivent dans la même PVC **RWX** ;
- **LoadBalancer :** un service `type: LoadBalancer` reçoit une IP externe et répond à `curl` depuis la VM ;
- **Nettoyage :** tout est supprimé, puis le script vérifie qu'il ne reste sur la VM ni zvol, ni cible NVMe, ni connexion NVMe, ni dossier NFS (sinon il échoue).

`./90-test.sh --keep` garde les ressources pour explorer.

### Étape 8 — Labs (`40-labs.sh`)
Génère le dossier `labs/` (énoncés, manifestes, corrigés) avec les valeurs de `lab.env`.

---

## 5. Utiliser le lab

### Parcours de labs
```bash
cd ~/storage-lab/labs && less README.md
```
| Lab | Thème | Version essentielle (`ESSENTIEL.md`) | Version complète (`README.md`) |
|---|---|---|---|
| 1 | Stockage éphémère | `emptyDir` partagé et lié au pod ; ConfigMap, Secret, Downward API | + `emptyDir` en mémoire, limite et éviction, volume éphémère générique |
| 2 | StorageClass `standard` (local-path) | StorageClass → PVC → PV, classe par défaut, dossier sur un nœud (affinité, pas de quota, pas de partage), `Delete` / `Retain` | + exploration du provisioner |
| 3 | NFS | serveur, **manuel** (PV statique, liaison, `Released`), **dynamique** (RWX sur 2 nœuds) | + volume NFS sans PV, capacité non appliquée, StorageClass sur mesure |
| 4 | NVMe/TCP | disque réseau, **manuel** (`tools/nvme-volume.sh`, PV statique), RWO / Multi-Attach, **dynamique**, volume qui suit le pod | + bloc brut, agrandissement à chaud |
| 5 | StatefulSets | identité stable, **un volume par pod** (`volumeClaimTemplates`), cycle de vie des PVC | — |
| 6 | Bases de données | — | PostgreSQL sur bloc réseau (persistance), perte de nœud (réseau vs local), sauvegarde / restauration NFS, **opérateur CloudNativePG** (2 instances répliquées, bascule automatique) |

Chaque lab contient ses manifestes numérotés, son ou ses énoncés et un `CORRIGE.md` pour le formateur. Le lab 6 utilise une **copie locale** du manifeste de l'opérateur (`vendor/cnpg-1.30.1.yaml`) : aucun téléchargement de manifeste pendant le lab.

> ⚠️ **Images des labs :** les images (postgres, nginx, busybox, opérateur et instances CloudNativePG) sont téléchargées par les nœuds au premier usage. Déroulez les labs une fois sur la VM avant de la cloner pour l'export : elles seront déjà présentes chez les apprenants, même sans Internet.

### Provisionnement manuel (côté administrateur stockage)
```bash
tools/nvme-volume.sh create demo 1G     # zvol + publication NVMe/TCP, chaque commande affichée
tools/nvme-volume.sh list
tools/nvme-volume.sh delete demo

tools/nfs-share.sh create demo          # dossier partagé
tools/nfs-share.sh list
tools/nfs-share.sh delete demo

tools/storage-status.sh                 # PV/PVC côté Kubernetes ↔ volumes côté VM
```
Publier un disque en NVMe/TCP revient à 6 opérations lisibles dans `/sys/kernel/config/nvmet` :
1. créer la cible ;
2. autoriser les clients ;
3. créer le namespace ;
4. indiquer le disque ;
5. l'activer ;
6. l'attacher au port.

### Préparer la VM pour les apprenants (export OVA)
Les apprenants reçoivent une VM **déjà prête** et un dossier qui ne contient que le nécessaire pour les labs : énoncés **sans corrigés**, outils, et un README décrivant l'environnement.
```bash
./90-test.sh                       # 1. l'environnement fonctionne
./50-package-apprenant.sh          # 2. produit storage-lab-apprenant.tar.gz (+ storage-lab-apprenant-CORRIGE.tar.gz)
```
3. **Cloner la VM** (VirtualBox : clic droit → Cloner → *clone intégral*, sans instantanés) : votre VM formateur reste intacte.
4. **Hôte, clone éteint : activer TRIM** sur les trois disques, pour que l'espace libéré dans la VM soit rendu au fichier `.vdi` :
   ```bash
   VBoxManage showvminfo "storage-lab-apprenant" --machinereadable | grep -E '^"SATA-[0-9]-0"'   # chemins des 3 .vdi
   for port in 0 1 2; do
     VBoxManage storageattach "storage-lab-apprenant" --storagectl SATA --port $port --device 0 \
       --type hdd --nonrotational on --discard on --medium "<chemin du .vdi du port $port>"
   done
   ```
   (Adaptez le nom de la VM et du contrôleur, `SATA` par défaut.) Après démarrage, `lsblk -D` doit afficher une valeur non nulle dans `DISC-GRAN` pour `sda`, `sdb` et `sdc`.
5. **Dans le clone :**
   ```bash
   unset HISTFILE                                         # cette session ne laissera pas d'historique
   cp ~/storage-lab/storage-lab-apprenant.tar.gz /tmp/
   sudo mv ~/storage-lab /root/storage-lab-formateur      # scripts de préparation et corrigés hors de portée
   tar xzf /tmp/storage-lab-apprenant.tar.gz -C ~
   ~/storage-lab/tools/reinitialiser-labs.sh              # aucun reste de vos essais
   ~/storage-lab/tools/verifier-env.sh                    # tout doit être ✔
   sudo rm -f /etc/sudoers.d/99-storage-lab               # si un sudo sans mot de passe avait été ajouté
   sudo /root/storage-lab-formateur/60-preparer-export.sh # nettoyage + TRIM, puis extinction
   ```
   `60-preparer-export.sh` :
   - vide les caches apt, le cache de build Docker et les anciennes révisions snap ;
   - vide les journaux, les fichiers temporaires et l'historique ;
   - arrête docker, pour qu'aucun pod ne soit évincé pendant l'opération ;
   - recrée `/swap.img` vide ;
   - libère l'espace libre des **trois** disques (`fstrim` sur `/` et `/srv/nfs`, `zpool trim` sur le pool ZFS) ;
   - éteint la VM.

   Le cluster kind et les images des labs sont **conservés** : les apprenants n'ont rien à télécharger.
   Sans TRIM : `--zero`. L'espace libre est alors rempli de zéros (`zpool initialize` pour ZFS, car un `dd` sur ZFS compressé n'écrit rien). ⚠️ Les `.vdi` grossissent temporairement jusqu'à leur taille maximale sur l'hôte.
6. **Hôte, clone éteint : compacter puis exporter.**
   ```bash
   VBoxManage modifymedium disk "<chemin>/storage-lab-apprenant.vdi"   --compact
   VBoxManage modifymedium disk "<chemin>/storage-nvme.vdi"            --compact
   VBoxManage modifymedium disk "<chemin>/storage-nfs.vdi"             --compact
   ```
   *Fichier → Exporter un appareil virtuel* : format **OVF 2.0**, politique MAC **« Retirer toutes les adresses MAC des cartes réseau »**, aucun ISO dans le lecteur optique. L'OVA compresse les disques. Le cluster kind redémarre seul à l'import.

| Contenu du paquet apprenant (`~/storage-lab`) | |
|---|---|
| `README.md` | l'environnement, les StorageClasses, les outils, « en cas de problème » |
| `labs/` | `README.md` (programme), et par lab : `ESSENTIEL.md`, `COMPLET.md`, manifestes YAML |
| `tools/` | `nfs-share.sh`, `nvme-volume.sh`, `storage-status.sh`, `verifier-env.sh`, `reinitialiser-labs.sh` |
| `.lab/` (caché) | `lib.sh`, `lab.env` : utilisés par les outils |

**Correction :** `storage-lab-apprenant-CORRIGE.tar.gz` ajoute, dans chaque dossier de lab, `ESSENTIEL-CORRIGE.md` et `COMPLET-CORRIGE.md` (chaque réponse sous sa question), plus un index `labs/CORRIGES.md`. À distribuer quand vous le décidez ; il s'extrait par-dessus le paquet apprenant (`tar xzf … -C ~`) sans rien écraser.

---

## 6. Recréer, détruire

```bash
./99-destroy.sh                          # cluster seulement (stockage de la VM conservé)
./20-create-cluster.sh && ./30-install-csi.sh && ./40-labs.sh   # nouveau cluster

./99-destroy.sh --purge                  # + cibles NVMe, zvols, dossiers NFS du lab
./99-destroy.sh --network                # + réseau docker kind
./99-destroy.sh --wipe-disks             # + efface /dev/sdb et /dev/sdc (confirmation « OUI »)
```

| Besoin | Scripts |
|---|---|
| Repartir d'un cluster propre pour une nouvelle session | `99 --purge` → `20` → `30` → `40` |
| Changer de version de Kubernetes | `lab.env` → `10` → `99` → `20` → `30` → `40` |
| Modifier un driver ou une StorageClass | template ou `lab.env` → `30` |
| Mettre à jour le lab (nouvelle archive : image, template kind) | `10` → `99 --purge` → `20` → `30` → `40` |
| Tout réinitialiser, sauf les outils | `99 --purge --network --wipe-disks` → `05` → `10` → `20` → `30` → `40` |

**Après un redémarrage de la VM :** le stockage revient seul (pool ZFS, `nvmet.service` qui republie les cibles enregistrées dans `/etc/nvmet/config.json`, `/etc/fstab`, export NFS), et les conteneurs kind redémarrent avec docker. Les pods se reconnectent à leurs volumes NVMe, sans reformatage. Vérifiez avec `kubectl get nodes` et `tools/storage-status.sh`.

---

## 7. Dépannage

| Symptôme | Piste |
|---|---|
| `permission denied` sur docker | session non rouverte après l'étape 1 (en attendant : `sg docker -c ./<script>.sh`) |
| `05` : « port 4420 configuré (le noyau l'ouvrira…) » | normal : `ss` ne montre le port qu'une fois une cible publiée |
| `05` : modules NVMe introuvables | `sudo apt install linux-modules-extra-$(uname -r)`, puis relancer |
| `05` : disque « non vierge » ou « appartient à un pool ZFS » | vérifier `lsblk` ; si c'est bien le disque du lab : `FORCE_WIPE=1` |
| `20` : réseau `kind` avec un autre sous-réseau | détruire les clusters kind, puis `docker network rm kind` |
| `20` : hostnqn dupliqué ou absent | image à reconstruire : `10` → `99` → `20` |
| `20` : `/dev/nvme-fabrics` absent | `sudo modprobe nvme-tcp` ; vérifier le montage `/dev` dans `build/kind-config.yaml` |
| `20` : base udev absente / `/sys/devices/virtual/nvme-fabrics non accessible en écriture` | cluster créé avec un ancien template ou une ancienne image : `10` → `99` → `20` |
| `nvme0: failed to disconnect, error 30` | `/sys` en lecture seule dans le nœud (image antérieure au correctif) : `10` → `99` → `20`. Connexions orphelines : `sudo nvme list-subsys`, puis `sudo nvme disconnect -n <nqn>` sur la VM |
| `25` NVMe : discovery KO | `systemctl status nvmet`, `ls /sys/kernel/config/nvmet/ports/1`, port 4420 (ufw) |
| `25` NVMe : connecté mais pas de disque | montage `/dev` ; `sudo nvme list-subsys` sur la VM |
| `25` NFS : mount KO | `systemctl status nfs-server`, `sudo exportfs -v`, port 2049 |
| PVC `nvme-block` en `Pending` | `kubectl -n democratic-csi logs deploy/csi-nvme-democratic-csi-controller -c csi-driver`. Causes possibles : SSH, `/run/nvmet-config-loaded` absent (→ `sudo systemctl restart nvmet`), dataset parent |
| Pod NVMe en `ContainerCreating` | `kubectl -n democratic-csi logs ds/csi-nvme-democratic-csi-node -c csi-driver` |
| Données NVMe perdues après redémarrage d'un pod (`mkfs.ext4` dans les logs du node CSI) | `/run/udev` non monté dans les workers : `docker exec storage-lab-worker ls /run/udev/data`, sinon `99` → `20` → `30` |
| `psql` : « No such file or directory » juste après le démarrage de `pg-0` | PostgreSQL démarre encore : `kubectl -n lab5 wait --for=condition=Ready pod/pg-0` |
| Après un redémarrage de la VM, nœuds `NotReady` | attendre 1 à 2 min ; `docker ps` doit montrer les 3 nœuds `Up` (sinon `docker start storage-lab-control-plane storage-lab-worker storage-lab-worker2`) |
| PVC `nfs-shared` en `Pending` | `kubectl -n kube-system logs deploy/csi-nfs-controller -c nfs` |
| Volumes orphelins après destruction du cluster | `./99-destroy.sh --purge` |
| Service `LoadBalancer` en `<pending>` | `docker ps -a \| grep cloud-provider-kind` ; `docker logs cloud-provider-kind` ; relancer `./20-create-cluster.sh` (recrée le conteneur) |
| `kind create cluster --config ~/cluster-1cp2w.yml` télécharge l'image | l'image de base a été supprimée : relancer `./10-build-image.sh` (la retague) |

---

## 8. Pour aller plus loin
- **Snapshots** : installer le snapshot-controller et ajouter une `VolumeSnapshotClass` (les datasets `k8s/snaps` sont prévus).
- **Contrôle d'accès NVMe** : `allow_any_host=0` et une liste de hostnqn autorisés.
- **StorageClass par défaut** : annotation `storageclass.kubernetes.io/is-default-class`.

---

## Annexe — Organisation des fichiers

```
storage-lab/
├── lab.env                   # TOUTES les variables (versions, disques, adresses, noms)
├── lib.sh                    # fonctions communes
├── image/                    # image des nœuds kind (nvme-cli, nfs-common, hostnqn unique)
├── templates/                # config kind, values des 3 drivers, tests, clusters/ (fichiers de cluster simples)
├── labs.tpl/                 # modèles des 6 labs (rendus dans labs/ par 40)
├── vendor/                   # cnpg-1.30.1.yaml : manifeste de l'opérateur CloudNativePG (lab 6)
├── tools/                    # nvme-volume.sh, nfs-share.sh, storage-status.sh
├── tools-formateur/          # inserer-reponses.py (versions des labs avec réponses)
├── apprenant.tpl/            # README et outils propres au paquet apprenant
├── 00-install-tools.sh       # docker, kind, kubectl, helm
├── 05-storage-server.sh      # la VM devient serveur de stockage (ZFS, NVMe/TCP, NFS, SSH)
├── 10-build-image.sh         # image des nœuds
├── 20-create-cluster.sh      # réseau figé + cluster
├── 25-smoke-tests.sh         # tests sans Kubernetes
├── 30-install-csi.sh         # drivers CSI + StorageClasses
├── 40-labs.sh                # génère les labs
├── 50-package-apprenant.sh   # paquet apprenant (sans corrigés ni préparation)
├── 60-preparer-export.sh     # nettoyage + TRIM du clone avant export OVA
├── 90-test.sh                # validation de bout en bout
└── 99-destroy.sh             # destruction
```

**Versions figées dans `lab.env`** : kind `v0.31.0`, Kubernetes et kubectl `v1.35.0`, chart csi-driver-nfs `4.13.6`, chart democratic-csi `0.15.1`, image democratic-csi `v1.9.5`, nvmetcli (commit figé), cloud-provider-kind `v0.12.0`, CloudNativePG `1.30.1`.
