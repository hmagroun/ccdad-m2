# storage-lab — Stockage iSCSI (bloc) + NFS (fichiers) sur KinD

Lab Kubernetes **KinD** sur une VM **VirtualBox Ubuntu 24.04 Desktop**, avec deux StorageClasses à provisionnement dynamique, pilotées par le même driver CSI (**democratic-csi**) :

| StorageClass | Protocole | Type de stockage | Mode d'accès | Backend sur la VM |
|---|---|---|---|---|
| `iscsi-block` | iSCSI | **bloc** (fs ext4 ou bloc brut) | **RWO** (ReadWriteOnce) | zvols ZFS sur `/dev/sdb` exposés par LIO/targetcli |
| `nfs-shared` | NFS v4.2 | **fichiers partagés** | **RWX** (ReadWriteMany) | datasets ZFS sur `/dev/sdc` exportés par nfs-kernel-server |

> **Rappel sur les modes d'accès :** un volume bloc iSCSI ne peut être monté en lecture/écriture que par **un seul nœud** (RWO), sinon le système de fichiers est corrompu. Le partage entre plusieurs pods ou nœuds (RWX) relève du stockage fichiers : c'est le rôle de NFS.
> (Seule exception : RWX en `volumeMode: Block` pour des applications qui gèrent elles-mêmes la concurrence, comme la migration à chaud KubeVirt ou les filesystems clusterisés. Ce cas sort du cadre du lab.)

---

## 1. Architecture

```
┌──────────────────────────── VM Ubuntu 24.04 ─────────────────────────────┐
│  /dev/sdb → pool ZFS tank-iscsi → zvols  → LIO (targetcli)  :3260 iSCSI  │
│  /dev/sdc → pool ZFS tank-nfs   → datasets → nfs-server      :2049 NFS   │
│  sshd :22  ← pilotage par le controller democratic-csi (zfs, targetcli)  │
│                                                                          │
│  ── réseau docker "kind" 172.18.0.0/16 (passerelle = VM = 172.18.0.1) ── │
│  ┌──────────────┐  ┌────────────────────┐  ┌────────────────────┐        │
│  │ control-plane│  │ worker             │  │ worker2            │        │
│  └──────────────┘  │ open-iscsi + nfs   │  │ open-iscsi + nfs   │        │
│                    └────────────────────┘  └────────────────────┘        │
│  democratic-csi : release csi-iscsi + release csi-nfs (controller+node)  │
└──────────────────────────────────────────────────────────────────────────┘
```

Il y a trois flux, avec trois rôles :
- **SSH (22)** : le pod *controller* administre la VM (création ou suppression de zvols, datasets, cibles iSCSI, exports NFS). C'est ce qui rend le provisionnement **dynamique** possible.
- **iSCSI (3260)** : les pods *node* connectent le disque sur le worker qui héberge le pod applicatif.
- **NFS (2049)** : le kubelet des workers monte le partage.

Les nœuds kind sont des conteneurs qui **partagent le noyau de la VM**. Les modules noyau (iscsi_tcp, nfs…) sont donc chargés sur la VM, et le `/dev` de la VM est monté dans les workers pour qu'ils voient les disques iSCSI.

---

## 2. Prérequis

**VM VirtualBox**
- 4 vCPU, 8 Go de RAM minimum.
- **Deux disques virtuels supplémentaires vierges** : `/dev/sdb` (iSCSI) et `/dev/sdc` (NFS), par exemple 20 Go chacun. Vérifiez leurs noms avec `lsblk` : s'ils diffèrent, adaptez `ISCSI_DISK` et `NFS_DISK` dans `lab.env`.

**Outils** (à installer avant le lab) :
```bash
sudo apt update && sudo apt install -y docker.io
sudo usermod -aG docker $USER && newgrp docker

curl -Lo kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64 && chmod +x kind && sudo mv kind /usr/local/bin/
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && sudo install kubectl /usr/local/bin/
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

> `lab.env` est calé sur **kind v0.31.0** (image `kindest/node:v1.35.0`). Avec une autre version de kind, remplacez `K8S_VERSION` et `KIND_BASE_IMAGE` par le tag et le digest indiqués dans ses release notes.

---

## 3. Organisation

```
storage-lab/
├── lab.env                    # TOUTES les variables (seul fichier à modifier)
├── lib.sh                     # fonctions communes (log, render envsubst…)
├── image/
│   ├── Dockerfile             # kindest/node + open-iscsi + nfs-common
│   └── iscsi-iqn.service      # IQN unique généré au boot de chaque nœud
├── templates/                 # rendus dans build/ avec envsubst
│   ├── kind-config.yaml.tpl
│   ├── values-iscsi.yaml.tpl
│   ├── values-nfs.yaml.tpl
│   ├── test-iscsi.yaml.tpl
│   └── test-nfs.yaml.tpl
├── 00-host-prereqs.sh         # VM : paquets, modules, pools ZFS, SSH  (une fois)
├── 10-build-image.sh          # image de nœud kind                     (une fois / version)
├── 20-create-cluster.sh       # réseau docker figé + cluster + vérifs
├── 25-smoke-tests.sh          # tests iSCSI + NFS sans Kubernetes
├── 30-install-csi.sh          # democratic-csi ×2 + StorageClasses
├── 90-test.sh                 # tests PVC de bout en bout
└── 99-destroy.sh              # destruction (cluster / purge / réseau / pools)
```

Principe : **aucune valeur n'est écrite en dur**. Les templates ne contiennent que des `${VARIABLES}` de `lab.env`, rendues dans `build/`. Le rendu ne remplace que ces variables, donc les `$(hostname)` des manifests restent intacts.

---

## 4. Déroulé

```bash
cd storage-lab
chmod +x *.sh
```

### Étape 0 — Préparer la VM (une seule fois)
```bash
./00-host-prereqs.sh
```
Ce que fait le script :
- installe `targetcli-fb`, `zfsutils-linux`, `nfs-kernel-server`, `openssh-server` et `gettext-base` (pour envsubst) ;
- charge et rend persistants les modules `target_core_mod iscsi_target_mod iscsi_tcp nfs nfsd` ;
- active le target LIO, nfs-server et ssh, et **désactive l'iscsid de la VM** (risque de conflit avec ceux des nœuds) ;
- crée les pools `tank-iscsi` (sur `/dev/sdb`) et `tank-nfs` (sur `/dev/sdc`), en lz4 sans atime, ainsi que les datasets `k8s/vols` et `k8s/snaps` ;
- génère `~/.ssh/democratic-csi` et l'autorise pour root, **uniquement depuis 172.18.0.0/16** ;
- ouvre 22, 3260 et 2049 dans ufw pour le réseau kind, si ufw est actif.

🛡️ **Sécurité disques :** le script **refuse** d'utiliser un disque monté ou non vierge. Pour forcer l'effacement : `FORCE_WIPE=1 ./00-host-prereqs.sh`.

Vérification :
```bash
sudo zpool status
sudo zfs list
```

### Étape 1 — Construire l'image des nœuds
```bash
./10-build-image.sh
```
L'image `kind-node-storage:v1.35.0` part de `kindest/node` et ajoute **open-iscsi** et **nfs-common**. Le fichier `/etc/iscsi/initiatorname.iscsi` est **supprimé** au build. Le service `iscsi-iqn.service` le régénère au premier démarrage de chaque nœud, ce qui garantit un IQN par nœud. Un IQN partagé ferait se voler les sessions iSCSI entre nœuds.

### Étape 2 — Créer le cluster
```bash
./20-create-cluster.sh
```
- Crée d'abord le réseau docker `kind` avec un sous-réseau **fixe** (172.18.0.0/16). Ainsi `HOST_IP=172.18.0.1` est connue dès le départ. Si un réseau `kind` existe avec un autre sous-réseau, le script s'arrête et explique quoi faire.
- Crée `storage-lab` : un control-plane et deux workers, qui montent `/dev` et `/lib/modules`.
- Vérifie sur chaque worker qu'iscsid est actif, que l'**IQN est unique** et que `mount.nfs` est présent.

### Étape 3 — Tests sans Kubernetes (recommandé)
```bash
./25-smoke-tests.sh
```
Ce test valide la partie la plus délicate, la chaîne VM ↔ conteneur kind, avant d'ajouter le CSI :
- **iSCSI** : crée une cible temporaire de 256 Mo, puis fait discovery et login depuis un worker, et vérifie que le disque `/dev/sdX` est visible et lisible ;
- **NFS** : crée un dataset temporaire exporté, le monte en NFS 4.2 depuis un worker, écrit dedans et relit le fichier côté VM.

Tout est nettoyé automatiquement, même en cas d'échec.

### Étape 4 — Installer democratic-csi
```bash
./30-install-csi.sh
```
Le script installe **deux releases** du même chart dans le namespace `democratic-csi` :

| Release | Driver | CSIDriver | StorageClass |
|---|---|---|---|
| `csi-iscsi` | `zfs-generic-iscsi` (shareStrategy `targetCli`) | `org.democratic-csi.iscsi` | `iscsi-block` |
| `csi-nfs` | `zfs-generic-nfs` (shareStrategy `setDatasetProperties` → `sharenfs`) | `org.democratic-csi.nfs` | `nfs-shared` |

La clé privée SSH est injectée avec `helm --set-file` : elle n'est écrite ni dans un template ni dans `build/`.

Vérification :
```bash
kubectl -n democratic-csi get pods -o wide   # 2 controllers + 2 DaemonSets node
kubectl get csidrivers
kubectl get sc
```

### Étape 5 — Tests de bout en bout
```bash
./90-test.sh           # teste, nettoie, vérifie que la VM est propre
./90-test.sh --keep    # garde les ressources pour explorer
```
- **iSCSI** : un pod avec une PVC `ext4` montée sur `/data` **et** une PVC en **bloc brut** (`volumeMode: Block`) exposée en `/dev/xvda`. Le script affiche les sessions iSCSI du nœud, les zvols et les cibles LIO.
- **NFS** : un Deployment de 2 réplicas, **forcés sur 2 workers différents** (anti-affinité), qui écrivent dans le même fichier. Le script compte les écrivains : il doit en trouver 2, ce qui prouve le RWX.
- **Nettoyage** : la suppression du namespace doit faire disparaître zvols, datasets, cibles et exports côté VM (`reclaimPolicy: Delete`).

---

## 5. Recréer ou détruire

```bash
./99-destroy.sh                      # cluster seulement (pools et image conservés)
./20-create-cluster.sh && ./30-install-csi.sh    # nouveau cluster prêt en quelques minutes

./99-destroy.sh --purge              # + cibles LIO / volumes orphelins du lab
./99-destroy.sh --network            # + réseau docker kind
./99-destroy.sh --wipe-pools         # + destruction des pools (confirmation "OUI")
```

| Changement | Scripts à relancer |
|---|---|
| Recréer le cluster | `20` → `30` |
| Changer de version Kubernetes | modifier `lab.env` → `10` → `99` → `20` → `30` |
| Modifier un paramètre CSI/SC | modifier `lab.env` ou le template → `30` |
| Réinstaller la VM | tout depuis `00` |

---

## 6. Explorer à la main

```bash
source lab.env
# Côté VM
sudo targetcli ls
sudo zfs list -t all -r tank-iscsi tank-nfs
sudo exportfs -v
# Côté nœud kind
docker exec storage-lab-worker iscsiadm -m session -P3
docker exec storage-lab-worker mount | grep nfs
# Côté CSI
kubectl -n democratic-csi logs deploy/csi-iscsi-democratic-csi-controller -c csi-driver
kubectl -n democratic-csi logs deploy/csi-nfs-democratic-csi-controller -c csi-driver
kubectl -n democratic-csi logs ds/csi-iscsi-democratic-csi-node -c csi-driver
```

---

## 7. Dépannage

| Symptôme | Cause probable / action |
|---|---|
| `20` : réseau `kind` avec un autre sous-réseau | un ancien réseau existe : détruire les clusters kind, puis `docker network rm kind` |
| `20` : IQN dupliqué | l'image a été construite avec un IQN : vérifier le Dockerfile et relancer `10` puis `99` et `20` |
| `25` iSCSI : discovery KO | port 3260, ufw, `sudo targetcli ls /iscsi` (portail 0.0.0.0:3260 présent ?) |
| `25` iSCSI : session OK, disque invisible | montage `/dev` absent dans la config kind, ou `iscsi_tcp` non chargé |
| Erreurs netlink / iscsid | un `iscsid` tourne sur la VM : `sudo systemctl disable --now iscsid.socket iscsid` |
| `25` NFS : mount KO | `systemctl status nfs-server`, `sudo exportfs -v`, port 2049 |
| PVC en `Pending` | logs du controller : SSH refusé (clé, `from=`, PermitRootLogin), dataset parent absent |
| Pod en `ContainerCreating` (iSCSI) | logs du DaemonSet node `-c csi-driver`, puis `iscsiadm -m session` sur le nœud |
| Pod en `ContainerCreating` (NFS) | `kubectl describe pod` : erreur `mount.nfs` (export, version NFS) |
| Volumes restants après suppression | `./99-destroy.sh --purge` (si le cluster a été détruit avant les PVC) |
| Après un reboot de la VM | `sudo zpool import -a` si besoin ; le cluster kind redémarre avec docker |

---

## 8. Pour aller plus loin

- **Snapshots CSI** : installer les CRD et le snapshot-controller, puis renseigner `volumeSnapshotClasses` dans les templates (les datasets `k8s/snaps` sont déjà prévus).
- **CHAP iSCSI** : `authentication: 1` et un bloc `auth` dans `values-iscsi.yaml.tpl`.
- **Utilisateur SSH dédié** au lieu de root, avec un `sudo` limité à `zfs`, `zpool` et `targetcli` (option `zfs.cli.sudoEnabled` de democratic-csi).
- **Expansion** : augmenter `resources.requests.storage` d'une PVC (`allowVolumeExpansion: true` est déjà activé).
- **StorageClass par défaut** : `defaultClass: true` sur l'une des deux.
