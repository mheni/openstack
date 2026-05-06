# LAB 0 : Installation OpenStack Single-Node avec DevStack

---

## Objectifs du LAB

À l'issue de ce LAB, vous serez capable de :

✅ Préparer une VM Ubuntu 22.04 pour DevStack  
✅ Configurer et déployer OpenStack en mode All-in-One  
✅ Vérifier les services OpenStack (Nova, Neutron, Glance, Cinder, Horizon)  
✅ Accéder au Dashboard Horizon et à la CLI OpenStack  
✅ Diagnostiquer et résoudre les problèmes courants  

---

## Architecture du LAB

```
┌─────────────────────────────────────────────┐
│              VM Unique (All-in-One)          │
│                                             │
│  ┌─────────────────────────────────────┐    │
│  │           DevStack                  │    │
│  │                                     │    │
│  │  Keystone   Nova-API   Nova-CPU     │    │
│  │  Glance     Neutron    Cinder       │    │
│  │  Placement  OVN        Horizon      │    │
│  │  MySQL      RabbitMQ               │    │
│  └─────────────────────────────────────┘    │
│                                             │
│  HOST_IP : 10.0.0.11 (ens33 NAT)           │
└─────────────────────────────────────────────┘
```

**Caractéristiques techniques :**

| Paramètre | Valeur |
|-----------|--------|
| OS | Ubuntu Server 22.04 LTS |
| DevStack version | 2026.1 (Epoxy) |
| Réseau VM | 1 carte NAT |
| Durée estimée | 1 h 30 min |
| Prérequis hôte | VMware Workstation, 12 Go RAM minimum |

---

## Table des matières

1. [Préparation de la VM](#partie-1--préparation-de-la-vm)
2. [Configuration réseau et système](#partie-2--configuration-réseau-et-système)
3. [Préparation DevStack](#partie-3--préparation-devstack)
4. [Fichier local.conf](#partie-4--fichier-localconf)
5. [Installation](#partie-5--installation)
6. [Vérifications post-installation](#partie-6--vérifications-post-installation)
7. [Tests fonctionnels](#partie-7--tests-fonctionnels)
8. [Annexe : Dépannage](#annexe--dépannage)

---

## PARTIE 1 : Préparation de la VM

### 1.1 Spécifications minimales

| Ressource | Minimum | Recommandé |
|-----------|---------|------------|
| vCPU | 2 | 4 |
| RAM | 8 Go | 12 Go |
| Disque | 60 Go | 100 Go |
| Carte réseau | 1 (NAT) | 1 (NAT) |
| OS | Ubuntu Server 22.04 LTS | Ubuntu Server 22.04 LTS |

### 1.2 Création de la VM dans VMware Workstation

1. **File → New Virtual Machine → Custom**
2. Sélectionner **Ubuntu Server 22.04 LTS ISO**
3. Configurer les ressources selon le tableau ci-dessus
4. **VM Settings → Network Adapter** : sélectionner **NAT (VMnet8)**
5. Dans **VM Settings → Processors** : cocher **Virtualize Intel VT-x/EPT**

### 1.3 Installation Ubuntu

Lors de l'installation Ubuntu :

- Utilisateur système : `stagiaire`
- Installer **OpenSSH server** : **Yes**
- Pas de packages supplémentaires
- Partitionnement : **Disque entier** (sans LVM)

**Résultat attendu :** Connexion SSH possible sur la VM.

---

## PARTIE 2 : Configuration réseau et système

### 2.1 Vérifier la connectivité Internet

```bash
ping -c 3 8.8.8.8
ping -c 3 openstack.org
```

**Résultat attendu :** 3 réponses reçues pour chaque commande.

### 2.2 Identifier l'interface réseau

```bash
ip a
```

**Résultat attendu :**

- `lo` : loopback (127.0.0.1)
- `ens33` : NAT — IP DHCP automatique (~10.10.10.x ou 192.168.x.x)

> ⚠️ **Important :** Notez l'IP attribuée par DHCP. C'est cette IP qui sera utilisée comme `HOST_IP` dans le `local.conf`.

### 2.3 Relever l'IP de la VM

```bash
ip a show ens33 | grep "inet "
```

Exemple de résultat :

```
inet 192.168.91.128/24 brd 192.168.91.255 scope global dynamic ens33
```

> 📝 **Notez votre IP ici :** `__________________`  
> Elle sera utilisée à la place de `10.0.0.11` dans toute la suite du lab.

### 2.4 Mise à jour du système

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y git sudo vim curl
```

**Durée :** 3 à 5 minutes.

---

## PARTIE 3 : Préparation DevStack

### 3.1 Créer l'utilisateur stack

DevStack **ne doit pas** être exécuté en root. Il faut créer un utilisateur dédié.

```bash
sudo useradd -s /bin/bash -d /opt/stack -m stack
sudo chmod +x /opt/stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo passwd stack
```

Mot de passe à définir : **stack**

Puis basculer vers cet utilisateur :

```bash
sudo su - stack
pwd
```

**Résultat attendu :** `/opt/stack`

### 3.2 Cloner le dépôt DevStack

```bash
git clone https://opendev.org/openstack/devstack
cd devstack
```

**Résultat attendu :** Répertoire `/opt/stack/devstack` créé avec succès.

```bash
ls /opt/stack/devstack
```

Vous devez voir les fichiers : `stack.sh`, `unstack.sh`, `clean.sh`, `local.conf.sample`, etc.

---

## PARTIE 4 : Fichier local.conf

Le fichier `local.conf` est le **cœur de la configuration DevStack**. Il définit quels services installer et comment les configurer.

### 4.1 Créer le fichier local.conf

Depuis `/opt/stack/devstack` :

```bash
cat > local.conf << 'EOF'
[[local|localrc]]

# ============================================================
# IP de cette VM — À ADAPTER selon votre résultat de "ip a"
# ============================================================
HOST_IP=10.0.0.11

# ============================================================
# Mots de passe (identiques pour simplifier le lab)
# ============================================================
ADMIN_PASSWORD=openstack
DATABASE_PASSWORD=openstack
RABBIT_PASSWORD=openstack
SERVICE_PASSWORD=openstack

# ============================================================
# Logs
# ============================================================
LOGFILE=/opt/stack/logs/stack.sh.log
LOG_COLOR=False
RECLONE=False

# ============================================================
# Services — All-in-One (tous sur ce nœud)
# ============================================================

# Socle : base de données et messagerie
ENABLED_SERVICES=mysql,rabbit,key

# Nova : API + Compute sur le même nœud
ENABLED_SERVICES+=,n-api,n-cpu,n-cond,n-sch,n-novnc,n-api-meta

# Placement
ENABLED_SERVICES+=,placement-api,placement-client

# Glance (Images)
ENABLED_SERVICES+=,g-api

# Cinder (Stockage Bloc) — API + Scheduler + Volume sur ce nœud
ENABLED_SERVICES+=,c-api,c-sch,c-vol,c-bak

# Neutron avec OVN
ENABLED_SERVICES+=,q-svc,q-meta
ENABLED_SERVICES+=,ovn-controller,ovn-northd,ovs-vswitchd,ovsdb-server,q-ovn-metadata-agent

# Horizon (Dashboard Web)
ENABLED_SERVICES+=,horizon

# ============================================================
# Configuration réseau
# ============================================================
FIXED_RANGE=10.11.12.0/24
FLOATING_RANGE=203.0.113.0/24
PUBLIC_NETWORK_GATEWAY=203.0.113.1
Q_FLOATING_ALLOCATION_POOL=start=203.0.113.101,end=203.0.113.250

Q_USE_PROVIDERNET_FOR_PUBLIC=True
PUBLIC_INTERFACE=ens33

# ============================================================
# VNC Console
# ============================================================
NOVA_VNC_ENABLED=True
VNCSERVER_LISTEN=0.0.0.0
VNCSERVER_PROXYCLIENT_ADDRESS=$HOST_IP

EOF
```

### 4.2 Vérifier le fichier

```bash
cat local.conf
```

> ⚠️ **Vérification critique :** La ligne `HOST_IP=` doit contenir l'IP réelle de votre VM relevée à l'étape 2.3. Corrigez-la si nécessaire :

```bash
# Exemple de correction :
sed -i 's/HOST_IP=10.0.0.11/HOST_IP=192.168.91.128/' local.conf
```

---

## PARTIE 5 : Installation

### 5.1 Lancer stack.sh

```bash
cd /opt/stack/devstack
./stack.sh
```

> ⏱️ **Durée estimée :** 20 à 40 minutes selon la connexion Internet.

Pendant l'installation, vous verrez défiler des messages de ce type :

```
[Call Trace]
./stack.sh:228:source
[...]
+lib/nova:install_nova:1         pip_install ...
+lib/neutron:install_neutron:1   pip_install ...
```

### 5.2 Résultat attendu en fin d'installation

```
=========================
DevStack Component Timing
=========================
...
This is your host IP address: 10.0.0.11
This is your host IPv6 address: ::1
Horizon is now available at http://10.0.0.11/dashboard
Keystone is serving at http://10.0.0.11/identity/
The default users are: admin and demo
The password: openstack

DevStack Version: 2026.1
OS Version: Ubuntu 22.04 jammy
```

> ✅ **Si vous voyez ce message, l'installation est réussie.**

### 5.3 En cas d'erreur Horizon (code 127)

Si DevStack échoue sur Horizon, commenter la ligne dans `local.conf` :

```bash
sed -i 's/ENABLED_SERVICES+=,horizon/#ENABLED_SERVICES+=,horizon/' local.conf
```

Relancer l'installation :

```bash
./stack.sh
```

Puis installer Horizon manuellement après :

```bash
sudo apt install -y openstack-dashboard
sudo systemctl restart apache2
```

Accéder ensuite via : `http://<HOST_IP>/horizon`

---

## PARTIE 6 : Vérifications post-installation

### 6.1 Sourcer les credentials admin

```bash
source /opt/stack/devstack/openrc admin admin
```

**Résultat attendu :** Aucune erreur. Les variables d'environnement `OS_*` sont chargées.

```bash
env | grep OS_
```

### 6.2 Vérifier les services Keystone

```bash
openstack service list
```

**Résultat attendu :**

```
+------------------+------------+
| Name             | Type       |
+------------------+------------+
| keystone         | identity   |
| nova             | compute    |
| glance           | image      |
| cinderv3         | volumev3   |
| neutron          | network    |
| placement        | placement  |
+------------------+------------+
```

### 6.3 Vérifier Nova (Compute)

```bash
openstack compute service list
```

**Résultat attendu :**

```
+----+----------------+------------+----------+---------+-------+
| ID | Binary         | Host       | Zone     | Status  | State |
+----+----------------+------------+----------+---------+-------+
|  1 | nova-scheduler | controller | internal | enabled | up    |
|  2 | nova-conductor | controller | internal | enabled | up    |
|  3 | nova-compute   | controller | nova     | enabled | up    |
+----+----------------+------------+----------+---------+-------+
```

### 6.4 Vérifier Cinder (Stockage)

```bash
openstack volume service list
```

**Résultat attendu :**

```
+------------------+-------------+------+---------+-------+
| Binary           | Host        | Zone | Status  | State |
+------------------+-------------+------+---------+-------+
| cinder-scheduler | controller  | nova | enabled | up    |
| cinder-volume    | controller@ | nova | enabled | up    |
| cinder-backup    | controller  | nova | enabled | up    |
+------------------+-------------+------+---------+-------+
```

### 6.5 Vérifier Neutron/OVN (Réseau)

```bash
openstack network agent list
```

**Résultat attendu :** Agents OVN Controller, OVN Metadata Agent visibles et en état `UP`.

### 6.6 Vérifier les endpoints

```bash
openstack endpoint list
```

> ⚠️ Si des endpoints affichent `127.0.0.1` au lieu de votre `HOST_IP`, corriger avec :

```bash
# Exemple pour Glance si l'adresse est incorrecte :
sudo sed -i 's/http-socket = 127.0.0.1:60999/http-socket = 0.0.0.0:9292/' /etc/glance/glance-uwsgi.ini
sudo systemctl restart devstack@g-api
```

---

## PARTIE 7 : Tests fonctionnels

### 7.1 Accès au Dashboard Horizon

Ouvrir un navigateur web et accéder à :

```
http://<HOST_IP>/dashboard
```

**Connexion :**

| Champ | Valeur |
|-------|--------|
| Domain | `default` |
| User Name | `admin` |
| Password | `openstack` |

**Résultat attendu :** Le Dashboard OpenStack s'affiche avec les menus Project, Admin, Identity.

### 7.2 Tester la création d'un volume

```bash
openstack volume create --size 2 test-volume
openstack volume list
```

**Résultat attendu :**

```
+--------------------------------------+-------------+-----------+------+
| ID                                   | Name        | Status    | Size |
+--------------------------------------+-------------+-----------+------+
| xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx | test-volume | available |    2 |
+--------------------------------------+-------------+-----------+------+
```

### 7.3 Tester la création d'un réseau

```bash
openstack network create test-net
openstack subnet create \
  --network test-net \
  --subnet-range 192.168.100.0/24 \
  test-subnet
openstack network list
```

**Résultat attendu :** Réseau `test-net` visible avec le subnet `192.168.100.0/24`.

### 7.4 Tester le lancement d'une instance

```bash
# Créer une paire de clés
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey 2>/dev/null || \
  ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa && \
  openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey

# Lancer une instance minimale avec l'image CirrOS
openstack server create \
  --flavor m1.tiny \
  --image cirros-0.6.2-x86_64-disk \
  --network test-net \
  --key-name mykey \
  vm-test

# Attendre quelques secondes puis vérifier
sleep 10
openstack server list
```

**Résultat attendu :**

```
+--------------------------------------+---------+--------+-----------------------------+
| ID                                   | Name    | Status | Networks                    |
+--------------------------------------+---------+--------+-----------------------------+
| xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx | vm-test | ACTIVE | test-net=192.168.100.x      |
+--------------------------------------+---------+--------+-----------------------------+
```

L'instance est en statut **ACTIVE**.

### 7.5 Accéder à la console VNC

```bash
openstack console url show vm-test
```

Ouvrir l'URL retournée dans un navigateur pour accéder à la console de la VM.

**Identifiants CirrOS :**
- Login : `cirros`
- Password : `gocubsgo`

---

## PARTIE 8 : Gestion du cycle de vie DevStack

### 8.1 Arrêter DevStack proprement

```bash
cd /opt/stack/devstack
./unstack.sh
```

### 8.2 Redémarrer DevStack après un reboot

> ⚠️ DevStack est **éphémère** : après un reboot de la VM, les services ne redémarrent pas automatiquement. Il faut relancer manuellement.

```bash
sudo su - stack
cd /opt/stack/devstack
./unstack.sh    # Nettoyer les résidus
./stack.sh      # Relancer l'installation
```

### 8.3 Réinitialisation complète

En cas de problème grave, repartir de zéro :

```bash
cd /opt/stack/devstack
./clean.sh
./stack.sh
```

---

## ANNEXE : Dépannage

### Problème : stack.sh échoue en milieu d'installation

**Symptôme :** Erreur rouge pendant `./stack.sh`

**Solutions :**

```bash
# Consulter les logs
tail -100 /opt/stack/logs/stack.sh.log

# Identifier le service en erreur
grep -i "error\|failed\|Error" /opt/stack/logs/stack.sh.log | tail -20
```

Corriger le problème puis relancer :

```bash
./stack.sh
```

---

### Problème : Horizon inaccessible (ERR_CONNECTION_REFUSED)

**Symptôme :** Le navigateur ne peut pas accéder à `http://<HOST_IP>/dashboard`

**Solutions :**

```bash
# Vérifier qu'Apache tourne
sudo systemctl status apache2

# Vérifier que le port 80 écoute
sudo ss -ltnp | grep :80

# Redémarrer Apache
sudo systemctl restart apache2

# Si Horizon n'est pas installé
sudo apt install -y openstack-dashboard
sudo systemctl restart apache2
```

---

### Problème : `openstack` CLI retourne des erreurs d'authentification

**Symptôme :** `The request you have made requires authentication (HTTP 401)`

**Solution :**

```bash
# Re-sourcer les credentials
source /opt/stack/devstack/openrc admin admin

# Vérifier les variables
echo $OS_AUTH_URL
echo $OS_USERNAME
```

---

### Problème : Instance reste en statut ERROR ou BUILD

**Symptôme :** `openstack server list` affiche `ERROR` ou reste sur `BUILD`

**Solutions :**

```bash
# Voir les détails de l'erreur
openstack server show vm-test

# Consulter les logs Nova
tail -50 /opt/stack/logs/n-cpu.log

# Vérifier que nova-compute tourne
systemctl status devstack@n-cpu
```

---

### Problème : Port Glance ou Cinder sur 127.0.0.1

**Symptôme :** Les endpoints pointent vers `127.0.0.1` et les services sont inaccessibles depuis l'extérieur

**Solution Glance :**

```bash
sudo sed -i 's/http-socket = 127.0.0.1:60999/http-socket = 0.0.0.0:9292/' /etc/glance/glance-uwsgi.ini
sudo systemctl restart devstack@g-api
sudo ss -ltnp | grep 9292
```

**Solution Cinder :**

```bash
sudo nano /etc/cinder/cinder-api-uwsgi.ini
# Ajouter la ligne : http-socket = 0.0.0.0:8776
sudo systemctl restart devstack@c-api
sudo ss -ltnp | grep 8776
```

---

### Tableau récapitulatif des ports OpenStack

| Service | Port | Vérification |
|---------|------|-------------|
| Keystone | 5000 | `sudo ss -ltnp \| grep 5000` |
| Nova API | 8774 | `sudo ss -ltnp \| grep 8774` |
| Glance | 9292 | `sudo ss -ltnp \| grep 9292` |
| Cinder | 8776 | `sudo ss -ltnp \| grep 8776` |
| Neutron | 9696 | `sudo ss -ltnp \| grep 9696` |
| Placement | 8778 | `sudo ss -ltnp \| grep 8778` |
| Horizon | 80 | `sudo ss -ltnp \| grep :80` |
| VNC Proxy | 6080 | `sudo ss -ltnp \| grep 6080` |

---

## FIN DU LAB

**Félicitations !** Vous disposez maintenant d'un cloud OpenStack All-in-One fonctionnel avec :

✅ Authentification (Keystone)  
✅ Calcul (Nova) avec hyperviseur local  
✅ Images (Glance)  
✅ Réseau (Neutron / OVN)  
✅ Stockage Bloc (Cinder / LVM)  
✅ Dashboard Web (Horizon)  

### Accès au cloud

| Interface | URL | Identifiants |
|-----------|-----|-------------|
| Dashboard Horizon | `http://<HOST_IP>/dashboard` | admin / openstack |
| API Keystone | `http://<HOST_IP>/identity/` | — |

### Prochaine étape

Ce lab est le prérequis pour les labs suivants :

- **LAB 1** : Administration – Projets, Utilisateurs, Rôles et Quotas
- **LAB 2** : Administration – Réseau, Sécurité et Stockage Bloc
- **LAB 3** : Flavors, Lancement d'Instances et Orchestration
- **LAB 4** : Gestion des Images (Glance) et Personnalisation

---

**Auteur :** Dr. Ing. MAHER HENI  
**Version :** 2026.1 (Epoxy)  
**Date :** Mai 2026  
**Licence :** Usage pédagogique et formation — All Rights Reserved
