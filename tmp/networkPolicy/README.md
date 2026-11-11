# Lab — Sécuriser une application microservices avec les NetworkPolicies

## Introduction : rappels sur les NetworkPolicies

- Par défaut, le réseau Kubernetes est **à plat**. Tout pod peut joindre tout autre pod, quel que soit son namespace.
- Une **NetworkPolicy** est un pare-feu L3/L4 déclaratif. Elle s'applique aux pods choisis par son `podSelector` et filtre :
  - l'**Ingress** (trafic entrant), par une liste `from` ;
  - l'**Egress** (trafic sortant), par une liste `to` ;
  - avec trois types de sélecteurs : `podSelector`, `namespaceSelector` et `ipBlock`, plus une restriction sur les `ports`.
- Règles du modèle :
  1. Un pod qui n'est sélectionné par **aucune** policy est totalement ouvert.
  2. Dès qu'une policy sélectionne un pod pour un type de trafic (Ingress ou Egress), seul ce qui est explicitement autorisé passe.
  3. Les policies sont **additives**, c'est-à-dire qu'elles s'additionnent (union des autorisations). On n'écrit que des « allow », jamais de « deny ».
  4. Le filtrage est **stateful** : les réponses à une connexion autorisée passent toujours.
- ⚠️ **L'objet NetworkPolicy n'est appliqué que par le plugin réseau (CNI).** Si le CNI ne le supporte pas, la policy est acceptée par l'API mais n'a **aucun effet**.
- Certains CNI (Calico, Cilium, Antrea…) proposent des **extensions** : règles « deny » explicites, policies globales au cluster, filtrage L7 (HTTP) ou par nom DNS. Elles sortent du cadre de ce lab.
- **Dans ce lab**, on s'en tient aux **NetworkPolicies standard** (`networking.k8s.io/v1`). Le CNI par défaut de kind, **kindnet**, les supporte : aucun CNI supplémentaire n'est à installer.

## Étude de cas

```
   Utilisateurs externes
           │  (NodePort 30080)
 ┌─────────▼───────── namespace myapp ──────────────┐
 │   frontend ──► backend1 ──► backend2              │
 └──────────────────│───────────────────────────────┘
                    ▼
 ┌──────────────── namespace app2 ──────────────────┐
 │                 api                              │
 └──────────────────────────────────────────────────┘
```

Chaque microservice est simulé par un pod **nginx**. Sa page d'accueil affiche le nom du service, le nom du pod, son namespace et son IP. La commande `curl`, présente dans l'image, sert aux tests de connectivité.

### Flux métier autorisés (cible)

| # | Source | Destination | Port |
|---|--------|-------------|------|
| F1 | Extérieur (tout le monde) | myapp/frontend | 80 |
| F2 | myapp/frontend | myapp/backend1 | 80 |
| F3 | myapp/backend1 | myapp/backend2 | 80 |
| F4 | myapp/backend1 | app2/api | 80 |
| F5 | pods de myapp | DNS (kube-system/kube-dns) | 53 UDP/TCP |
| — | **Tout le reste** | | **interdit** |

## Contenu du kit

```
manifests/            fichiers fournis aux étudiants
  00-kind-cluster.yaml
  01-namespaces.yaml
  02-frontend.yaml     (Deployment + Service NodePort 30080)
  03-backend1.yaml     (Deployment + Service)
  04-backend2.yaml
  05-app2-api.yaml
  06-intrus.yaml       (pod "attaquant" dans le namespace default)
solutions/            corrigés des NetworkPolicies
test-connectivity.sh  affiche la matrice de connectivité
```

Les labels par défaut `app: <nom>` suffisent pour toutes les policies. Le label `kubernetes.io/metadata.name` est ajouté automatiquement à chaque namespace.

---

## Phase 0 — Déploiement, sans aucune NetworkPolicy

```bash
kind create cluster --config manifests/00-kind-cluster.yaml
kubectl apply -f manifests/01-namespaces.yaml
kubectl apply -f manifests/
kubectl get pods -A -o wide
kubectl get networkpolicy -A        # aucune
```

Test manuel :
```bash
kubectl -n myapp exec deploy/frontend -- curl -s backend1
kubectl -n myapp exec deploy/frontend -- curl -s api.app2   # appel inter-namespace
curl localhost:30080                                       # accès externe
```

Puis `./test-connectivity.sh`.

✅ **Constat attendu :** tout est OK, y compris `frontend → backend2`, `intrus → backend1` ou `api → backend2`. Le réseau est à plat.

**Question :** parmi ces flux, lesquels ne respectent pas le tableau des flux métier ?

## Phase 1 — Isolation en entrée de myapp (default deny Ingress)

**Consigne :** écrire une policy qui bloque **tout** trafic entrant vers les pods de `myapp`.
Corrigé : `solutions/np-01-default-deny-ingress.yaml`

**Constat :** plus rien n'atteint `myapp`, même depuis l'extérieur. `api.app2` reste joignable par tous.
C'est aussi la preuve que le CNI applique bien les policies.

## Phase 2 — Ouvrir le frontend au public (F1)

**Consigne :** autoriser tout le monde vers le frontend, **uniquement sur le port 80**.
Corrigé : `np-02-frontend-allow-external.yaml`

**À remarquer :** une règle `ingress` sans `from` signifie « toute source ».

## Phase 3 — Chaîne interne frontend → backend1 → backend2 (F2, F3)

**Consigne :** backend1 n'accepte que le frontend ; backend2 n'accepte que backend1.
Corrigés : `np-03-…`, `np-04-…`

**Points de vérification :**
- `frontend → backend2` : BLOQUÉ, car on ne peut pas court-circuiter backend1.
- `intrus → backend1` : BLOQUÉ.

## Phase 4 — Flux inter-namespace backend1 → app2/api (F4)

**Consigne :** dans `app2`, appliquer un default deny Ingress, puis n'autoriser vers `api` **que** `backend1` du namespace `myapp`.
Corrigé : `np-05-app2-api-allow-from-backend1.yaml`

**Point clé :** `namespaceSelector` et `podSelector` sont placés dans le **même élément** de `from`. Les deux conditions doivent donc être vraies (ET).

## Phase 5 — Contrôle des sorties (Egress)

1. **Consigne :** appliquer un default deny Egress sur `myapp`, puis relancer les tests.
   **Constat :** même F2 échoue, et `curl backend1` renvoie une erreur de **résolution DNS**.
   **Question :** pourquoi ?
2. Autoriser le DNS vers CoreDNS (corrigé : `np-06-…`).
3. Autoriser explicitement les sorties F2, F3 et F4 (corrigé : `np-07-egress-rules.yaml`).

**À retenir :** pour un flux entre deux pods protégés en Ingress et en Egress, il faut **deux autorisations**, l'Egress côté source et l'Ingress côté destination.

## Phase 6 — Simulation d'attaques

On rejoue chaque scénario et on compare avec la phase 0.

| # | Scénario | Commande | Attendu |
|---|----------|----------|---------|
| A1 | Le frontend compromis tente d'atteindre backend2 directement | `kubectl -n myapp exec deploy/frontend -- curl -m2 backend2` | BLOQUÉ |
| A2 | Le frontend compromis tente de joindre app2 | `… deploy/frontend -- curl -m2 api.app2` | BLOQUÉ |
| A3 | backend2 compromis tente de remonter vers backend1 | `kubectl -n myapp exec deploy/backend2 -- curl -m2 backend1` | BLOQUÉ |
| A4 | Exfiltration vers Internet depuis backend1 | `kubectl -n myapp exec deploy/backend1 -- curl -m2 https://example.com` | BLOQUÉ |
| A5 | Pod intrus dans `default` | `kubectl exec intrus -- curl -m2 backend1.myapp` | BLOQUÉ |
| A6 | Usurpation de label : faux frontend dans `default` | `kubectl run faux-frontend --image=nginx:1.27 -l app=frontend` puis `kubectl exec faux-frontend -- curl -m2 backend1.myapp` | BLOQUÉ (un `podSelector` seul ne vise que le namespace de la policy) |
| A7 | Faux backend1 créé dans `app2` | `kubectl -n app2 run faux-backend1 --image=nginx:1.27 -l app=backend1` puis `kubectl -n app2 exec faux-backend1 -- curl -m2 api` | BLOQUÉ |

## Phase 7 — Le piège AND / OR

Appliquer `solutions/piege-and-or.yaml`. Il remplace la policy de la phase 4 par une version où `namespaceSelector` et `podSelector` sont **deux éléments distincts** de `from`.

Rejouer **A7** : le faux backend1 accède maintenant à `api`.

**Explication :** deux éléments séparés font un **OU**, soit « tout pod de myapp » OU « tout pod `app=backend1` d'app2 ».
Réappliquer ensuite `np-05-…` pour corriger.

## Matrice finale attendue

| Source \ Dest | frontend | backend1 | backend2 | api.app2 |
|---|---|---|---|---|
| myapp/frontend | – | OK | BLOQUÉ | BLOQUÉ |
| myapp/backend1 | BLOQUÉ | – | OK | OK |
| myapp/backend2 | BLOQUÉ | BLOQUÉ | – | BLOQUÉ |
| app2/api | OK (public) | BLOQUÉ | BLOQUÉ | – |
| default/intrus | OK (public) | BLOQUÉ | BLOQUÉ | BLOQUÉ |
| Externe :30080 | OK | | | |

(« – » : un pod vers son propre service, cas non significatif.)

## Questions de synthèse

1. Que se passe-t-il pour un pod qu'aucune policy ne sélectionne ?
2. Pourquoi le default deny Egress casse-t-il la résolution de noms ?
3. Quelle est la différence entre un `namespaceSelector` + `podSelector` dans le même élément `from` et dans deux éléments séparés ?
4. Pourquoi faut-il protéger par le RBAC le droit de modifier les labels des pods et des namespaces ?
5. Quelles limites des NetworkPolicies standard justifieraient un CNI comme Calico ou Cilium ? Exemples : filtrage par nom de domaine, règles deny, policies globales au cluster, filtrage L7.

## Nettoyage

```bash
kind delete cluster --name np-lab
```
