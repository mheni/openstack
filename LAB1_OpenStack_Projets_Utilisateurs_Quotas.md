# LAB : Administration OpenStack – Projets, Utilisateurs, Rôles et Quotas

---

##  Objectifs du LAB

À l’issue de ce LAB, vous serez capable de :

- Créer et organiser des **projets** OpenStack.
- Créer des **utilisateurs** et leur attribuer des **rôles**.
- Travailler en **multi‑tenant** (isolement entre projets).
- Configurer et vérifier les **quotas** (instances, vCPU, RAM, volumes, floating IPs).
- Utiliser à la fois le **Dashboard Horizon** et la **CLI OpenStack** pour l’administration.

Ce lab s’appuie sur l’architecture DevStack multi‑nœuds déployée précédemment :

- 1 Controller (API, DB, Cinder API) – `10.0.0.11`
- 2 Compute nodes (nova-compute + OVN)
- 1 Block storage (Cinder volume/backup)

Identifiants admin par défaut :

- URL Horizon : `http://10.0.0.11/dashboard`
- Domaine : `default`
- Utilisateur : `admin`
- Mot de passe : `openstack`

---

##  Prérequis

Avant de commencer ce LAB, assurez-vous que :

1. Le déploiement DevStack multi‑nœuds est **terminé** et **fonctionnel** (Controller + Compute1 + Compute2 + Block1).
2. Vous pouvez vous connecter à Horizon avec le compte **admin**.
3. Depuis le nœud **controller**, la commande suivante fonctionne :

```bash
source /opt/stack/devstack/openrc admin admin
openstack service list
```

Résultat attendu : liste des services (identity, compute, image, volume, network, etc.) avec l’état **enabled** / **up**.

---

##  Scénario pédagogique

Vous allez préparer un environnement de formation avec :

- Un projet **Formation1** pour les stagiaires du groupe 1.
- Un projet **Formation2** pour un second groupe.
- Un projet **AdminLab** réservé aux tests de l’administrateur.
- Un utilisateur **user1** rattaché à Formation1.
- Un utilisateur **user2** rattaché à Formation2.
- Un utilisateur **trainer** avec des droits admin sur AdminLab.

Vous configurerez ensuite des **quotas différents** pour chaque projet, et vous vérifierez en pratique l’impact sur la création d’instances et de volumes.

---

## PARTIE 1 : Création des projets (tenants)

### 1.1 Création des projets via CLI

Sur le **controller**, en tant qu’utilisateur **stack** :

```bash
source /opt/stack/devstack/openrc admin admin

# Créer les projets
openstack project create Formation1
openstack project create Formation2
openstack project create AdminLab

# Vérifier
openstack project list
```

**Résultat attendu** : Les projets `Formation1`, `Formation2` et `AdminLab` apparaissent dans la liste, avec un `ID` propre à chacun.

### 1.2 Vérification dans Horizon

1. Connectez-vous à **Horizon** avec le compte `admin`.
2. Allez dans **Identity → Projects**.
3. Vérifiez la présence des 3 projets : `Formation1`, `Formation2`, `AdminLab`.

> Note : Profitez-en pour observer les colonnes (Description, Enabled, Domain, etc.).

---

## PARTIE 2 : Création des utilisateurs et attribution des rôles

### 2.1 Création des utilisateurs via CLI

Toujours sur le **controller** :

```bash
source /opt/stack/devstack/openrc admin admin

# Créer les utilisateurs avec mot de passe simple
openstack user create --password user1pass --enable user1
openstack user create --password user2pass --enable user2
openstack user create --password trainerpass --enable trainer

# Vérifier la liste des utilisateurs
openstack user list
```

**Résultat attendu** : Les utilisateurs `user1`, `user2`, `trainer` apparaissent dans la liste.

### 2.2 Attribution des rôles

Rappel : dans DevStack, les rôles courants sont `admin` et `member`.

- `user1` doit être **member** dans Formation1.
- `user2` doit être **member** dans Formation2.
- `trainer` doit être **admin** dans AdminLab.

```bash
# Récupérer les IDs des rôles
openstack role list

# Associer les rôles
openstack role add --project Formation1 --user user1 member
openstack role add --project Formation2 --user user2 member
openstack role add --project AdminLab  --user trainer admin
```

Vérification :

```bash
openstack role assignment list --names --user user1
openstack role assignment list --names --user user2
openstack role assignment list --names --user trainer
```

**Résultat attendu** :

- `user1` → rôle `member` sur projet `Formation1`.
- `user2` → rôle `member` sur projet `Formation2`.
- `trainer` → rôle `admin` sur projet `AdminLab`.

### 2.3 Vérification dans Horizon

1. Connectez-vous à Horizon en `admin`.
2. Allez dans **Identity → Users**.
3. Pour chaque utilisateur (`user1`, `user2`, `trainer`), vérifiez :
   - Le projet principal (Primary Project).
   - Les rôles associés.

---

## PARTIE 3 : Tests d’isolement entre projets

### 3.1 Connexion en tant que user1

Dans un navigateur, connectez-vous à Horizon avec :

- User Name : `user1`
- Password : `user1pass`
- Domain : `default`

Actions à effectuer :

1. Aller dans **Compute → Instances**.
2. Vérifier que `user1` ne voit **aucune** instance des autres projets (normalement liste vide au début).
3. Aller dans **Project → Compute → Volumes** et **Images** et noter quelles ressources sont visibles.

### 3.2 Connexion en tant que user2

Même démarche pour `user2` :

- User Name : `user2`
- Password : `user2pass`

Vérifier que `user2` ne voit **pas** les futures ressources créées dans Formation1.

> Ce test sera complété après la partie 5 (création d’instances/volumes par projet).

---

## PARTIE 4 : Quotas – état initial et modification

### 4.1 Afficher les quotas par défaut

En CLI, avec les credentials admin :

```bash
source /opt/stack/devstack/openrc admin admin

# Quotas compute + réseau pour Formation1
openstack quota show Formation1

```

Notez les valeurs par défaut (instances, cores, ram, volumes, gigabytes, floating IPs…).

Répétez pour `Formation2` et `AdminLab` si besoin :

```bash
openstack quota show Formation2
```

### 4.2 Définir des quotas spécifiques

Objectif :

- **Formation1** : environnement restreint.
- **Formation2** : environnement plus généreux.
- **AdminLab** : environnement quasi illimité (pour tests admin).

#### 4.2.1 Quotas Formation1 (restreint)

Exemple de quotas :

- 3 instances max
- 4 vCPUs
- 8 Go de RAM
- 2 volumes / 10 Go
- 2 floating IPs

```bash
# Compute / réseau
openstack quota set   --instances 3   --cores 4   --ram 8192   --floating-ips 2   Formation1

# Cinder (stockage bloc)
openstack quota set   --volumes 2   --gigabytes 10   Formation1
```

#### 4.2.2 Quotas Formation2 (plus large)

Exemple de quotas :

- 8 instances max
- 16 vCPUs
- 32 Go de RAM
- 10 volumes / 100 Go
- 5 floating IPs

```bash
openstack quota set   --instances 8   --cores 16   --ram 32768   --floating-ips 5   Formation2

openstack quota set   --volumes 10   --gigabytes 100   Formation2
```

#### 4.2.3 Quotas AdminLab (large)

```bash
openstack quota set   --instances 20   --cores 40   --ram 131072   --floating-ips 20   AdminLab

openstack quota set   --volumes 20   --gigabytes 200   AdminLab
```

### 4.3 Vérification des quotas

```bash
openstack quota show Formation1


openstack quota show Formation2

```

Vérifiez que les valeurs affichées correspondent bien aux objectifs définis.

Dans Horizon :

1. Connectez-vous en `admin`.
2. Allez dans **Identity → Projects → Formation1 → Quotas**.
3. Vérifiez visuellement les limites (instances, vCPUs, RAM, volumes…).

---

## PARTIE 5 : Validation pratique des quotas

### 5.1 Préparation – environnement réseau commun

Pour simplifier ce lab, vous pouvez utiliser le réseau privé déjà créé dans le lab précédent (par exemple `private-net`). Sinon, créez un réseau pour `Formation1` :

En CLI, en tant que `admin`, mais en ciblant le projet `Formation1` :

```bash
source /opt/stack/devstack/openrc admin admin
# Vérifier d'abord que le réseau "public" existe
openstack network list --external
# Option 1 : créer un réseau dédié Formation1
openstack network create --project Formation1 f1-net
openstack subnet create   --project Formation1   --network f1-net   --subnet-range 192.168.201.0/24   f1-subnet

openstack router create --project Formation1 f1-router
openstack router add subnet f1-router f1-subnet
openstack router set --external-gateway public f1-router
```

### 5.2 Se placer en contexte user1 (Formation1)

Sur le controller, créez un fichier RC pour `user1` :

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
```

**Résultat attendu** : un token est émis pour le projet `Formation1` et l’utilisateur `user1`.

### 5.3 Création d’instances jusqu’à atteindre le quota

Hypothèse : une flavor légère `m1.tiny` et une image `cirros-0.6.2-x86_64-disk` sont disponibles.

```bash
# Vérifier les quotas vus par user1
openstack quota show

# Lancer des instances dans Formation1
openstack server create   --flavor m1.tiny   --image cirros-0.6.2-x86_64-disk   --network f1-net --build f1-vm1

openstack server create   --flavor m1.tiny   --image cirros-0.6.2-x86_64-disk   --network f1-net  --build f1-vm2

openstack server create   --flavor m1.tiny   --image cirros-0.6.2-x86_64-disk   --network f1-net --build f1-vm3
```

Essayez ensuite de créer **une 4e instance** :

```bash
openstack server create   --flavor m1.tiny   --image cirros-0.6.2-x86_64-disk   --network f1-net   f1-vm4
```

**Résultat attendu** : la commande échoue avec une erreur liée au quota (nombre d’instances ou cores dépassé).

### 5.4 Test des quotas de volumes pour Formation1

Toujours en tant que `user1` :

```bash
# Créer deux volumes de 5 Go chacun
openstack volume create --size 5 f1-vol1
openstack volume create --size 5 f1-vol2

# Tenter un 3e volume (quota volumes=2 ou gigabytes=10)
openstack volume create --size 1 f1-vol3

# Dans les quotas Formation1 définis en Partie 4 :
# volumes=2, gigabytes=10

```

**Résultat attendu** : la création du 3e volume échoue pour cause de quotas Cinder.
**explication**
- Vous créez f1-vol1 (5Go) + f1-vol2 (5Go) = 10Go → quota gigabytes atteint
- Le 3e volume échoue pour gigabytes ET pour volumes count → bien

- laquelle des deux limites est atteinte en premier :
- volumes=2 → atteint après vol2 (count)
- gigabytes=10 → atteint après vol1+vol2 (5+5=10Go)
- → le 3e échoue sur les DEUX critères simultanément

### 5.5 Comparaison avec Formation2 (quotas plus larges)

Créez un fichier RC pour `user2` :

```bash
cat > user2-openrc.sh << 'EOF'
export OS_AUTH_URL=http://10.0.0.11/identity
export OS_PROJECT_NAME=Formation2
export OS_USERNAME=user2
export OS_PASSWORD=user2pass
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
EOF

source user2-openrc.sh
openstack token issue
```

Créez plusieurs instances et volumes dans `Formation2` : vous devriez pouvoir dépasser largement les limites de `Formation1` sans erreur, grâce aux quotas plus généreux.

```bash
cat > trainer-openrc.sh << 'EOF'
export OS_AUTH_URL=http://10.0.0.11/identity
export OS_PROJECT_NAME=AdminLab
export OS_USERNAME=trainer
export OS_PASSWORD=trainerpass
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
EOF
```

# bonne pratique: Ajouter une description aux projets
---
```bash
openstack project create --description "Groupe stagiaires 1" Formation1
openstack project create --description "Groupe stagiaires 2" Formation2
openstack project create --description "Tests administrateur" AdminLab
```
---

## PARTIE 6 : Vue administrateur et supervision

Revenez en contexte `admin` :

```bash
source /opt/stack/devstack/openrc admin admin

# Lister toutes les instances, tous projets confondus
openstack server list --all-projects

# Lister les volumes de tous les projets
openstack volume list --all-projects
```

Dans Horizon :

- Utilisez le menu **Admin → Compute → Instances** pour voir les VMs de tous les projets.
- Comparez avec la vue projet `Formation1` et `Formation2` (en vous reconnectant avec `user1` et `user2`).

> L’administrateur voit tout, alors que chaque utilisateur ne voit que son propre projet.

---

## 🔎 Questions de validation (à poser aux stagiaires)

1. Quelle est la différence entre un **projet** et un **utilisateur** dans OpenStack ?
2. Que permet le rôle **admin** par rapport au rôle **member** ?
3. Comment un quota d’instances est-il appliqué lorsqu’il y a plusieurs flavors ?
4. Où peut-on voir et modifier les quotas : en CLI ? dans Horizon ?
5. Que se passe-t-il lorsqu’un utilisateur atteint son quota de volumes ?

---

##  Bilan du LAB

À ce stade, vous avez :

- Créé plusieurs projets et utilisateurs.
- Assigné des rôles adaptés (member / admin).
- Observé l’isolement entre projets (multi‑tenant).
- Mis en place des quotas différenciés et vérifié leur impact réel.
- Utilisé à la fois la CLI et Horizon pour l’administration.

Ce LAB constitue une brique de base pour tous les labs suivants (réseau, stockage, automatisation), qui réutiliseront ces projets et utilisateurs.

---

**Auteur** : R; ING MAHER HENI LAB DevStack Multi-nœuds – Administration OpenStack  
**Version** : 1.0  
**Date** : Février 2026  
**Licence** : Usage pédagogique et formation
