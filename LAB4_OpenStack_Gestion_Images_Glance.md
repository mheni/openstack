# LAB 4 : Gestion des Images OpenStack (Glance) et Personnalisation de Base

---

## 🎯 Objectifs du LAB

À l’issue de ce LAB, vous serez capable de :

- Lister, examiner et gérer les **images** OpenStack (Glance).
- Importer de nouvelles images (Cirros, Ubuntu Cloud).
- Gérer la **visibilité** des images (publique, privée, projet spécifique).
- Utiliser les **propriétés** d’images (metadata) pour décrire leur usage.
- Lancer des instances basées sur ces images et vérifier leur comportement.

Ce LAB s’inscrit dans la continuité des LABs 1–3 (projets, réseau, flavors, instances). Il se concentre sur le rôle d’**administrateur d’images** dans OpenStack.

---

## 📚 Prérequis

Avant de commencer :

1. DevStack multi‑nœuds est déployé et fonctionnel.
2. Vous disposez au minimum d’une image `cirros-0.6.2-x86_64-disk` (déployée par DevStack) et idéalement d’une image Ubuntu Cloud (ex: `ubuntu-22.04-server-cloudimg-amd64.img`).
3. Vous pouvez exécuter des commandes en tant qu’`admin` sur le nœud **controller** :

```bash
source /opt/stack/devstack/openrc admin admin
openstack image list
```

Résultat attendu : au moins une image en statut `active`.

---

## PARTIE 1 : Découverte et inspection des images

### 1.1 Lister les images disponibles

Sur le **controller**, en tant qu’admin :

```bash
source /opt/stack/devstack/openrc admin admin

# Lister les images
openstack image list
```

Observez pour chaque image :

- `Name` (nom)
- `Status` (active, queued, etc.)
- `Visibility` (public, private, shared)
- `Disk Format` (qcow2, raw, vmdk…)

### 1.2 Inspecter les détails d’une image

Choisissez une image (par ex. `cirros-0.6.2-x86_64-disk`) :

```bash
openstack image show cirros-0.6.2-x86_64-disk
```

Points d’attention :

- `min_disk`, `min_ram` : prérequis minimum pour les flavors.
- `properties` : architecture, hypervisor type, etc.
- `visibility` : qui peut voir/utiliser l’image.

### 1.3 Exploration via Horizon

1. Connectez-vous à **Horizon** avec `admin / openstack`.
2. Allez dans **Admin → Compute → Images**.
3. Observez la liste des images, leurs formats, tailles, visibilité.
4. Comparez avec ce que vous voyez dans les **Projects → Compute → Images** (Vue projet Formation1, par exemple avec `user1`).

---

## PARTIE 2 : Import d’une nouvelle image (Ubuntu Cloud)

### 2.1 Préparation du fichier image

Placez sur le **controller** un fichier image Ubuntu Cloud, par exemple :

- `/tmp/ubuntu-22.04-server-cloudimg-amd64.img`

(Le téléchargement peut être fait en amont avec `wget` ou `curl` depuis le site officiel Ubuntu.)

Vérifiez la présence du fichier :

```bash
ls -lh /tmp/ubuntu-22.04-server-cloudimg-amd64.img
```

### 2.2 Import de l’image via CLI

En tant qu’`admin` :

```bash
source /opt/stack/devstack/openrc admin admin

openstack image create "ubuntu-22.04-cloud"   --file /tmp/ubuntu-22.04-server-cloudimg-amd64.img   --disk-format qcow2   --container-format bare   --public   --property os_distro=ubuntu   --property os_version=22.04   --property hw_qemu_guest_agent=yes

# Vérifier
openstack image list
openstack image show ubuntu-22.04-cloud
```

**Résultat attendu** : une image `ubuntu-22.04-cloud` en `active`, `visibility=public`, avec les propriétés définies.

### 2.3 Import via Horizon (optionnel)

1. Dans Horizon, allez dans **Admin → Compute → Images → Create Image**.
2. Donnez un nom (ex : `ubuntu-22.04-cloud-Horizon`).
3. Sélectionnez `Image File` et téléversez le fichier `.img`.
4. Choisissez `Format = QCOW2`, `Visibility = Public`.
5. Validez et attendez le statut `Active`.

---

## PARTIE 3 : Visibilité, partage et images par projet

### 3.1 Images publiques vs privées

En CLI, examinez la visibilité :

```bash
openstack image list --long
```

Identifiez :

- Les images `public` (visibles par tous les projets).
- Les images `private` (visibles seulement par leur propriétaire).

### 3.2 Création d’une image privée pour AdminLab

Objectif : réserver une image à un projet particulier (ex: `AdminLab`).

```bash
# Toujours en admin

openstack image create "adminlab-tools"   --file /tmp/ubuntu-22.04-server-cloudimg-amd64.img   --disk-format qcow2   --container-format bare   --private   --project AdminLab   --property purpose=admin-tools

openstack image show adminlab-tools
```

**Résultat attendu** : `visibility=private`, `owner=AdminLab`.

### 3.3 Vérification côté projets

- Connectez-vous dans Horizon avec `user1` (Formation1) :
  - Dans **Project → Compute → Images**, vous devez voir :
    - Les images `public`.
    - Pas l’image `adminlab-tools` (privée pour AdminLab).

- Connectez-vous avec un utilisateur du projet `AdminLab` (ex: `trainer`) :
  - Vous voyez `adminlab-tools` + les images publiques.

> Question : que se passe-t-il si vous changez la visibilité d’une image privée en `public` ?

---

## PARTIE 4 : Propriétés d’images et compatibilité flavors

### 4.1 Définir des propriétés hardware

Certaines propriétés peuvent influencer le scheduler ou la compatibilité avec certains hyperviseurs.

Exemple : marquer une image comme optimisée pour des flavor “large” :

```bash
source /opt/stack/devstack/openrc admin admin

openstack image set ubuntu-22.04-cloud   --property recommended_flavor=demo.medium   --property environment=training

openstack image show ubuntu-22.04-cloud
```

Observez la section `properties` et discutez de l’usage de ces métadonnées (documentation interne, sélection dynamique côté outils d’IA / Terraform / Ansible, etc.).

### 4.2 Tester avec un flavor inadapté (min_disk / min_ram)

Modifiez (à titre de test) les propriétés d’une image pour exiger plus de ressources :

```bash
openstack image set ubuntu-22.04-cloud --min-ram 4096
openstack image show ubuntu-22.04-cloud | egrep 'min_ram|min_disk'
```

Ensuite, depuis Formation1 (user1) :

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

# Tenter de booter avec un flavor trop petit (ex: demo.tiny 512 Mo)
openstack server create   --flavor demo.tiny   --image ubuntu-22.04-cloud   --network f1-net   --key-name mykey   img-test-small
```

**Résultat attendu** : échec ou erreur liée aux ressources insuffisantes (min_ram non respecté). Essayez ensuite avec un flavor plus grand (ex: `demo.medium`).

---

## PARTIE 5 : Lancement d’instances basées sur les nouvelles images

### 5.1 Instance Ubuntu cloud pour Formation1

Toujours côté `user1` :

```bash
source user1-openrc.sh

openstack server create   --flavor demo.medium   --image ubuntu-22.04-cloud   --network f1-net   --key-name mykey   f1-ubuntu-test

openstack server list
```

Depuis Horizon (Formation1) :

- Vérifiez que l’instance `f1-ubuntu-test` est `ACTIVE`.
- Récupérez son IP privée et, si besoin, associez-lui une **floating IP** pour vous y connecter en SSH.

### 5.2 Vérification dans la VM

Depuis l’hôte, connectez-vous en SSH sur `f1-ubuntu-test` :

```bash
ssh ubuntu@<FLOATING_IP>
```

Vérifiez :

```bash
uname -a
cat /etc/os-release
```

Vous confirmez ainsi que l’instance tourne bien sur l’image Ubuntu cloud importée.

---

## PARTIE 6 : Nettoyage et cycle de vie des images

### 6.1 Désactivation temporaire d’une image

Vous pouvez désactiver une image sans la supprimer (maintenance) :

```bash
source /opt/stack/devstack/openrc admin admin

openstack image set ubuntu-22.04-cloud --deactivated
openstack image show ubuntu-22.04-cloud | grep status
```

Effet : l’image n’est plus utilisable pour créer de nouvelles instances, mais reste disponible pour audit ou réactivation.

Réactivation :

```bash
openstack image set ubuntu-22.04-cloud --activated
```

### 6.2 Suppression d’images non utilisées

Identifiez les images obsolètes (non utilisées par des instances) :

```bash
openstack server list --all-projects --image ubuntu-22.04-cloud
```

Si aucune instance ne dépend d’une image de test, vous pouvez la supprimer :

```bash
openstack image delete ubuntu-22.04-cloud-Horizon
```

> Attention : ne supprimez pas une image encore utilisée par des VMs de stagiaires.

---

## 🔎 Questions de validation

1. Quelle est la différence entre une image **publique** et **privée** ?
2. Que représentent `min_disk` et `min_ram` sur une image ?
3. Comment réserver une image à un **projet spécifique** ?
4. Dans quels cas utiliseriez-vous des **propriétés** d’images (metadata) ?
5. Quel risque y a-t-il à supprimer une image encore utilisée par des instances ?

---

## ✅ Bilan du LAB 4

Dans ce LAB, vous avez appris à :

- Gérer le **catalogue d’images** de votre cloud OpenStack (Glance).
- Importer et configurer des images cloud (Cirros, Ubuntu) pour vos formations.
- Contrôler la **visibilité** et l’appartenance projet des images.
- Utiliser des **propriétés** pour documenter et contraindre l’usage des images.
- Lancer des instances Ubuntu cloud et vérifier leur origine et leurs paramètres.

Ces compétences sont essentielles pour maintenir un catalogue d’images propre, cohérent et adapté aux besoins de vos stagiaires et de vos environnements de test.

---

**Auteur** : LAB DevStack Multi-nœuds – Gestion des Images (Glance)  
**Version** : 1.0  
**Date** : Février 2026  
**Licence** : Usage pédagogique et formation
