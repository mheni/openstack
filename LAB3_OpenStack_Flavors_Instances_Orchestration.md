# LAB 3 : Flavors, Lancement d’Instances et Orchestration de Déploiements

---

## 🎯 Objectifs du LAB

À l’issue de ce LAB, vous serez capable de :

- Concevoir et gérer des **flavors** adaptées à différents cas d’usage.
- Lancer des **instances** sur des nœuds de calcul spécifiques (Compute1 / Compute2).
- Organiser un **déploiement orchestré** de plusieurs VMs (topologie applicative simple).
- Automatiser le lancement d’un ensemble d’instances via un **script** (pré‑orchestration).

Ce lab s’appuie sur :

- L’architecture DevStack multi‑nœuds (Controller + 2 Compute + Block1).
- Les projets et utilisateurs du LAB 1 (`Formation1`, `Formation2`, `AdminLab`).
- Le réseau et le stockage mis en place dans le LAB 2 (`f1-net`, `f2-net`, volumes Cinder, floating IPs).

---

## 📚 Prérequis

Avant de commencer :

1. Les LABs 1 et 2 sont réalisés (projets, quotas, réseaux, security groups, volumes).
2. Au moins une image est disponible pour le boot :
   - `cirros-0.6.2-x86_64-disk` (test rapide) ou
   - une image Ubuntu cloud (pour scénarios plus riches).
3. Depuis le **controller** :

```bash
source /opt/stack/devstack/openrc admin admin
openstack hypervisor list
openstack flavor list
openstack image list
```

Résultat attendu :

- Deux hyperviseurs (compute1, compute2).
- Quelques flavors par défaut (m1.tiny, etc.).
- Au moins une image active.

---

## 🧩 Scénario pédagogique

Vous allez :

1. Créer un **catalogue de flavors** cohérent (tiny, small, medium, large).
2. Lancer des instances de types différents et observer leur placement sur les hyperviseurs.
3. Mettre en place une **mini‑architecture 3‑tiers** pour `Formation1` (web + app + db).
4. Écrire un **script shell** qui déploie automatiquement cette topologie.

Ce LAB vous rapproche de la logique d’**orchestration** (sans aller jusqu’à Heat) et prépare le terrain pour des outils comme Ansible, Terraform ou les templates Heat.

---

## PARTIE 1 : Conception et création de flavors

### 1.1 Analyse des hyperviseurs

En tant qu’`admin` :

```bash
source /opt/stack/devstack/openrc admin admin

# Voir les ressources des hyperviseurs
openstack hypervisor list
openstack hypervisor show compute1
openstack hypervisor show compute2
```

Notez pour chaque hyperviseur : nombre de vCPUs, RAM totale, disque.

### 1.2 Définition d’un catalogue de flavors

Proposition de flavors :

| Flavor       | vCPUs | RAM (Mo) | Disque (Go) | Usage prévu                  |
|--------------|-------|----------|-------------|------------------------------|
| `demo.tiny`  | 1     | 512      | 5           | Tests, Cirros, ping/SSH      |
| `demo.small` | 1     | 2048     | 10          | Petites apps / bastion       |
| `demo.medium`| 2     | 4096     | 20          | App serveur web / API        |
| `demo.large` | 4     | 8192     | 40          | DB / appli lourde (formation)|

Créez ces flavors :

```bash
# Idées de IDs : 101, 102, 103, 104 (optionnel)
openstack flavor create demo.tiny   --id 101 --vcpus 1 --ram 512   --disk 5
openstack flavor create demo.small  --id 102 --vcpus 1 --ram 2048  --disk 10
openstack flavor create demo.medium --id 103 --vcpus 2 --ram 4096  --disk 20
openstack flavor create demo.large  --id 104 --vcpus 4 --ram 8192  --disk 40

# Vérifier
openstack flavor list
```

### 1.3 (Option) Limitation de flavors par projet

OpenStack permet de rendre certains flavors privés et de les associer à un projet donné.

Exemple : réserver `demo.large` au projet `AdminLab` (facultatif) :

```bash
# Rendre demo.large privé
openstack flavor set demo.large --property "class=adminlab" --private

# Associer le flavor à un projet
openstack flavor set demo.large --project AdminLab
```

> Exercice : tester depuis `user1` si `demo.large` est visible ou non selon la configuration.

---

## PARTIE 2 : Lancement d’instances ciblées par hyperviseur

### 2.1 Contexte Formation1 (user1)

Sur le controller :

```bash
cat > user1-openrc.sh << 'EOF'
export OS_AUTH_URL=http://10.0.0.11/identity
export OS_PROJECT_NAME=Formation1
export OS_USERNAME=user1
export OS_PASSWORD=user1pass
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
EOF

source user1-openrc.sh

# Vérifier
openstack token issue
openstack flavor list
openstack network list
```

Assurez-vous que `f1-net` est présent et que les flavors `demo.*` sont visibles.

### 2.2 Lancer une instance sur compute1

Nous allons utiliser l’option `--availability-zone nova:compute1`.

```bash
# Variables d'aide (adapter l'image si besoin)
IMG="cirros-0.6.2-x86_64-disk"
NET="f1-net"
KEY="mykey"   # à créer si nécessaire

# Instance sur compute1
openstack server create   --flavor demo.small   --image "$IMG"   --network "$NET"   --key-name "$KEY"   --availability-zone nova:compute1   f1-web-compute1

# Vérifier
openstack server list
openstack server show f1-web-compute1 -f yaml | grep hypervisor
```

**Résultat attendu** : l’instance `f1-web-compute1` est ACTIVE et placée sur `compute1`.

### 2.3 Lancer une instance sur compute2

```bash
openstack server create   --flavor demo.small   --image "$IMG"   --network "$NET"   --key-name "$KEY"   --availability-zone nova:compute2   f1-web-compute2

openstack server show f1-web-compute2 -f yaml | grep hypervisor
```

**Résultat attendu** : `f1-web-compute2` est sur `compute2`.

> Question : que se passe-t-il si compute2 est down (service `n-cpu` arrêté) et que vous relancez la commande ?

---

## PARTIE 3 : Mini‑architecture 3‑tiers pour Formation1

Objectif : déployer une petite topologie :

- 1 VM `f1-front` (web) – flavor `demo.small`.
- 1 VM `f1-app` (application) – flavor `demo.medium`.
- 1 VM `f1-db` (base de données) – flavor `demo.large`.

Toutes sur `f1-net`, mais réparties sur les deux hyperviseurs.

### 3.1 Préparation – Security groups

On réutilise le security group `ssh-only` du LAB 2 ou on en crée un nouveau `web-ssh-db` (exercice possible). Pour simplifier, utilisons `ssh-only` :

```bash
source user1-openrc.sh
openstack security group list
```

Assurez-vous que `ssh-only` existe et autorise au minimum SSH et ICMP.

### 3.2 Déploiement des 3 VMs

```bash
# f1-front sur compute1
openstack server create   --flavor demo.small   --image "$IMG"   --network "$NET"   --key-name "$KEY"   --security-group ssh-only   --availability-zone nova:compute1   f1-front

# f1-app sur compute2
openstack server create   --flavor demo.medium   --image "$IMG"   --network "$NET"   --key-name "$KEY"   --security-group ssh-only   --availability-zone nova:compute2   f1-app

# f1-db sur compute2 (par exemple)
openstack server create   --flavor demo.large   --image "$IMG"   --network "$NET"   --key-name "$KEY"   --security-group ssh-only   --availability-zone nova:compute2   f1-db

# Vérifier l'état global
openstack server list
```

### 3.3 Attribution de floating IPs

Attribuez une floating IP au front (et éventuellement à l’app) :

```bash
# Créer une floating IP
FIP_FRONT=$(openstack floating ip create -f value -c floating_ip_address public)

# Associer à f1-front
openstack server add floating ip f1-front "$FIP_FRONT"

# Vérifier
openstack server show f1-front -f yaml | egrep 'addresses|floating'
```

Depuis l’hôte (ou une machine ayant accès au réseau provider) :

```bash
ping "$FIP_FRONT"
ssh cirros@"$FIP_FRONT"   # ou ubuntu@ selon l'image
```

> Exercice avancé :
> - Installer un serveur web sur `f1-front` et une base sur `f1-db` (si image Ubuntu cloud).
> - Tester la communication interne (ping entre VMs, accès DB depuis `f1-app`).

---

## PARTIE 4 : Script d’“orchestration” simple

Objectif : écrire un script shell qui déploie toute la stack `Formation1` automatiquement :

- Crée le réseau `f1-net` et `f1-subnet` (si non existants).
- Crée les flavors `demo.*` (si absents).
- Lance `f1-front`, `f1-app`, `f1-db` avec la topologie définie.

### 4.1 Exemple de script `deploy_f1_stack.sh`

Sur le **controller** :

```bash
cat > deploy_f1_stack.sh << 'EOF'
#!/usr/bin/env bash
set -e

# Charger l'environnement user1
source ~/user1-openrc.sh

IMG="cirros-0.6.2-x86_64-disk"
NET_NAME="f1-net"
SUBNET_NAME="f1-subnet"
SUBNET_CIDR="192.168.201.0/24"
KEY="mykey"

# 1. Réseau (idempotent)
if ! openstack network show "$NET_NAME" &>/dev/null; then
  echo "[INFO] Création du réseau $NET_NAME"
  openstack network create --project Formation1 "$NET_NAME"
  openstack subnet create     --project Formation1     --network "$NET_NAME"     --subnet-range "$SUBNET_CIDR"     --dns-nameserver 8.8.8.8     "$SUBNET_NAME"
fi

# 2. Flavors (idempotent)
create_flavor_if_missing() {
  local name=$1 vcpus=$2 ram=$3 disk=$4 id=$5
  if ! openstack flavor show "$name" &>/dev/null; then
    echo "[INFO] Création du flavor $name"
    openstack flavor create "$name" --id "$id" --vcpus "$vcpus" --ram "$ram" --disk "$disk"
  fi
}

create_flavor_if_missing demo.tiny   1 512   5  101
create_flavor_if_missing demo.small  1 2048  10 102
create_flavor_if_missing demo.medium 2 4096  20 103
create_flavor_if_missing demo.large  4 8192  40 104

# 3. Security group ssh-only (idempotent)
if ! openstack security group show ssh-only &>/dev/null; then
  echo "[INFO] Création du security group ssh-only"
  openstack security group create ssh-only --description "SSH + ICMP"
  openstack security group rule create --proto tcp --dst-port 22 --ingress ssh-only
  openstack security group rule create --proto icmp --ingress ssh-only
fi

# 4. Lancement des VMs
NET_ID=$(openstack network show "$NET_NAME" -f value -c id)

create_server_if_missing() {
  local name=$1 flavor=$2 az=$3
  if ! openstack server show "$name" &>/dev/null; then
    echo "[INFO] Création de la VM $name"
    openstack server create       --flavor "$flavor"       --image "$IMG"       --network "$NET_ID"       --key-name "$KEY"       --security-group ssh-only       --availability-zone "$az"       "$name"
  else
    echo "[INFO] VM $name déjà existante, skip"
  fi
}

create_server_if_missing f1-front demo.small  "nova:compute1"
create_server_if_missing f1-app   demo.medium "nova:compute2"
create_server_if_missing f1-db    demo.large  "nova:compute2"

openstack server list
EOF

chmod +x deploy_f1_stack.sh
```

Exécuter le script :

```bash
./deploy_f1_stack.sh
```

**Résultat attendu** :

- Si rien n’existe encore, le script crée réseau + flavors + security group + VMs.
- Si vous relancez le script, il détecte les éléments existants et ne les duplique pas (idempotence simple).

### 4.2 Améliorations possibles

- Ajouter la création d’un routeur `f1-router` et la configuration de la gateway externe.
- Ajouter l’allocation et l’association automatique de floating IPs à `f1-front`.
- Ajouter la création/attachement d’un volume Cinder à `f1-db`.

---

## PARTIE 5 : Vue d’ensemble admin et contrôle de capacité

En tant qu’`admin` :

```bash
source /opt/stack/devstack/openrc admin admin

# Vue globale des instances
openstack server list --all-projects

# Vue des hyperviseurs et ressources consommées
openstack hypervisor stats show
openstack hypervisor list
openstack hypervisor show compute1
openstack hypervisor show compute2

# Vue des flavors
openstack flavor list
```

Analyse :

- Vérifiez la répartition des VMs entre compute1 et compute2.
- Comparez les ressources allouées aux capacités restantes.
- Discutez des limites possibles (quotas projet, capacités hyperviseur, overcommit, etc.).

---

## 🔎 Questions de validation

1. À quoi sert un **flavor** dans OpenStack et quels sont ses paramètres principaux ?
2. Comment forcer le placement d’une instance sur un hyperviseur spécifique ?
3. Quels sont les risques d’un catalogue de flavors mal dimensionné ?
4. Qu’apporte un script d’“orchestration” par rapport à des commandes manuelles ?
5. Comment vérifier l’impact de vos déploiements sur la **capacité** des hyperviseurs ?

---

## ✅ Bilan du LAB 3

Dans ce troisième LAB, vous avez :

- Construit un catalogue de flavors adapté à des scénarios de formation.
- Placé des instances sur des hyperviseurs précis via les availability zones.
- Déployé une petite architecture multi‑VM pour un projet donné.
- Automatisé ce déploiement avec un script shell réutilisable.
- Appris à analyser la capacité et la répartition de charge entre vos compute nodes.

Ce LAB prépare la transition vers de vraies solutions d’orchestration (Heat, Ansible, Terraform) qui industrialisent le déploiement d’environnements complexes.

---

**Auteur** : DR; ING MAHER HENI LAB DevStack Multi-nœuds – Flavors & Orchestration  
**Version** : 1.0  
**Date** : Février 2026  
**Licence** : Usage pédagogique et formation
