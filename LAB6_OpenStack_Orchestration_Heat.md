# LAB 6 : Orchestration OpenStack avec Heat

---

##  Objectifs du LAB

À l'issue de ce LAB, vous serez capable de :

- Comprendre les **concepts fondamentaux** de Heat (stacks, templates, ressources).
- Activer et configurer le service **Heat** dans DevStack.
- Créer des **templates HOT** (Heat Orchestration Template) pour orchestrer des déploiements.
- Déployer des stacks complètes (réseau + instances + volumes + security groups).
- Gérer le **cycle de vie** d'un stack (création, mise à jour, suppression).
- Utiliser les **paramètres** et **outputs** pour rendre les templates réutilisables.
- Déboguer et résoudre les erreurs de déploiement Heat.

Ce LAB s'appuie sur l'architecture DevStack multi‑nœuds existante et les projets/réseaux créés dans les LABs précédents.

---

##  Prérequis

Avant de commencer :

1. Les LABs 1–5 sont terminés (projets, réseau, flavors, images, instances).
2. DevStack multi‑nœuds fonctionnel (Controller + Compute1 + Compute2 + Block1).
3. Accès SSH au controller avec l'utilisateur **stack**.
4. Une image Ubuntu cloud disponible (ex: `ubuntu-22.04-cloud`).
5. Au moins un réseau privé configuré (ex: `f1-net`).

---

## PARTIE 1 : Installation et activation de Heat dans DevStack

### 1.1 Vérifier si Heat est déjà installé

Sur le **controller**, en tant que **stack** :

```bash
source /opt/stack/devstack/openrc admin admin
openstack service list | grep orchestration
```

Si Heat n'apparaît pas, vous devez l'activer dans DevStack.

### 1.2 Activer Heat dans DevStack

Si Heat n'est pas installé, ajoutez-le à la configuration DevStack :

```bash
cd /opt/stack/devstack

# Backup du local.conf actuel
cp local.conf local.conf.backup

# Ajouter Heat aux services
cat >> local.conf << 'EOF'

# Heat - Orchestration Service
enable_service heat h-api h-api-cfn h-eng
EOF

# Relancer DevStack pour installer Heat
./stack.sh
```

**Durée** : 10-20 minutes pour l'ajout de Heat.

### 1.3 Vérification de l'installation

```bash
source /opt/stack/devstack/openrc admin admin

# Vérifier le service Heat
openstack service list | grep orchestration

# Vérifier les endpoints
openstack endpoint list | grep orchestration

# Vérifier les services Heat
systemctl status devstack@h-eng
systemctl status devstack@h-api

# Tester la commande Heat
openstack stack list
```

**Résultat attendu** : 

- Service `heat` et `heat-cfn` présents.
- Endpoints pour orchestration.
- Commande `openstack stack list` fonctionne (liste vide au début).

---

## PARTIE 2 : Concepts fondamentaux de Heat

### 2.1 Architecture Heat

Heat est composé de plusieurs services :

- **heat-engine** : moteur d'orchestration qui gère les stacks.
- **heat-api** : API REST pour gérer les stacks.
- **heat-api-cfn** : API compatible AWS CloudFormation.

### 2.2 Éléments d'un template HOT

Un template Heat contient :

- **heat_template_version** : version du format HOT.
- **description** : description du template.
- **parameters** : paramètres d'entrée (configurables).
- **resources** : ressources OpenStack à créer.
- **outputs** : valeurs retournées après déploiement.

### 2.3 Exemple minimal

Créez un premier template simple :

```bash
cd ~
mkdir heat-templates
cd heat-templates

cat > simple-instance.yaml << 'EOF'
heat_template_version: 2021-04-16

description: Template simple pour créer une instance

parameters:
  key_name:
    type: string
    description: Nom de la keypair SSH
    default: mykey

  image:
    type: string
    description: Image à utiliser
    default: cirros-0.6.2-x86_64-disk

  flavor:
    type: string
    description: Flavor de l'instance
    default: demo.tiny

  network:
    type: string
    description: Réseau privé
    default: f1-net

resources:
  my_instance:
    type: OS::Nova::Server
    properties:
      name: heat-simple-vm
      flavor: { get_param: flavor }
      image: { get_param: image }
      key_name: { get_param: key_name }
      networks:
        - network: { get_param: network }

outputs:
  instance_ip:
    description: Adresse IP de l'instance
    value: { get_attr: [my_instance, first_address] }

  instance_name:
    description: Nom de l'instance
    value: { get_attr: [my_instance, name] }
EOF
```

---

## PARTIE 3 : Déploiement d'un premier stack

### 3.1 Contexte utilisateur Formation1

```bash
# Contexte user1
cat > ~/user1-openrc.sh << 'EOF'
export OS_AUTH_URL=http://10.0.0.11/identity
export OS_PROJECT_NAME=Formation1
export OS_USERNAME=user1
export OS_PASSWORD=user1pass
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_IDENTITY_API_VERSION=3
EOF

source ~/user1-openrc.sh
```

### 3.2 Créer le stack

```bash
cd ~/heat-templates

# Déployer le stack
openstack stack create -t simple-instance.yaml formation-stack-01

# Vérifier l'état
openstack stack list
openstack stack show formation-stack-01
```

**Résultat attendu** : Stack en statut `CREATE_IN_PROGRESS` puis `CREATE_COMPLETE`.

### 3.3 Explorer les ressources du stack

```bash
# Lister les ressources créées par le stack
openstack stack resource list formation-stack-01

# Afficher les outputs
openstack stack output list formation-stack-01
openstack stack output show formation-stack-01 instance_ip

# Vérifier que l'instance existe
openstack server list | grep heat-simple-vm
```

### 3.4 Accéder aux logs et événements

```bash
# Voir les événements du stack
openstack stack event list formation-stack-01

# Voir les détails d'une ressource
openstack stack resource show formation-stack-01 my_instance
```

---

## PARTIE 4 : Template complet avec réseau et sécurité

### 4.1 Créer un template avancé

```bash
cat > web-stack-complete.yaml << 'EOF'
heat_template_version: 2021-04-16

description: Stack complète Web avec réseau et security group

parameters:
  key_name:
    type: string
    description: Keypair SSH
    default: mykey

  image:
    type: string
    description: Image Ubuntu cloud
    default: ubuntu-22.04-cloud

  flavor:
    type: string
    description: Flavor pour le serveur web
    default: demo.small

  network_name:
    type: string
    description: Nom du réseau privé existant
    default: f1-net

  public_network:
    type: string
    description: Réseau public pour floating IP
    default: public

resources:
  # Security Group
  web_security_group:
    type: OS::Neutron::SecurityGroup
    properties:
      name: heat-web-sg
      description: Security group pour serveur web Heat
      rules:
        - protocol: tcp
          port_range_min: 22
          port_range_max: 22
          remote_ip_prefix: 0.0.0.0/0
        - protocol: tcp
          port_range_min: 80
          port_range_max: 80
          remote_ip_prefix: 0.0.0.0/0
        - protocol: tcp
          port_range_min: 443
          port_range_max: 443
          remote_ip_prefix: 0.0.0.0/0
        - protocol: icmp
          remote_ip_prefix: 0.0.0.0/0

  # Instance Web
  web_instance:
    type: OS::Nova::Server
    properties:
      name: heat-web-server
      flavor: { get_param: flavor }
      image: { get_param: image }
      key_name: { get_param: key_name }
      networks:
        - network: { get_param: network_name }
      security_groups:
        - { get_resource: web_security_group }
      user_data: |
        #!/bin/bash
        apt-get update
        apt-get install -y nginx
        systemctl enable nginx
        systemctl start nginx
        echo "<h1>Heat Orchestration - Formation OpenStack</h1>" > /var/www/html/index.html
        echo "<p>Stack deployed at $(date)</p>" >> /var/www/html/index.html

  # Floating IP
  web_floating_ip:
    type: OS::Neutron::FloatingIP
    properties:
      floating_network: { get_param: public_network }

  # Association Floating IP
  web_floating_ip_assoc:
    type: OS::Nova::FloatingIPAssociation
    properties:
      floating_ip: { get_resource: web_floating_ip }
      server_id: { get_resource: web_instance }

outputs:
  web_url:
    description: URL du serveur web
    value:
      str_replace:
        template: http://FLOATING_IP
        params:
          FLOATING_IP: { get_attr: [web_floating_ip, floating_ip_address] }

  floating_ip:
    description: Adresse IP publique
    value: { get_attr: [web_floating_ip, floating_ip_address] }

  private_ip:
    description: Adresse IP privée
    value: { get_attr: [web_instance, first_address] }
EOF
```

### 4.2 Déployer le stack complet

```bash
source ~/user1-openrc.sh

# Déployer
openstack stack create -t web-stack-complete.yaml formation-web-stack

# Suivre la progression
watch -n 5 openstack stack list

# Une fois CREATE_COMPLETE, afficher les outputs
openstack stack output show formation-web-stack web_url
openstack stack output show formation-web-stack floating_ip
```

### 4.3 Tester le serveur web

```bash
# Récupérer l'URL
WEB_URL=$(openstack stack output show formation-web-stack web_url -f value -c output_value)

echo "URL: $WEB_URL"

# Attendre que cloud-init termine (2-3 minutes)
sleep 180

# Tester
curl $WEB_URL

# Tester SSH
FIP=$(openstack stack output show formation-web-stack floating_ip -f value -c output_value)
ping -c 3 $FIP
ssh ubuntu@$FIP
```

---

## PARTIE 5 : Stack 3-tiers avec Heat

### 5.1 Template architecture 3-tiers

```bash
cat > stack-3tier.yaml << 'EOF'
heat_template_version: 2021-04-16

description: Architecture 3-tiers complète (Web + App + DB)

parameters:
  key_name:
    type: string
    description: Keypair SSH
    default: mykey

  image:
    type: string
    description: Image Ubuntu
    default: ubuntu-22.04-cloud

  network_name:
    type: string
    description: Réseau privé
    default: f1-net

  public_network:
    type: string
    description: Réseau public
    default: public

resources:
  # Security Groups
  web_sg:
    type: OS::Neutron::SecurityGroup
    properties:
      name: heat-web-tier
      description: Web tier security group
      rules:
        - protocol: tcp
          port_range_min: 22
          port_range_max: 22
        - protocol: tcp
          port_range_min: 80
          port_range_max: 80
        - protocol: icmp

  app_sg:
    type: OS::Neutron::SecurityGroup
    properties:
      name: heat-app-tier
      description: App tier security group
      rules:
        - protocol: tcp
          port_range_min: 22
          port_range_max: 22
        - protocol: tcp
          port_range_min: 8080
          port_range_max: 8080
          remote_group: { get_resource: web_sg }

  db_sg:
    type: OS::Neutron::SecurityGroup
    properties:
      name: heat-db-tier
      description: DB tier security group
      rules:
        - protocol: tcp
          port_range_min: 3306
          port_range_max: 3306
          remote_group: { get_resource: app_sg }
        - protocol: tcp
          port_range_min: 22
          port_range_max: 22

  # Instances
  web_server:
    type: OS::Nova::Server
    properties:
      name: heat-3tier-web
      flavor: demo.small
      image: { get_param: image }
      key_name: { get_param: key_name }
      networks:
        - network: { get_param: network_name }
      security_groups:
        - { get_resource: web_sg }
      availability_zone: nova:compute1
      user_data: |
        #!/bin/bash
        apt-get update
        apt-get install -y nginx
        systemctl enable nginx
        systemctl start nginx
        echo "<h1>3-Tier Architecture - Web Tier</h1>" > /var/www/html/index.html

  app_server:
    type: OS::Nova::Server
    properties:
      name: heat-3tier-app
      flavor: demo.medium
      image: { get_param: image }
      key_name: { get_param: key_name }
      networks:
        - network: { get_param: network_name }
      security_groups:
        - { get_resource: app_sg }
      availability_zone: nova:compute2

  db_server:
    type: OS::Nova::Server
    properties:
      name: heat-3tier-db
      flavor: demo.large
      image: { get_param: image }
      key_name: { get_param: key_name }
      networks:
        - network: { get_param: network_name }
      security_groups:
        - { get_resource: db_sg }
      availability_zone: nova:compute2
      user_data: |
        #!/bin/bash
        apt-get update
        apt-get install -y mysql-server
        systemctl enable mysql
        systemctl start mysql

  # Floating IP pour Web
  web_floating_ip:
    type: OS::Neutron::FloatingIP
    properties:
      floating_network: { get_param: public_network }

  web_floating_assoc:
    type: OS::Nova::FloatingIPAssociation
    properties:
      floating_ip: { get_resource: web_floating_ip }
      server_id: { get_resource: web_server }

  # Volume pour DB
  db_volume:
    type: OS::Cinder::Volume
    properties:
      name: heat-db-data
      size: 10

  db_volume_attachment:
    type: OS::Cinder::VolumeAttachment
    properties:
      volume_id: { get_resource: db_volume }
      instance_uuid: { get_resource: db_server }

outputs:
  web_public_ip:
    description: IP publique du serveur Web
    value: { get_attr: [web_floating_ip, floating_ip_address] }

  web_private_ip:
    description: IP privée du serveur Web
    value: { get_attr: [web_server, first_address] }

  app_private_ip:
    description: IP privée du serveur App
    value: { get_attr: [app_server, first_address] }

  db_private_ip:
    description: IP privée du serveur DB
    value: { get_attr: [db_server, first_address] }

  db_volume_id:
    description: ID du volume DB
    value: { get_resource: db_volume }
EOF
```

### 5.2 Déployer le stack 3-tiers

```bash
source ~/user1-openrc.sh

# Déployer
openstack stack create -t stack-3tier.yaml formation-3tier

# Suivre la création
watch -n 5 'openstack stack list && openstack stack resource list formation-3tier'

# Afficher tous les outputs
openstack stack output list formation-3tier
openstack stack output show formation-3tier web_public_ip
```

---

## PARTIE 6 : Mise à jour et gestion des stacks

### 6.1 Mise à jour d'un stack

Modifiez le template pour ajouter une règle de security group :

```bash
# Modifier le template web-stack-complete.yaml
# Ajouter une règle HTTPS (déjà présente dans notre exemple)

# Mettre à jour le stack
openstack stack update -t web-stack-complete.yaml formation-web-stack

# Suivre la mise à jour
openstack stack event list formation-web-stack
openstack stack show formation-web-stack
```

### 6.2 Paramètres en ligne de commande

Surcharger les paramètres par défaut :

```bash
openstack stack create -t simple-instance.yaml   --parameter "flavor=demo.medium"   --parameter "image=ubuntu-22.04-cloud"   my-custom-stack
```

### 6.3 Fichier de paramètres (environment file)

Créer un fichier d'environnement :

```bash
cat > env-production.yaml << 'EOF'
parameters:
  flavor: demo.large
  image: ubuntu-22.04-cloud
  network_name: f1-net
  key_name: mykey
EOF

# Déployer avec environment
openstack stack create -t web-stack-complete.yaml   -e env-production.yaml   production-stack
```

---

## PARTIE 7 : Dépannage et résolution d'erreurs

### 7.1 Stack en erreur

```bash
# Voir l'état détaillé
openstack stack show <stack-name>

# Voir les événements pour identifier l'erreur
openstack stack event list <stack-name> --nested-depth 5

# Voir les ressources en erreur
openstack stack resource list <stack-name> --nested-depth 5 | grep FAILED
```

### 7.2 Logs Heat

Sur le controller :

```bash
cd /opt/stack/logs
tail -50 h-eng.log
tail -50 h-api.log

# Rechercher des erreurs
grep -i error h-eng.log | tail -20
grep -i traceback h-eng.log | tail -30
```

### 7.3 Validation d'un template

Avant de déployer, valider la syntaxe :

```bash
openstack orchestration template validate -t stack-3tier.yaml
```

### 7.4 Suppression d'un stack bloqué

```bash
# Tentative de suppression normale
openstack stack delete <stack-name>

# Si bloqué, forcer l'abandon
openstack stack abandon <stack-name>

# Ou suppression forcée (admin)
source /opt/stack/devstack/openrc admin admin
openstack stack delete --yes <stack-name>
```

---

## PARTIE 8 : Patterns avancés et bonnes pratiques

### 8.1 Utilisation de depends_on

Contrôler l'ordre de création :

```yaml
resources:
  my_instance:
    type: OS::Nova::Server
    depends_on: db_volume
    properties:
      # ...

  db_volume:
    type: OS::Cinder::Volume
    properties:
      # ...
```

### 8.2 Conditions

```yaml
parameters:
  create_volume:
    type: boolean
    default: false

conditions:
  volume_enabled: { equals: [{get_param: create_volume}, true] }

resources:
  my_volume:
    type: OS::Cinder::Volume
    condition: volume_enabled
    properties:
      size: 10
```

### 8.3 Utilisation de get_attr et get_resource

```yaml
outputs:
  instance_info:
    value:
      str_replace:
        template: "Server NAME is available at IP"
        params:
          NAME: { get_attr: [my_instance, name] }
          IP: { get_attr: [my_instance, first_address] }
```

### 8.4 Templates imbriqués (nested stacks)

Créer un template réutilisable pour un serveur :

```bash
cat > server-template.yaml << 'EOF'
heat_template_version: 2021-04-16

parameters:
  server_name:
    type: string
  flavor:
    type: string
  image:
    type: string
  network:
    type: string
  security_group:
    type: string

resources:
  server:
    type: OS::Nova::Server
    properties:
      name: { get_param: server_name }
      flavor: { get_param: flavor }
      image: { get_param: image }
      networks:
        - network: { get_param: network }
      security_groups:
        - { get_param: security_group }

outputs:
  server_ip:
    value: { get_attr: [server, first_address] }
EOF
```

L'utiliser dans un template parent :

```yaml
resources:
  web_server:
    type: server-template.yaml
    properties:
      server_name: web-01
      flavor: demo.small
      image: ubuntu-22.04-cloud
      network: f1-net
      security_group: web-sg
```

---

## PARTIE 9 : Exercices pratiques

### Exercice 1 : Stack avec réseau complet

Créer un template qui :
- Crée un nouveau réseau privé
- Crée un subnet avec DHCP
- Crée un routeur et le connecte au réseau public
- Lance 2 instances sur ce réseau
- Associe une floating IP à la première instance

### Exercice 2 : Stack avec scaling

Créer un template utilisant `OS::Heat::ResourceGroup` pour déployer N instances identiques.

### Exercice 3 : Stack avec auto-healing

Utiliser `OS::Heat::AutoScalingGroup` pour créer un groupe d'instances qui se recréent automatiquement en cas de panne.

---

##  Questions de validation

1. Quelle est la différence entre Heat et un simple script shell ?
2. Qu'est-ce qu'un template HOT et quels sont ses éléments principaux ?
3. Comment passer des paramètres à un stack Heat ?
4. Que se passe-t-il quand on met à jour un stack existant ?
5. Comment déboguer un stack en erreur ?
6. Quel est l'intérêt des outputs dans un template ?
7. Comment créer des dépendances entre ressources ?
8. Qu'apportent les templates imbriqués ?

---

##  Bilan du LAB 6

Dans ce LAB, vous avez maîtrisé :

- L'**installation et configuration** de Heat dans DevStack.
- La **création de templates HOT** pour différents scénarios (simple, complet, 3-tiers).
- Le **déploiement et la gestion** du cycle de vie des stacks (create, update, delete).
- L'utilisation des **paramètres** et **outputs** pour des templates réutilisables.
- Le **dépannage** et la résolution d'erreurs de déploiement Heat.
- Les **patterns avancés** (depends_on, conditions, templates imbriqués).

Heat vous permet d'industrialiser vos déploiements OpenStack et de gérer des infrastructures complexes comme du code (Infrastructure as Code). C'est un outil essentiel pour l'orchestration en production et la certification COA avancée.

---

**Auteur** :Dr.Ing.MAHER HENI LAB DevStack Multi-nœuds – Orchestration Heat  
**Version** : 1.0  
**Date** : Février 2026  
**Licence** : Usage pédagogique et formation
