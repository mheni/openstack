# LAB 2 : Administration OpenStack – Réseau, Sécurité et Stockage Bloc

---

##  Objectifs du LAB

À l’issue de ce LAB, vous serez capable de :

- Créer et administrer des **réseaux**, **subnets** et **routeurs** Neutron.
- Gérer les **security groups** et contrôler l’accès aux instances.
- Créer, attacher et gérer des **volumes Cinder** (stockage bloc).
- Tester la connectivité réseau (privé, flottant, accès externe) et le comportement en cas d’erreurs.

Ce lab réutilise les projets et utilisateurs du LAB 1 :

- Projets : `Formation1`, `Formation2`, `AdminLab`.
- Utilisateurs : `user1` (Formation1), `user2` (Formation2), `trainer` (AdminLab, admin).

Architecture DevStack multi‑nœuds :

- 1 Controller – `10.0.0.11` (API, DB, Cinder API, Horizon).
- 2 Compute nodes – nova-compute + OVN.
- 1 Block storage – Cinder volume/backup (Block1).

---

##  Prérequis

Avant de commencer :

1. Le LAB 1 (Projets, Utilisateurs, Rôles et Quotas) est **terminé**.
2. Vous pouvez vous connecter à Horizon avec les comptes :
   - `admin / openstack`
   - `user1 / user1pass`
   - `user2 / user2pass`.
3. Depuis le nœud **controller**, les commandes suivantes fonctionnent :

```bash
source /opt/stack/devstack/openrc admin admin
openstack network agent list
openstack volume service list
```

Résultat attendu : agents réseau OVN en **up** et services Cinder (scheduler, volume, backup) en **enabled/up**.[web:11][web:15]

---

##  Scénario pédagogique

Dans ce LAB, vous allez :

- Créer une **topologie réseau complète** pour `Formation1` et `Formation2` (réseaux privés + routeurs + accès externe).
- Définir des **security groups** adaptés (SSH only, Web, tout fermé) et les appliquer aux instances.
- Mettre en œuvre le **stockage bloc** Cinder : création, attachement, détachement et suppression de volumes.
- Valider les flux réseau (ping, SSH) et les opérations sur les disques vus depuis les VMs.

---

## PARTIE 1 : Réseau Formation1 (Neutron)

### 1.1 Création du réseau privé et subnet

Sur le **controller**, avec les credentials admin, en ciblant le projet `Formation1` :

```bash
source /opt/stack/devstack/openrc admin admin

# Créer un réseau privé pour Formation1
openstack network create --project Formation1 f1-net

# Créer un subnet pour ce réseau
openstack subnet create   --project Formation1   --network f1-net   --subnet-range 192.168.201.0/24   --dns-nameserver 8.8.8.8   f1-subnet

# Vérifier
openstack network list --project Formation1
openstack subnet list --project Formation1
```

**Résultat attendu** : un réseau `f1-net` avec le subnet `192.168.201.0/24` est visible pour le projet Formation1.[web:11][web:15]

### 1.2 Routeur et connexion au réseau public

Le réseau public (provider) a été créé par DevStack (souvent nommé `public`).

```bash
# Créer un routeur pour Formation1
openstack router create --project Formation1 f1-router

# Attacher le subnet privé au routeur
openstack router add subnet f1-router f1-subnet

# Configurer la passerelle externe vers le réseau public
openstack router set --external-gateway public f1-router

# Vérifier
openstack router list
openstack router show f1-router
```

**Résultat attendu** : le routeur `f1-router` a une interface interne sur `f1-subnet` et une gateway sur `public`.[web:11][web:15]

### 1.3 Vérification dans Horizon

1. Connectez-vous à Horizon avec `user1`.
2. Allez dans **Project → Network → Network Topology**.
3. Vérifiez que vous voyez :
   - Le réseau `f1-net`.
   - Le routeur `f1-router` connecté à `f1-net` et au réseau externe.

---

## PARTIE 2 : Réseau Formation2 et différenciation

### 2.1 Réseau privé et routeur Formation2

Même démarche pour `Formation2` :

```bash
source /opt/stack/devstack/openrc admin admin

# Réseau privé Formation2
openstack network create --project Formation2 f2-net

openstack subnet create   --project Formation2   --network f2-net   --subnet-range 192.168.202.0/24   --dns-nameserver 8.8.8.8   f2-subnet

# Routeur Formation2
openstack router create --project Formation2 f2-router
openstack router add subnet f2-router f2-subnet
openstack router set --external-gateway public f2-router
```

**Résultat attendu** : `f2-net`, `f2-subnet` et `f2-router` apparaissent et sont isolés de `f1-net`.[web:11][web:15]

### 2.2 Tests d’isolement (Vue Horizon)

- Connectez-vous avec `user1` → vous ne devez voir que `f1-net` / `f1-router`.
- Connectez-vous avec `user2` → vous ne devez voir que `f2-net` / `f2-router`.

> Question aux stagiaires : Que se passe-t-il si un utilisateur de Formation1 tente d’attacher une interface sur `f2-net` ?

---

## PARTIE 3 : Security Groups et accès SSH

### 3.1 Création d’un security group “SSH only” pour Formation1

En tant que `user1` (ou admin ciblant Formation1) :

```bash
# Sur le controller
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

# Créer un security group SSH only
openstack security group create ssh-only --description "SSH access only"

# Règles : autoriser SSH (port 22) depuis partout
openstack security group rule create   --proto tcp   --dst-port 22   --ingress   ssh-only

# autoriser ICMP pour ping
openstack security group rule create   --proto icmp   --ingress   ssh-only

# Vérifier
openstack security group list
openstack security group show ssh-only
```

**Résultat attendu** : un security group `ssh-only` avec au moins deux règles (TCP 22, ICMP).[web:11][web:16]

### 3.2 Lancement d’une instance Formation1 avec security group SSH

Hypothèses :

- Une image `cirros-0.6.2-x86_64-disk` ou Ubuntu cloud est disponible.
- Une paire de clés `mykey` existe (sinon, la créer : `openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey`).

```bash
# Toujours en contexte user1

openstack server create   --flavor m1.tiny   --image cirros-0.6.2-x86_64-disk   --network f1-net   --key-name mykey   --security-group ssh-only --wait   f1-vm-ssh

# Vérifier
openstack server list
```

**Résultat attendu** : `f1-vm-ssh` passe en statut `ACTIVE` et reçoit une IP privée dans `192.168.201.0/24`.[web:11][web:15]

### 3.3 Attribution d’une floating IP et test d’accès

En tant que `user1` :

```bash
# Créer une floating IP sur le réseau public
openstack floating ip create public

# Noter l'adresse (ex: 203.0.113.150)
openstack floating ip list

# Associer à l'instance
openstack server add floating ip f1-vm-ssh <FLOATING_IP>

# Vérifier
openstack server show f1-vm-ssh
```

Depuis l’hôte physique (ou une machine ayant accès au réseau provider) :

```bash
ping <FLOATING_IP>
ssh cirros@<FLOATING_IP>   # ou ubuntu@<FLOATING_IP> selon l’image
# mot de passe : gocubsgo
# le mot de passe par défaut de cirros
# OU utiliser la console VNC depuis Horizon si SSH ne fonctionne pas
```

**Résultat attendu** : ping OK (si ICMP autorisé) et connexion SSH possible grâce au security group `ssh-only`.[web:11][web:16]

---

## PARTIE 4 : Storage bloc Cinder pour Formation1

### 4.1 Création de volumes

Toujours en contexte `user1` :

```bash
source user1-openrc.sh

# Créer deux volumes de 2 Go
openstack volume create --size 2 f1-data1
openstack volume create --size 2 f1-data2

# Vérifier
openstack volume list
```
**si vous voyer le status des volume en ERROR alors**

```correction dans le ficher /etc/cinder/cinder.conf sur la machine block1 
```bash
# Sur block1 — ajouter transport_url dans cinder.conf
sudo nano /etc/cinder/cinder.conf
```

```ini
[DEFAULT]
transport_url = rabbit://stackrabbit:openstack@10.0.0.11:5672/
```

```bash
# Redémarrer sur block1
sudo systemctl restart devstack@c-vol.service

# Redémarrer sur controller
sudo systemctl restart devstack@c-api.service devstack@c-sch.service
```
```

**Résultat attendu** : `f1-data1` et `f1-data2` en statut `available`.[web:11][web:13]

### 4.2 Attacher un volume à une instance

```bash
# Attacher f1-data1 à f1-vm-ssh
openstack server add volume f1-vm-ssh f1-data1

# Vérifier
openstack volume show f1-data1
openstack server show f1-vm-ssh
```

**Résultat attendu** : le volume `f1-data1` passe en statut `in-use` et est listé dans les volumes attachés de `f1-vm-ssh`.[web:11][web:13]

### 4.3 Vérification dans la VM

Depuis l’hôte, connectez-vous en SSH sur `f1-vm-ssh` (via sa floating IP) et vérifiez la présence du disque :

```bash
# Exemple sur Cirros (fonctionnalités limitées) ou Ubuntu cloud

# Afficher les disques
sudo lsblk

# Selon l’image, le nouveau disque apparaît (ex: /dev/vdb).
```

> Pour un lab plus avancé avec Ubuntu cloud, vous pouvez créer un filesystem sur ce disque, monter un point de montage et y écrire des données.

### 4.4 Détacher et supprimer un volume

Toujours en `user1` :

```bash
# Détacher le volume
openstack server remove volume f1-vm-ssh f1-data1

# Vérifier le statut
openstack volume show f1-data1

# Supprimer le volume
openstack volume delete f1-data1
```

**Résultat attendu** : le volume revient en `available` après détachement, puis disparaît après suppression.[web:11][web:13]

---

## PARTIE 5 : Réseau et stockage pour Formation2 (exercice)

Pour `Formation2`, reproduisez les mêmes opérations **en autonomie** :

1. Créer un réseau `f2-net`, un subnet `f2-subnet` et un routeur `f2-router` (déjà fait en Partie 2).  
2. Créer un security group `web-ssh` qui :
   - Autorise SSH (TCP 22) et HTTP (TCP 80) en entrée.
   - Autorise ICMP (ping).
3. Lancer une instance `f2-vm1` attachée à `f2-net` avec `web-ssh`.
4. Lui associer une floating IP et tester :
   - ping,  
   - SSH,  
   - ouverture d’un port 80 (ex: `curl http://<FLOATING_IP>` si un service web est lancé dans la VM).
5. Créer un volume `f2-data1`, l’attacher à `f2-vm1`, vérifier dans la VM, puis le détacher et supprimer.

> L’instructeur peut ajuster les quotas de `Formation2` pour permettre plus de volumes/instances si besoin.[web:11][web:15]

---

## PARTIE 6 : Supervision admin et dépannage rapide

Revenez en contexte `admin` sur le controller :

```bash
source /opt/stack/devstack/openrc admin admin

# Voir les réseaux de tous les projets
openstack network list --all-projects

# Voir toutes les instances
openstack server list --all-projects

# Voir tous les volumes
openstack volume list --all-projects

# Vérifier les agents réseau
openstack network agent list

# Vérifier les services Cinder
openstack volume service list
```

En cas de problème réseau :

- Vérifier les routers et leurs interfaces : `openstack router show <router>`.[web:11][web:15][web:16]
- Vérifier les security groups associés aux VMs.
- Vérifier que les floating IPs sont bien associées aux bonnes instances.

En cas de problème Cinder :

- Vérifier les logs sur `block1` et `controller` (`/opt/stack/logs/c-*`).
- Vérifier les quotas Cinder pour le projet (`openstack volume quota show <projet>`).[web:11][web:13]

---

## 🔎 Questions de validation

1. Quelle différence y a-t-il entre le **réseau privé** d’un projet et le **réseau public/provider** ?
2. À quoi sert un **routeur** Neutron dans ce contexte multi‑tenant ?
3. Comment les **security groups** se comparent-ils à un firewall traditionnel ?
4. Quelle est la différence entre un **volume Cinder** et le disque racine d’une instance ?
5. Que se passe-t-il si vous supprimez une instance qui a un volume **attaché** ?

---

##  Bilan du LAB 2

Dans ce deuxième LAB, vous avez :

- Mis en place des réseaux privés par projet et relié ces réseaux au provider via des routeurs.
- Créé et appliqué des security groups pour contrôler finement les flux réseau.
- Utilisé le stockage bloc Cinder pour ajouter/supprimer des disques aux instances.
- Validé les flux (ping, SSH, éventuellement HTTP) via des floating IPs.
- Pris l’habitude d’utiliser à la fois la CLI et Horizon pour diagnostiquer les problèmes.

Ces compétences servent de base aux labs suivants (haute dispo, monitoring, automatisation avancée).

---

**Auteur** : DR. ING MAHER HENILAB DevStack Multi-nœuds – Réseau & Stockage  
**Version** : 1.0  
**Date** : Février 2026  
**Licence** : Usage pédagogique et formation
