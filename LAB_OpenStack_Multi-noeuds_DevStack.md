# LAB : Installation OpenStack Multi-nœuds avec DevStack

---

## 📋 Objectifs du LAB

À l'issue de ce LAB, vous serez capable de :

✅ Configurer un environnement de virtualisation VMware Workstation pour OpenStack  
✅ Déployer une architecture OpenStack multi-nœuds (1 controller + 2 compute + 1 storage)  
✅ Maîtriser la configuration réseau multi-interfaces (NAT, Provider, Management)  
✅ Installer et configurer DevStack en mode distribué  
✅ Vérifier et valider les services OpenStack (Nova, Neutron, Glance, Cinder, Horizon)  
✅ Créer et gérer des instances virtuelles et volumes de stockage  
✅ Diagnostiquer et résoudre les problèmes courants d'installation

---

## 📚 Table des matières

1. [Architecture du lab](#architecture-du-lab)
2. [PARTIE 1 : Préparation VMware Workstation](#partie-1--préparation-vmware-workstation)
   - [1.1 Créer les réseaux virtuels](#11-créer-les-réseaux-virtuels)
   - [1.2 Créer les 4 VMs Ubuntu 22.04](#12-créer-les-4-vms-ubuntu-2204)
3. [PARTIE 2 : Configuration réseau et système](#partie-2--configuration-réseau-et-système)
   - [2.1 Hostnames](#21-hostnames)
   - [2.2 Identifier les interfaces](#22-identifier-les-interfaces)
   - [2.3 Plan d'adressage](#23-plan-dadressage)
   - [2.4 Configuration Netplan](#24-configuration-netplan)
   - [2.5 Configuration /etc/hosts](#25-configuration-etchosts)
   - [2.6 Tests de connectivité](#26-tests-de-connectivité)
4. [PARTIE 3 : Préparation DevStack](#partie-3--préparation-devstack)
   - [3.1 Créer l'utilisateur stack](#31-créer-lutilisateur-stack-sur-les-4-vms)
   - [3.2 Configuration SSH](#32-configuration-ssh-controller--tous-les-nœuds)
5. [PARTIE 4 : Installation DevStack – Controller](#partie-4--installation-devstack--controller)
   - [4.1 Télécharger DevStack](#41-télécharger-devstack)
   - [4.2 Fichier local.conf](#42-fichier-localconf)
   - [4.3 Installation](#43-installation)
   - [4.4 Vérifications](#44-vérifications)
6. [PARTIE 5 : Installation DevStack – Compute1](#partie-5--installation-devstack--compute1)
7. [PARTIE 6 : Installation DevStack – Compute2](#partie-6--installation-devstack--compute2)
8. [PARTIE 7 : Installation DevStack – Block1](#partie-7--installation-devstack--block1)
9. [PARTIE 8 : Enregistrement et vérifications](#partie-8--enregistrement-et-vérifications)
   - [8.1 Enregistrer les compute nodes](#81-enregistrer-les-compute-nodes)
   - [8.2 Vérifier les agents réseau](#82-vérifier-les-agents-réseau)
   - [8.3 Vérifier Cinder](#83-vérifier-cinder)
10. [PARTIE 9 : Tests fonctionnels](#partie-9--tests-fonctionnels)
    - [9.1 Dashboard Horizon](#91-dashboard-horizon)
    - [9.2 Test volume](#92-test-volume)
    - [9.3 Test instances](#93-test-instances)
11. [ANNEXE : Dépannage](#annexe--dépannage)

---

## Architecture du lab

```
                ┌──────────────────────────────────────────┐
                │        Internet (NAT - VMnet8)           │
                │          10.10.10.0/24                   │
                └──────────────┬───────────────────────────┘
                               │ DHCP
                ┌──────────────┼───────────────────────────┐
                │              │                           │
          ┌─────▼──────┐  ┌───▼───────┐  ┌───▼───────┐  ┌▼──────────┐
          │ Controller │  │ Compute1  │  │ Compute2  │  │  Block1   │
          │ 10.0.0.11  │  │ 10.0.0.31 │  │ 10.0.0.32 │  │ 10.0.0.41 │
          │ (ens38)    │  │ (ens38)   │  │ (ens38)   │  │ (ens38)   │
          └─────┬──────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
                │               │              │              │
   ┌────────────┴───────────────┴──────────────┴──────────────┴────────┐
   │          Management Network (Host-Only - VMnet3)                  │
   │                    10.0.0.0/24                                    │
   └───────────────────────────────────────────────────────────────────┘

   ┌───────────────────────────────────────────────────────────────────┐
   │          Provider Network (Host-Only - VMnet2)                    │
   │                  203.0.113.0/24                                   │
   │    Controller: 203.0.113.11  |  Compute1: 203.0.113.31           │
   │    Compute2: 203.0.113.32    |  Block1: 203.0.113.41             │
   └───────────────────────────────────────────────────────────────────┘
```

### Services par nœud

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
┌─────────────┬─────────────────────────────────────────────────────────┐
│ Controller  │ MySQL, RabbitMQ, Keystone, Nova API, Glance, Horizon,   │
│             │ Neutron API, Placement, Cinder API, OVN Northd          │
├─────────────┼─────────────────────────────────────────────────────────┤
│ Compute1/2  │ nova-compute, OVN Controller, OVS, Metadata Agent       │
├─────────────┼─────────────────────────────────────────────────────────┤
│ Block1      │ cinder-volume (LVM), cinder-scheduler, cinder-backup    │
└─────────────┴─────────────────────────────────────────────────────────┘
```

**Caractéristiques techniques :**
- **OS** : Ubuntu Server 22.04 LTS
- **DevStack version** : 2026.1
- **Durée estimée** : 3–4 heures
- **Prérequis** : VMware Workstation, 32 Go RAM hôte minimum, 200 Go disque

---

## PARTIE 1 : Préparation VMware Workstation

### 1.1 Créer les réseaux virtuels

**Menu VMware** : `Edit → Virtual Network Editor → Run as Administrator`

Créer **3 réseaux** :

| Réseau | VMnet | Type | Subnet | Masque | DHCP |
|--------|-------|------|--------|--------|------|
| **Internet** | VMnet8 | NAT | 10.10.10.0 | 255.255.255.0 | Yes |
| **Provider** | VMnet2 | Host-Only | 203.0.113.0 | 255.255.255.0 | Yes |
| **Management** | VMnet3 | Host-Only | 10.0.0.0 | 255.255.255.0 | Yes |

**Résultat attendu** : 3 réseaux visibles dans Virtual Network Editor.

---

### 1.2 Créer les 4 VMs Ubuntu 22.04

#### Spécifications minimales

| VM | vCPU | RAM | Disque | Cartes réseau |
|----|------|-----|--------|---------------|
| **controller** | 4 | 8 Go | 60 Go | 3 (NAT + VMnet2 + VMnet3) |
| **compute1** | 4 | 8 Go | 50 Go | 3 (NAT + VMnet2 + VMnet3) |
| **compute2** | 4 | 8 Go | 50 Go | 3 (NAT + VMnet2 + VMnet3) |
| **block1** | 2 | 4 Go | 50 Go | 3 (NAT + VMnet2 + VMnet3) |

#### Procédure pour chaque VM

1. **File → New Virtual Machine → Custom**
2. Sélectionner **Ubuntu Server 22.04 ISO**
3. Configurer les ressources (vCPU, RAM, disque)
4. Dans **VM Settings → Network Adapter**, ajouter 3 cartes :
   - **Adapter 1** → NAT (VMnet8)
   - **Adapter 2** → Custom: VMnet2 (Provider)
   - **Adapter 3** → Custom: VMnet3 (Management)

#### Installation Ubuntu

- Utilisateur système : `stagiaire`
- Installer OpenSSH server : **Yes**
- Pas de packages supplémentaires
- Partitionnement : Disque entier (pas de LVM)

**Répéter** pour les 4 VMs : controller, compute1, compute2, block1.

---

## PARTIE 2 : Configuration réseau et système

### 2.1 Hostnames

Sur **chaque VM** (en tant que root ou avec sudo) :

```bash
# Sur Controller
sudo hostnamectl set-hostname controller

# Sur Compute1
sudo hostnamectl set-hostname compute1

# Sur Compute2
sudo hostnamectl set-hostname compute2

# Sur Block1
sudo hostnamectl set-hostname block1
```

Déconnecte-toi et reconnecte-toi pour voir le changement dans le prompt.

**Résultat attendu** : Le prompt affiche le nouveau hostname.

---

### 2.2 Identifier les interfaces

Sur chaque VM :

```bash
ip a
```

**Résultat attendu** :

- `lo` : loopback (127.0.0.1)
- `ens33` : NAT (DHCP automatique, ~10.10.10.x)
- `ens37` : Provider (pas d'IP configurée)
- `ens38` : Management (pas d'IP configurée)

⚠️ **Important** : Si les noms d'interfaces sont différents (ex: ens160, ens192), note-les et adapte tous les fichiers Netplan ci-dessous avec tes noms réels.

---

### 2.3 Plan d'adressage

| Nœud | Interface NAT (ens33) | Interface Provider (ens37) | Interface Management (ens38) |
|------|---------------------|--------------------------|----------------------------|
| controller | DHCP auto | 203.0.113.11/24 | 10.0.0.11/24 |
| compute1 | DHCP auto | 203.0.113.31/24 | 10.0.0.31/24 |
| compute2 | DHCP auto | 203.0.113.32/24 | 10.0.0.32/24 |
| block1 | DHCP auto | 203.0.113.41/24 | 10.0.0.41/24 |

**Rôle des réseaux :**

- **NAT (ens33)** : Accès Internet pour télécharger les packages
- **Provider (ens37)** : Réseau externe OpenStack (floating IPs)
- **Management (ens38)** : Communication inter-nœuds OpenStack

---

### 2.4 Configuration Netplan

#### Controller – /etc/netplan/01-netcfg.yaml

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

Remplacer le contenu par :

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
      nameservers:
        addresses: [8.8.8.8]
    ens36:
      dhcp4: false
      addresses: [203.0.113.11/24]
    ens38:
      dhcp4: false
      addresses: [10.0.0.11/24]

```

Appliquer :

```bash
sudo systemctl enable systemd-networkd
sudo netplan apply
ip addr show
```

**Résultat attendu** :

- `ens33` : IP ~10.10.10.x (DHCP)
- `ens37` : IP 203.0.113.11/24
- `ens38` : IP 10.0.0.11/24

---

#### Compute1 – /etc/netplan/01-netcfg.yaml

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
      nameservers:
        addresses: [8.8.8.8]
    ens36:
      dhcp4: false
      addresses: [203.0.113.31/24]
    ens38:
      dhcp4: false
      addresses: [10.0.0.31/24]

```

```bash
sudo netplan apply
ip addr show
```

---

#### Compute2 – /etc/netplan/01-netcfg.yaml

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
      nameservers:
        addresses: [8.8.8.8]
    ens36:
      dhcp4: false
      addresses: [203.0.113.32/24]
    ens38:
      dhcp4: false
      addresses: [10.0.0.32/24]

```

```bash
sudo netplan apply
ip addr show
```

---

#### Block1 – /etc/netplan/01-netcfg.yaml

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: true
      nameservers:
        addresses: [8.8.8.8]
    ens36:
      dhcp4: false
      addresses: [203.0.113.41/24]
    ens38:
      dhcp4: false
      addresses: [10.0.0.41/24]

```

```bash
sudo netplan apply
ip addr show
```

---

### 2.5 Configuration /etc/hosts

Sur toutes les VMs, éditer `/etc/hosts` :

```bash
sudo nano /etc/hosts
```

Ajouter (en conservant la ligne `127.0.0.1 localhost`) :

```
127.0.0.1   localhost

# Management Network OpenStack
10.0.0.11   controller
10.0.0.31   compute1
10.0.0.32   compute2
10.0.0.41   block1
```

Sauvegarder et quitter.

---

### 2.6 Tests de connectivité

Sur chaque VM, effectuer les tests suivants :

#### Test 1 : Accès Internet

```bash
ping -c 3 openstack.org
```

**Résultat attendu** : 3 réponses reçues.

#### Test 2 : Résolution par hostname

```bash
ping -c 3 controller
ping -c 3 compute1
ping -c 3 compute2
ping -c 3 block1
```

**Résultat attendu** : Toutes les commandes reçoivent des réponses.

#### Test 3 : Ping par IP Management

```bash
ping -c 3 10.0.0.11
ping -c 3 10.0.0.31
ping -c 3 10.0.0.32
ping -c 3 10.0.0.41
```

**Résultat attendu** : Toutes les IP Management répondent.

⚠️ **POINT DE CONTRÔLE** : Si un test échoue, ne pas continuer. Vérifier Netplan, /etc/hosts, et la configuration VMware (tous les nœuds doivent être sur VMnet2 et VMnet3).

---

## PARTIE 3 : Préparation DevStack

### 3.1 Créer l'utilisateur stack (sur les 4 VMs)

Sur chaque VM (controller, compute1, compute2, block1) :

```bash
sudo apt-get update
sudo apt-get install -y git sudo vim

sudo useradd -s /bin/bash -d /opt/stack -m stack
sudo chmod +x /opt/stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo passwd stack
```

Mot de passe à définir : **stack**

Puis devenir utilisateur stack :

```bash
sudo su - stack
pwd
```

**Résultat attendu** : `/opt/stack`

---

### 3.2 Configuration SSH (Controller → tous les nœuds)

Sur **controller uniquement**, en tant qu'utilisateur **stack** :

#### Générer la clé SSH

```bash
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
```

**Résultat attendu** : Clé créée dans `/opt/stack/.ssh/id_rsa`.

#### Copier la clé vers tous les nœuds

```bash
ssh-copy-id stack@controller
ssh-copy-id stack@compute1
ssh-copy-id stack@compute2
ssh-copy-id stack@block1
```

Mot de passe demandé : **stack** (pour chaque nœud)

#### Vérifier l'accès sans mot de passe

```bash
ssh stack@compute1 hostname
ssh stack@compute2 hostname
ssh stack@block1 hostname
```

**Résultat attendu** :

- Pas de mot de passe demandé
- Affichage des hostnames : compute1, compute2, block1

---

## PARTIE 4 : Installation DevStack – Controller

### 4.1 Télécharger DevStack

Sur **controller**, connecté en utilisateur **stack** :

```bash
cd /opt/stack
git clone https://opendev.org/openstack/devstack
cd devstack
```

**Résultat attendu** : Répertoire `/opt/stack/devstack` créé.

---

### 4.2 Fichier local.conf

```bash
[[local|localrc]]

HOST_IP=10.0.0.11

FIXED_RANGE=10.11.12.0/24
FLOATING_RANGE=10.0.0.200/27
PUBLIC_NETWORK_GATEWAY=10.0.0.1

ADMIN_PASSWORD=openstack
DATABASE_PASSWORD=openstack
RABBIT_PASSWORD=openstack
SERVICE_PASSWORD=openstack

LOGFILE=/opt/stack/logs/stack.sh.log
LOG_COLOR=False

# Services de base
ENABLED_SERVICES=mysql,rabbit,key

# Nova (sans n-cpu qui sera sur les compute nodes)
ENABLED_SERVICES+=,n-api,n-cond,n-sch,n-novnc,n-api-meta

# Placement
ENABLED_SERVICES+=,placement-api,placement-client

# Glance
ENABLED_SERVICES+=,g-api

# Cinder API + scheduler (obligatoire pour que block1 fonctionne)
ENABLED_SERVICES+=,c-api,c-sch

# Neutron avec OVN
ENABLED_SERVICES+=,q-svc,q-meta
ENABLED_SERVICES+=,ovn-controller,ovn-northd,ovs-vswitchd,ovsdb-server,q-ovn-metadata-agent

# Horizon
ENABLED_SERVICES+=,horizon

Q_USE_PROVIDERNET_FOR_PUBLIC=True
PUBLIC_INTERFACE=ens38

NOVA_VNC_ENABLED=True
VNCSERVER_LISTEN=0.0.0.0
VNCSERVER_PROXYCLIENT_ADDRESS=$HOST_IP

```

Vérifier le fichier :

```bash
cat local.conf | head -20
```

---

### 4.3 Installation

```bash
./stack.sh
```

**Durée** : 20–40 minutes selon la connexion Internet

**Résultat attendu en fin d'installation** :

```
This is your host IP address: 10.0.0.11
Horizon is now available at http://10.0.0.11/dashboard
Keystone is serving at http://10.0.0.11/identity/
The default users are: admin and demo
The password: openstack

DevStack Version: 2026.1
OS Version: Ubuntu 22.04 jammy
```

---

### 4.4 Vérifications

#### Vérifier les services OpenStack

```bash
source /opt/stack/devstack/openrc admin admin
openstack service list
```

**Résultat attendu** : Services keystone, nova, glance, neutron, placement, cinderv3 listés.

#### Vérifier les endpoints

```bash
openstack endpoint list
#modifier les adresse si vous les trouvez 127.0.0.1:60999 ou 9292 pour eviter les problème




```

**Résultat attendu** : Endpoints public, internal, admin pour chaque service avec IP 10.0.0.11.

#### Vérifier que Cinder API écoute

```bash
sudo ss -ltnp | grep 8776
sudo sed -i 's/http-socket = 127.0.0.1:60999/http-socket = 0.0.0.0:9292/' /etc/glance/glance-uwsgi.ini
grep http-socket /etc/glance/glance-uwsgi.ini
sudo systemctl restart devstack@g-api
sudo ss -ltnp | grep 9292
résultat pour ce ci LISTEN  0  100  0.0.0.0:9292  0.0.0.0:*  users:(("uwsgi",...))

```

**Résultat attendu** : Port 8776 en LISTEN sur 10.0.0.11.


---

## PARTIE 5 : Installation DevStack – Compute1

### 5.1 Télécharger DevStack

Sur **compute1**, en utilisateur **stack** :

```bash
cd /opt/stack
git clone https://opendev.org/openstack/devstack
cd devstack
```

---

### 5.2 Fichier local.conf

```bash
cat > local.conf << 'EOF'
[[local|localrc]]

HOST_IP=10.0.0.31

FIXED_RANGE=10.11.12.0/24
FLOATING_RANGE=10.0.0.200/27

ADMIN_PASSWORD=openstack
DATABASE_PASSWORD=openstack
RABBIT_PASSWORD=openstack
SERVICE_PASSWORD=openstack

LOGFILE=/opt/stack/logs/stack.sh.log

SERVICE_HOST=10.0.0.11
MYSQL_HOST=$SERVICE_HOST
RABBIT_HOST=$SERVICE_HOST
GLANCE_HOSTPORT=$SERVICE_HOST:9292
KEYSTONE_AUTH_HOST=$SERVICE_HOST
KEYSTONE_SERVICE_HOST=$SERVICE_HOST

ENABLED_SERVICES=n-cpu,placement-client,ovn-controller,ovs-vswitchd,ovsdb-server,q-ovn-metadata-agent

NOVA_VNC_ENABLED=True
NOVNCPROXY_URL="http://$SERVICE_HOST:6080/vnc_auto.html"
VNCSERVER_LISTEN=$HOST_IP
VNCSERVER_PROXYCLIENT_ADDRESS=$VNCSERVER_LISTEN
EOF


---

### 5.3 Installation

```bash
./stack.sh
```

**Durée** : 15–25 minutes

**Résultat attendu** :

```
This is your host IP address: 10.0.0.31
DevStack Version: 2026.1
OS Version: Ubuntu 22.04 jammy
```

---

## PARTIE 6 : Installation DevStack – Compute2

### 6.1 Télécharger DevStack

Sur **compute2**, en utilisateur **stack** :

```bash
cd /opt/stack
git clone https://opendev.org/openstack/devstack
cd devstack
```

---

### 6.2 Fichier local.conf

```bash
cat > local.conf << 'EOF'
[[local|localrc]]

# IP Management de Compute2
HOST_IP=10.0.0.32

# Réseaux internes (identiques au controller)
FIXED_RANGE=10.11.12.0/24
FLOATING_RANGE=10.0.0.200/27

# Mots de passe (identiques au controller)
ADMIN_PASSWORD=openstack
DATABASE_PASSWORD=openstack
RABBIT_PASSWORD=openstack
SERVICE_PASSWORD=openstack

LOGFILE=/opt/stack/logs/stack.sh.log

# Multi-nœud : pointer vers le controller
SERVICE_HOST=10.0.0.11
MYSQL_HOST=$SERVICE_HOST
RABBIT_HOST=$SERVICE_HOST
GLANCE_HOSTPORT=$SERVICE_HOST:9292
KEYSTONE_AUTH_HOST=$SERVICE_HOST
KEYSTONE_SERVICE_HOST=$SERVICE_HOST

# Services sur compute : Nova compute + OVN
ENABLED_SERVICES=n-cpu,placement-client,ovn-controller,ovs-vswitchd,ovsdb-server,q-ovn-metadata-agent

# Configuration VNC
NOVA_VNC_ENABLED=True
NOVNCPROXY_URL="http://$SERVICE_HOST:6080/vnc_auto.html"
VNCSERVER_LISTEN=$HOST_IP
VNCSERVER_PROXYCLIENT_ADDRESS=$VNCSERVER_LISTEN
EOF

```

---

### 6.3 Installation

```bash
./stack.sh
```

**Durée** : 15–25 minutes



---

## PARTIE 7 : Installation DevStack – Block1

### 7.1 Télécharger DevStack

Sur **block1**, en utilisateur **stack** :

```bash
cd /opt/stack
git clone https://opendev.org/openstack/devstack
cd devstack
```

---

### 7.2 Fichier local.conf

```bash
cat > local.conf << 'EOF'
[[local|localrc]]

# IP Management de block1
HOST_IP=10.0.0.41

# Mots de passe (identiques au controller)
ADMIN_PASSWORD=openstack
DATABASE_PASSWORD=openstack
RABBIT_PASSWORD=openstack
SERVICE_PASSWORD=openstack

LOGFILE=/opt/stack/logs/stack.sh.log

# Multi-nœud : pointer vers le controller
SERVICE_HOST=10.0.0.11
MYSQL_HOST=$SERVICE_HOST
RABBIT_HOST=$SERVICE_HOST
GLANCE_HOSTPORT=$SERVICE_HOST:9292
KEYSTONE_AUTH_HOST=$SERVICE_HOST
KEYSTONE_SERVICE_HOST=$SERVICE_HOST

# Services Cinder : UNIQUEMENT volume, scheduler, backup (PAS c-api)
ENABLED_SERVICES=c-vol,c-sch,c-bak

# Backend LVM pour Cinder
VOLUME_BACKING_FILE_SIZE=50G
VOLUME_BACKING_FILE=/opt/stack/data/stack-volumes-backing-file
VOLUME_GROUP=stack-volumes-lvm
EOF

```

**Note importante** : `c-api` n'est PAS dans ENABLED_SERVICES car il reste sur le controller.

---

### 7.3 Installation

```bash
./stack.sh
```

**Durée** : 10–20 minutes

**Résultat attendu** :

```
This is your host IP address: 10.0.0.41
DevStack Version: 2026.1
OS Version: Ubuntu 22.04 jammy
```

---

## PARTIE 8 : Enregistrement et vérifications

### 8.1 Enregistrer les compute nodes

Sur **controller** :

```bash
source /opt/stack/devstack/openrc admin admin
cd /opt/stack/devstack
./tools/discover_hosts.sh
```

**Résultat attendu** :

```
Discovering compute hosts...
Found 2 unmapped computes: compute1, compute2
```

Vérifier :

```bash
openstack compute service list
```

**Résultat attendu** :

```
+----+----------------+------------+----------+---------+-------+
| ID | Binary         | Host       | Zone     | Status  | State |
+----+----------------+------------+----------+---------+-------+
|  1 | nova-scheduler | controller | internal | enabled | up    |
|  2 | nova-conductor | controller | internal | enabled | up    |
|  3 | nova-compute   | compute1   | nova     | enabled | up    |
|  4 | nova-compute   | compute2   | nova     | enabled | up    |
+----+----------------+------------+----------+---------+-------+
```

---

### 8.2 Vérifier les agents réseau

```bash
openstack network agent list
```

**Résultat attendu** : Agents OVN Controller et Metadata pour controller, compute1, compute2.

---

### 8.3 Vérifier Cinder

```bash
openstack volume service list
```

**Résultat attendu** :

```
+------------------+------------------------+------+---------+-------+
| Binary           | Host                   | Zone | Status  | State |
+------------------+------------------------+------+---------+-------+
| cinder-scheduler | controller             | nova | enabled | up    |
| cinder-volume    | block1@lvm             | nova | enabled | up    |
| cinder-backup    | block1                 | nova | enabled | up    |
+------------------+------------------------+------+---------+-------+
```

---

## PARTIE 9 : Tests fonctionnels

### 9.1 Dashboard Horizon

Ouvrir un navigateur web et accéder à :

```
http://10.0.0.11/dashboard
```

**Connexion** :

- Domain: `default`
- User Name: `admin`
- Password: `openstack`

**Résultat attendu** : Dashboard OpenStack s'affiche. Dans Admin > Compute > Hypervisors, tu vois 2 hyperviseurs (compute1, compute2).

---

### 9.2 Test volume

Sur **controller** :

```bash
source /opt/stack/devstack/openrc admin admin
openstack volume create --size 2 test-volume
openstack volume list
```

**Résultat attendu** :

```
+--------------------------------------+-------------+-----------+------+-------------+
| ID                                   | Name        | Status    | Size | Attached to |
+--------------------------------------+-------------+-----------+------+-------------+
| xxxxx...                             | test-volume | available |    2 |             |
+--------------------------------------+-------------+-----------+------+-------------+
```

---

### 9.3 Test instances

```bash
# Créer le réseau privé
openstack network create private-net
openstack subnet create --network private-net --subnet-range 192.168.100.0/24 private-subnet

# Créer et configurer le routeur
openstack router create router1
openstack router add subnet router1 private-subnet
openstack router set --external-gateway public router1

# Créer une paire de clés SSH
openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey

# Créer une instance sur compute1
openstack server create   --flavor m1.tiny   --image cirros-0.6.2-x86_64-disk   --network private-net   --key-name mykey   --availability-zone nova:compute1   vm-compute1

# Créer une instance sur compute2
openstack server create   --flavor m1.tiny   --image cirros-0.6.2-x86_64-disk   --network private-net   --key-name mykey   --availability-zone nova:compute2   vm-compute2

# Vérifier les instances
openstack server list
```

**Résultat attendu** :

```
+--------------------------------------+-------------+--------+------------------------+
| ID                                   | Name        | Status | Networks               |
+--------------------------------------+-------------+--------+------------------------+
| xxxxx...                             | vm-compute1 | ACTIVE | private-net=192.168... |
| xxxxx...                             | vm-compute2 | ACTIVE | private-net=192.168... |
+--------------------------------------+-------------+--------+------------------------+
```

Les deux instances sont en statut ACTIVE, une sur chaque nœud compute.

---

## ANNEXE : Dépannage

### Problème : Ping entre VMs ne fonctionne pas

**Symptôme** : `ping controller` ou `ping 10.0.0.11` échoue depuis compute/block

**Solutions** :

- Vérifier la configuration IP : `ip addr show`, `ip route`
- Vérifier `/etc/hosts` sur toutes les VMs
- Vérifier que toutes les VMs sont sur les bons réseaux VMware :
  - Adapter 2 → VMnet2 (Provider)
  - Adapter 3 → VMnet3 (Management)
- Refaire `sudo netplan apply` sur les nœuds concernés
- Vérifier le firewall : `sudo ufw status` (doit être inactif ou bien configuré)

---

### Problème : DevStack échoue avec "No route to host"

**Symptôme** : Erreur `No route to host` ou `Connection refused` pendant `./stack.sh`

**Solutions** :

- Depuis le nœud qui échoue, tester : `ping 10.0.0.11`
- Vérifier que le service cible écoute sur controller : `sudo ss -ltnp | grep 8776` (pour Cinder)
- Vérifier `/etc/hosts` et résolution DNS
- Relancer DevStack sur controller si nécessaire

---

### Problème : Port 8776 (Cinder API) refuse les connexions

**Symptôme** : Block1 ne peut pas contacter Cinder API sur controller

**Solutions** :

- Sur controller, vérifier que `c-api` est dans ENABLED_SERVICES du local.conf
- Vérifier que le port écoute : `sudo ss -ltnp | grep 8776`
- Vérifier les logs : `tail -50 /opt/stack/logs/c-api.log`
- Si nécessaire, relancer DevStack sur controller : `cd /opt/stack/devstack && ./stack.sh`

---

### Problème : Compute node apparaît en "down"

**Symptôme** : `openstack compute service list` montre compute1 ou compute2 en état "down"

**Solutions** :

- Sur controller : `cd /opt/stack/devstack && ./tools/discover_hosts.sh`
- Attendre 30 secondes et vérifier à nouveau : `openstack compute service list`
- Sur le compute concerné, vérifier les services : `systemctl status devstack@n-cpu`
- Consulter les logs : `tail -50 /opt/stack/logs/n-cpu.log`

---

### Problème : Services OVN manquants

**Symptôme** : `openstack network agent list` ne montre pas les agents OVN

**Solutions** :

- Vérifier que `q-agt`, `q-l3`, `q-dhcp` ne sont PAS dans ENABLED_SERVICES (incompatibles avec OVN)
- Sur chaque nœud, vérifier : `systemctl status devstack@ovn-controller`
- Relancer DevStack si nécessaire après correction du local.conf

---

## FIN DU LAB

**Félicitations !** Vous disposez maintenant d'un cloud OpenStack multi-nœuds fonctionnel avec :

✅ 1 Controller (API, base de données, Cinder API)  
✅ 2 Compute nodes (nova-compute + OVN)  
✅ 1 Block storage (Cinder volume/backup LVM)

### Accès au cloud

**Dashboard Horizon** : http://10.0.0.11/dashboard  
**Identifiants** : admin / openstack

### Services disponibles

- Compute (Nova) avec 2 hyperviseurs
- Réseau (Neutron/OVN)
- Images (Glance)
- Stockage bloc (Cinder)
- Dashboard web (Horizon)

---

**Auteur** : LAB DevStack Multi-nœuds  
**Version** : 2026.1  
**Date** : Février 2026  
**Licence** : Usage pédagogique et formation

---

*Préparé à l'aide de Claude Sonnet 4.5*
