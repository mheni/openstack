# LAB 5 : Administration Avancée OpenStack – Cloud-init, Stockage, Réseau, Logs et Orchestration

---

##  Objectifs du LAB

À l'issue de ce LAB complet, vous serez capable de :

### 1. Images, métadonnées et personnalisation (cloud-init)
- Injecter des scripts **user-data** pour personnaliser les instances au boot.
- Gérer les **metadata** d'instances et comprendre le service metadata.
- Utiliser cloud-init pour installer des packages, créer des utilisateurs, configurer SSH.

### 2. Stockage avancé (Cinder)
- Créer et gérer des **snapshots** de volumes.
- Effectuer des **backups** et restaurations de volumes.
- Configurer et tester les **quotas Cinder** fins.
- Comprendre le cycle de vie complet d'un volume.

### 3. Réseau avancé et troubleshooting Neutron/OVN
- Analyser les **namespaces réseau** et les flux Est/Ouest et Nord/Sud.
- Utiliser **tcpdump** et les outils de debug réseau.
- Gérer les **security groups** complexes et politiques de routage.
- Diagnostiquer et résoudre des problèmes de connectivité réseau.

### 4. Supervision, logs et dépannage "pro"
- Naviguer dans les logs Nova, Neutron, Cinder, Keystone.
- Utiliser les commandes de diagnostic système (`systemctl`, `journalctl`).
- Simuler et résoudre des pannes (compute down, agent réseau down, volume en erreur).

### 5. Orchestration avec scripts avancés
- Créer des templates de déploiement réutilisables.
- Automatiser le déploiement de stacks complètes (multi-VM + réseau + volumes).
- Introduire les concepts Heat (si activé) ou équivalent scriptés.

---

##  Prérequis

Avant de commencer :

1. Les LABs 1–4 sont terminés (projets, réseau, flavors, instances, images).
2. DevStack multi‑nœuds fonctionnel (Controller + Compute1 + Compute2 + Block1).
3. Au moins une image Ubuntu cloud disponible (ex: `ubuntu-22.04-cloud`).
4. Accès SSH au controller et aux nœuds compute/block pour l'analyse des logs.

Vérifications initiales sur le **controller** :

```bash
source /opt/stack/devstack/openrc admin admin
openstack service list
openstack compute service list
openstack network agent list
openstack volume service list
```

Tous les services doivent être **enabled** et **up**.

---

## PARTIE 1 : Personnalisation d'instances avec cloud-init

### 1.1 Introduction à cloud-init

Cloud-init permet d'exécuter des scripts au premier boot d'une instance cloud. C'est le mécanisme standard pour :

- Créer des utilisateurs
- Installer des packages
- Configurer des services
- Injecter des clés SSH
- Exécuter des commandes arbitraires

### 1.2 Créer un script user-data

Sur le **controller**, créez un fichier `userdata-web.yaml` :

```bash
cat > userdata-web.yaml << 'EOF'
#cloud-config
hostname: web-server-f1

users:
  - name: stagiaire
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: sudo
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa AAAAB3... votre-cle-publique-ici

packages:
  - nginx
  - curl
  - git
  - vim

runcmd:
  - systemctl enable nginx
  - systemctl start nginx
  - echo "<h1>Formation OpenStack - Web Server</h1>" > /var/www/html/index.html
  - echo "Server deployed via cloud-init" >> /var/www/html/index.html

final_message: "Cloud-init configuration completed after $UPTIME seconds"
EOF
```

Remplacez la clé SSH par votre clé publique réelle (`~/.ssh/id_rsa.pub`).

### 1.3 Lancer une instance avec user-data

En tant que `user1` (Formation1) :

```bash
# Contexte user1
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

# Lancer l'instance avec user-data
openstack server create   --flavor demo.small   --image ubuntu-22.04-cloud   --network f1-net   --key-name mykey   --user-data userdata-web.yaml   f1-web-cloudinit

# Attendre que l'instance soit ACTIVE
openstack server list
```

### 1.4 Attribuer une floating IP et tester

```bash
# Créer et associer une floating IP
FIP=$(openstack floating ip create -f value -c floating_ip_address public)
openstack server add floating ip f1-web-cloudinit $FIP

echo "Floating IP: $FIP"
```

Attendez 2-3 minutes que cloud-init termine son exécution, puis testez :

```bash
# Test ping
ping -c 3 $FIP

# Test SSH
ssh stagiaire@$FIP

# Dans la VM, vérifier cloud-init
sudo cloud-init status
sudo cat /var/log/cloud-init.log

# Test web
curl http://$FIP
```

**Résultat attendu** : 

- Connexion SSH avec l'utilisateur `stagiaire`.
- Nginx installé et démarré automatiquement.
- Page web personnalisée accessible.

### 1.5 Exercice avancé : script avec paramètres

Créez un second user-data pour déployer une base de données :

```bash
cat > userdata-db.yaml << 'EOF'
#cloud-config
hostname: db-server-f1

packages:
  - mysql-server
  - python3-mysqldb

runcmd:
  - systemctl enable mysql
  - systemctl start mysql
  - mysql -e "CREATE DATABASE formation_db;"
  - mysql -e "CREATE USER 'formateur'@'%' IDENTIFIED BY 'passwordformation';"
  - mysql -e "GRANT ALL PRIVILEGES ON formation_db.* TO 'formateur'@'%';"
  - mysql -e "FLUSH PRIVILEGES;"
  - sed -i 's/127.0.0.1/0.0.0.0/' /etc/mysql/mysql.conf.d/mysqld.cnf
  - systemctl restart mysql

final_message: "MySQL Database ready for Formation1"
EOF

# Lancer l'instance DB
openstack server create   --flavor demo.medium   --image ubuntu-22.04-cloud   --network f1-net   --key-name mykey   --user-data userdata-db.yaml   f1-db-cloudinit
```

---

## PARTIE 2 : Stockage avancé Cinder

### 2.1 Snapshots de volumes

Créez un volume, attachez-le à une instance, écrivez des données, puis créez un snapshot :

```bash
source user1-openrc.sh

# Créer un volume
openstack volume create --size 5 f1-data-vol

# Attacher à une instance existante
openstack server add volume f1-web-cloudinit f1-data-vol

# Connectez-vous en SSH et montez le volume
ssh stagiaire@$FIP
sudo mkfs.ext4 /dev/vdb
sudo mkdir /data
sudo mount /dev/vdb /data
echo "Données importantes Formation1" | sudo tee /data/test.txt
sudo umount /data
exit

# Créer un snapshot du volume
openstack volume snapshot create --volume f1-data-vol f1-data-snapshot

# Vérifier
openstack volume snapshot list
openstack volume snapshot show f1-data-snapshot
```

### 2.2 Restauration depuis snapshot

```bash
# Créer un nouveau volume depuis le snapshot
openstack volume create --snapshot f1-data-snapshot --size 5 f1-data-restored

# Vérifier
openstack volume list
openstack volume show f1-data-restored
```

Vous pouvez maintenant attacher `f1-data-restored` à une autre instance et vérifier que les données sont présentes.

### 2.3 Backups de volumes

Les backups Cinder sont stockés dans un backend de backup (par défaut Swift ou NFS dans DevStack) :

```bash
source user1-openrc.sh

# Créer un backup du volume
openstack volume backup create --name f1-data-backup f1-data-vol

# Attendre que le backup soit disponible
openstack volume backup list

# Restaurer depuis backup
openstack volume backup restore f1-data-backup f1-data-vol-restored

# Vérifier
openstack volume list | grep restored
```

### 2.4 Gestion des quotas Cinder détaillés

En tant qu'admin, ajuster les quotas par type de ressource :

```bash
source /opt/stack/devstack/openrc admin admin

# Voir les quotas Cinder actuels pour Formation1
openstack volume quota show Formation1

# Définir des limites strictes
openstack volume quota set   --volumes 5   --snapshots 3   --backups 2   --gigabytes 20   --per-volume-gigabytes 10   Formation1

# Vérifier
openstack volume quota show Formation1
```

Test : en tant que `user1`, essayez de créer un 6e volume ou un snapshot au-delà de la limite.

---

## PARTIE 3 : Réseau avancé et troubleshooting Neutron/OVN

### 3.1 Exploration des namespaces réseau

Sur le **controller** (ou les compute nodes), en tant que root :

```bash
sudo su -

# Lister les namespaces réseau
ip netns list

# Les namespaces typiques :
# - qrouter-<UUID> : routeurs Neutron
# - qdhcp-<UUID> : agents DHCP (si applicable)
```

Exemple : inspecter un routeur :

```bash
# Identifier le namespace du routeur f1-router
ROUTER_ID=$(openstack router show f1-router -f value -c id)
ROUTER_NS="qrouter-$ROUTER_ID"

# Entrer dans le namespace du routeur
sudo ip netns exec $ROUTER_NS bash

# Vérifier les interfaces et routes
ip a
ip route
iptables -t nat -L -n

# Tester la connectivité depuis le routeur
ping -c 3 8.8.8.8
exit
```

### 3.2 Analyse des flux avec tcpdump

Capturer le trafic sur une interface du routeur :

```bash
# Sur le controller
sudo ip netns exec $ROUTER_NS tcpdump -i qr-+ -n icmp

# Dans un autre terminal, depuis une instance, pingez une IP externe
# Observez les paquets ICMP traverser le routeur
```

### 3.3 Debug de connectivité Est/Ouest (VM à VM)

Créez deux instances dans Formation1 et testez le ping entre elles :

```bash
source user1-openrc.sh

# Récupérer les IPs privées
openstack server list -c Name -c Networks

# Depuis f1-web-cloudinit, ping vers f1-db-cloudinit
ssh stagiaire@<FIP_WEB>
ping <IP_PRIVEE_DB>
```

Si le ping échoue :

- Vérifier les **security groups** (autoriser ICMP).
- Vérifier que les deux VMs sont sur le même réseau.
- Utiliser `tcpdump` dans les namespaces pour tracer les paquets.

### 3.4 Security groups complexes

Créer un security group pour une architecture 3-tiers :

```bash
source user1-openrc.sh

# Security group Web (HTTP + HTTPS + SSH)
openstack security group create web-tier --description "Web tier access"
openstack security group rule create --proto tcp --dst-port 22 web-tier
openstack security group rule create --proto tcp --dst-port 80 web-tier
openstack security group rule create --proto tcp --dst-port 443 web-tier
openstack security group rule create --proto icmp web-tier

# Security group App (accès depuis Web uniquement)
openstack security group create app-tier --description "App tier access"
openstack security group rule create --proto tcp --dst-port 22 app-tier
openstack security group rule create --proto tcp --dst-port 8080   --remote-group web-tier app-tier

# Security group DB (accès depuis App uniquement)
openstack security group create db-tier --description "DB tier access"
openstack security group rule create --proto tcp --dst-port 3306   --remote-group app-tier db-tier

# Lister les security groups
openstack security group list
```

Appliquez ces security groups à vos instances selon leur rôle.

### 3.5 Test de MTU et fragmentation

```bash
# Depuis une instance, tester MTU
ssh stagiaire@<FIP>
ping -M do -s 1472 8.8.8.8   # Test avec MTU 1500
ping -M do -s 8972 8.8.8.8   # Test avec MTU > 9000 (jumbo frames)
```

---

## PARTIE 4 : Supervision, logs et dépannage "pro"

### 4.1 Navigation dans les logs OpenStack

Sur le **controller** :

```bash
cd /opt/stack/logs

# Logs principaux
ls -lh *.log

# Nova (compute API, scheduler, conductor)
tail -50 n-api.log
tail -50 n-sch.log

# Neutron (API, metadata, OVN)
tail -50 q-svc.log
tail -50 q-meta.log

# Cinder (API, volume, scheduler)
tail -50 c-api.log

# Keystone (authentification)
tail -50 key.log

# Rechercher des erreurs
grep -i error *.log | tail -20
grep -i traceback *.log | tail -20
```

Sur les **compute nodes** :

```bash
ssh stack@compute1
cd /opt/stack/logs
tail -50 n-cpu.log
tail -50 ovn-controller.log
```

Sur **block1** :

```bash
ssh stack@block1
cd /opt/stack/logs
tail -50 c-vol.log
tail -50 c-bak.log
```

### 4.2 Utilisation de systemctl et journalctl

Vérifier l'état des services DevStack :

```bash
# Sur le controller
systemctl list-units devstack@* --all

# Vérifier un service spécifique
systemctl status devstack@n-api

# Logs système pour un service
journalctl -u devstack@n-api -n 50

# Suivre les logs en temps réel
journalctl -u devstack@q-svc -f
```

### 4.3 Simulation de panne : Compute node down

Sur **compute1**, arrêter le service nova-compute :

```bash
ssh stack@compute1
systemctl stop devstack@n-cpu
```

Sur le controller, vérifier :

```bash
source /opt/stack/devstack/openrc admin admin
openstack compute service list

# compute1 doit apparaître en "down" après ~60 secondes
```

Tentez de créer une instance ciblant compute1 :

```bash
openstack server create   --flavor demo.tiny   --image cirros-0.6.2-x86_64-disk   --network f1-net   --availability-zone nova:compute1   test-compute-down
```

**Résultat** : l'instance reste en `BUILD` ou `ERROR`.

Résolution :

```bash
ssh stack@compute1
systemctl start devstack@n-cpu
systemctl status devstack@n-cpu

# Retour sur le controller
openstack compute service list
# compute1 repasse en "up"
```

### 4.4 Simulation de panne : Agent réseau OVN down

Sur **compute2**, arrêter OVN controller :

```bash
ssh stack@compute2
systemctl stop devstack@ovn-controller
```

Sur le controller :

```bash
openstack network agent list
# L'agent OVN de compute2 passe en "down"
```

Impact : les nouvelles instances sur compute2 peuvent avoir des problèmes réseau.

Résolution :

```bash
ssh stack@compute2
systemctl start devstack@ovn-controller
```

### 4.5 Simulation de panne : Volume en erreur

Créer un volume et forcer son état en erreur (simulation) :

```bash
source user1-openrc.sh
openstack volume create --size 1 test-error-vol

# En admin, forcer l'état error
source /opt/stack/devstack/openrc admin admin
openstack volume set --state error test-error-vol

# Vérifier
openstack volume list --all-projects | grep test-error

# Résolution : réinitialiser l'état
openstack volume set --state available test-error-vol
```

---

## PARTIE 5 : Orchestration avec scripts avancés

### 5.1 Template de déploiement complet

Créez un script réutilisable pour déployer une stack 3-tiers :

```bash
cat > deploy_3tier_stack.sh << 'EOF'
#!/usr/bin/env bash
set -e

# Configuration
PROJECT_NAME=${1:-Formation1}
STACK_NAME=${2:-stack-3tier}
IMG="ubuntu-22.04-cloud"
KEY="mykey"
NET_NAME="${PROJECT_NAME,,}-net"  # formation1-net

# Source credentials
source ~/user1-openrc.sh

echo "=== Déploiement de la stack $STACK_NAME pour $PROJECT_NAME ==="

# 1. Créer security groups si nécessaire
for SG in web-tier app-tier db-tier; do
  if ! openstack security group show $SG &>/dev/null; then
    openstack security group create $SG --description "$SG security group"
    case $SG in
      web-tier)
        openstack security group rule create --proto tcp --dst-port 22 $SG
        openstack security group rule create --proto tcp --dst-port 80 $SG
        openstack security group rule create --proto icmp $SG
        ;;
      app-tier)
        openstack security group rule create --proto tcp --dst-port 22 $SG
        openstack security group rule create --proto tcp --dst-port 8080 $SG
        ;;
      db-tier)
        openstack security group rule create --proto tcp --dst-port 3306 $SG
        ;;
    esac
  fi
done

# 2. Lancer les instances
echo "[INFO] Lancement du tier Web"
openstack server create   --flavor demo.small   --image "$IMG"   --network "$NET_NAME"   --key-name "$KEY"   --security-group web-tier   --availability-zone nova:compute1   --user-data userdata-web.yaml   ${STACK_NAME}-web || true

echo "[INFO] Lancement du tier App"
openstack server create   --flavor demo.medium   --image "$IMG"   --network "$NET_NAME"   --key-name "$KEY"   --security-group app-tier   --availability-zone nova:compute2   ${STACK_NAME}-app || true

echo "[INFO] Lancement du tier DB"
openstack server create   --flavor demo.large   --image "$IMG"   --network "$NET_NAME"   --key-name "$KEY"   --security-group db-tier   --availability-zone nova:compute2   --user-data userdata-db.yaml   ${STACK_NAME}-db || true

# 3. Attendre que les instances soient ACTIVE
echo "[INFO] Attente de disponibilité des instances..."
sleep 30

# 4. Associer floating IP au web
WEB_FIP=$(openstack floating ip create -f value -c floating_ip_address public)
openstack server add floating ip ${STACK_NAME}-web $WEB_FIP

echo ""
echo "=== Stack $STACK_NAME déployée avec succès ==="
echo "Web Tier: ${STACK_NAME}-web -> $WEB_FIP"
echo "App Tier: ${STACK_NAME}-app"
echo "DB Tier: ${STACK_NAME}-db"
echo ""
echo "Testez avec: curl http://$WEB_FIP"
EOF

chmod +x deploy_3tier_stack.sh
```

### 5.2 Exécution du script

```bash
./deploy_3tier_stack.sh Formation1 demo-app
```

**Résultat attendu** : 3 VMs déployées avec leurs security groups, user-data appliqué, floating IP associée au web.

### 5.3 Script de nettoyage

```bash
cat > cleanup_stack.sh << 'EOF'
#!/usr/bin/env bash
STACK_NAME=${1:-stack-3tier}
source ~/user1-openrc.sh

echo "=== Nettoyage de la stack $STACK_NAME ==="

# Supprimer les instances
for VM in ${STACK_NAME}-web ${STACK_NAME}-app ${STACK_NAME}-db; do
  if openstack server show $VM &>/dev/null; then
    echo "[INFO] Suppression de $VM"
    openstack server delete $VM
  fi
done

echo "=== Stack $STACK_NAME nettoyée ==="
EOF

chmod +x cleanup_stack.sh
```

### 5.4 Introduction à Heat (si activé)

Si Heat est activé dans votre DevStack, vous pouvez créer un template HOT (Heat Orchestration Template) :

```yaml
# Fichier stack-3tier.yaml
heat_template_version: 2021-04-16

description: Stack 3-tiers Formation OpenStack

parameters:
  key_name:
    type: string
    description: Keypair SSH
    default: mykey

  image:
    type: string
    description: Image ID
    default: ubuntu-22.04-cloud

  network:
    type: string
    description: Network name
    default: f1-net

resources:
  web_instance:
    type: OS::Nova::Server
    properties:
      name: heat-web
      flavor: demo.small
      image: { get_param: image }
      key_name: { get_param: key_name }
      networks:
        - network: { get_param: network }
      security_groups:
        - web-tier

  app_instance:
    type: OS::Nova::Server
    properties:
      name: heat-app
      flavor: demo.medium
      image: { get_param: image }
      key_name: { get_param: key_name }
      networks:
        - network: { get_param: network }
      security_groups:
        - app-tier

  db_instance:
    type: OS::Nova::Server
    properties:
      name: heat-db
      flavor: demo.large
      image: { get_param: image }
      key_name: { get_param: key_name }
      networks:
        - network: { get_param: network }
      security_groups:
        - db-tier

outputs:
  web_ip:
    description: IP privée du serveur web
    value: { get_attr: [web_instance, first_address] }
```

Déploiement :

```bash
openstack stack create -t stack-3tier.yaml formation-stack
openstack stack list
openstack stack show formation-stack
```

---

## 🔎 Questions de validation

### Cloud-init
1. Que permet d'automatiser cloud-init au démarrage d'une instance ?
2. Comment vérifier l'exécution de cloud-init dans une VM ?

### Stockage
3. Quelle est la différence entre un snapshot et un backup de volume ?
4. Pourquoi détacher un volume avant de créer un snapshot ?

### Réseau
5. À quoi servent les namespaces réseau dans OpenStack ?
6. Comment diagnostiquer un problème de connectivité entre deux VMs ?

### Logs
7. Où se trouvent les logs des services OpenStack dans DevStack ?
8. Quelle commande permet de suivre les logs d'un service en temps réel ?

### Orchestration
9. Quel est l'avantage d'utiliser un script d'orchestration ?
10. Qu'apporte Heat par rapport à un script shell ?

---

## ✅ Bilan du LAB 5

Dans ce LAB avancé, vous avez maîtrisé :

- La **personnalisation automatique** des instances avec cloud-init.
- Les **opérations avancées de stockage** (snapshots, backups, restauration).
- Le **troubleshooting réseau** approfondi (namespaces, tcpdump, security groups complexes).
- L'**analyse des logs** et la résolution de pannes réelles (compute down, agent réseau, volumes en erreur).
- L'**orchestration de déploiements** avec scripts réutilisables et introduction à Heat.

Vous disposez maintenant d'un niveau d'administration OpenStack avancé, proche des besoins de production et des certifications COA (Certified OpenStack Administrator).

---

**Auteur** : Dr. Ing.Maher HENI -LAB DevStack Multi-nœuds – Administration Avancée  
**Version** : 1.0  
**Date** : Février 2026  
**Licence** : Usage pédagogique et formation
