# Labo de Supervision Réseau – Projet Supervision_CDEC

Simulation d'une infrastructure réseau d'entreprise (routeur + pare-feu + serveurs) avec **GNS3** et **VMware Workstation**, en préparation d'un déploiement de supervision réseau **Zabbix** pour la Caisse des Dépôts et Consignations (CDEC).

Ce document décrit l'intégralité de la mise en place du labo réseau, de l'installation des outils jusqu'à l'obtention d'un réseau fonctionnel avec accès Internet. Il est rédigé pour être **reproductible par un débutant complet** en GNS3/virtualisation réseau.

---

## Table des matières

1. [Contexte du projet](#1-contexte-du-projet)
2. [Architecture cible](#2-architecture-cible)
3. [Prérequis et outils](#3-prérequis-et-outils)
4. [Installation de la GNS3 VM dans VMware](#4-installation-de-la-gns3-vm-dans-vmware)
5. [Configuration réseau VMware (VMnet)](#5-configuration-réseau-vmware-vmnet)
6. [Connexion de GNS3 à la GNS3 VM](#6-connexion-de-gns3-à-la-gns3-vm)
7. [Résolution d'un problème critique : pilote Npcap](#7-résolution-dun-problème-critique--pilote-npcap)
8. [Création des templates VyOS et pfSense](#8-création-des-templates-vyos-et-pfsense)
9. [Construction de la topologie GNS3](#9-construction-de-la-topologie-gns3)
10. [Câblage vers les VM VMware (Cloud nodes)](#10-câblage-vers-les-vm-vmware-cloud-nodes)
11. [Configuration de VyOS](#11-configuration-de-vyos)
12. [Configuration de pfSense](#12-configuration-de-pfsense)
13. [Configuration IP des VM supervisées](#13-configuration-ip-des-vm-supervisées)
14. [Accès Internet complet (NAT + Cloud1)](#14-accès-internet-complet-nat--cloud1)
15. [Installation permanente de VyOS sur disque](#15-installation-permanente-de-vyos-sur-disque)
16. [Récapitulatif du plan d'adressage IP](#16-récapitulatif-du-plan-dadressage-ip)
17. [État actuel du labo](#17-état-actuel-du-labo)
18. [Problèmes rencontrés et solutions (résumé)](#18-problèmes-rencontrés-et-solutions-résumé)
19. [Prochaines étapes](#19-prochaines-étapes)

---

## 1. Contexte du projet

Ce labo s'inscrit dans le cadre d'un **mémoire d'ingénieur en Management des Systèmes d'Information**, dont le sujet est :

> **Mise en place d'une solution de supervision réseau Zabbix à la Caisse des Dépôts et Consignations (CDEC)**

L'objectif pédagogique et méthodologique est de dérouler le projet en deux temps :

1. **Simulation** : reconstituer une topologie réseau représentative (routeur, pare-feu, serveurs) dans un environnement virtualisé maîtrisé, avant tout déploiement réel.
2. **Implémentation réelle** : une fois la configuration validée en simulation, reproduire les mêmes réglages (templates Zabbix, seuils, règles de pare-feu) sur l'infrastructure réelle de la CDEC.

Ce document couvre uniquement la **phase de simulation**.

### Pourquoi cette approche par simulation ?

- Valider l'architecture réseau et les règles de pare-feu sans risque sur un environnement de production bancaire
- Permettre de documenter précisément chaque étape pour la partie "mise en œuvre" du mémoire
- Tester des scénarios de panne/charge (via des outils comme `iperf3`, `hping3`, `stress-ng`) sans impact réel

---

## 2. Architecture cible

### Schéma logique

```
                    ┌─────────────┐
                    │   Cloud1    │  (accès Internet réel,
                    │  (VMnet8)   │   via NAT VMware)
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   Switch1   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ VyOS-Router │  Routeur / simule le WAN
                    │             │  eth0 : DHCP (vers Internet)
                    │             │  eth1 : 10.0.0.1/30 (vers pfSense)
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   pfSense   │  Pare-feu principal
                    │             │  WAN (em0) : 10.0.0.2/30
                    │             │  LAN (em1) : 192.168.47.254/24
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   Switch2   │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   Cloud2    │  Pont vers le réseau VMware
                    │  (VMnet1)   │  (Host-only, sans DHCP)
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
┌───────▼──────┐   ┌───────▼──────┐   ┌───────▼──────┐
│ ZABBIX_SERVER │   │   Windows    │   │ Ubuntu Server │
│ 192.168.47.10 │   │192.168.47.20 │   │192.168.47.30  │
│  (Debian 12)  │   │              │   │               │
└───────────────┘   └──────────────┘   └───────────────┘
```



<img width="1366" height="768" alt="Capture d’écran (2811)" src="https://github.com/user-attachments/assets/57a65946-5346-4c11-aa0d-24039168e309" />


### Répartition des rôles

| Élément | Rôle | Logiciel / Type |
|---|---|---|
| Cloud1 | Pont vers Internet réel | Cloud node GNS3 → VMnet8 (NAT) |
| Switch1 | Switch WAN simulé | Ethernet Switch natif GNS3 |
| VyOS-Router | Routeur d'interconnexion + NAT | QEMU (image VyOS rolling) |
| pfSense | Pare-feu principal (protection du LAN) | QEMU (image pfSense CE 2.7.2) |
| Switch2 | Switch LAN interne | Ethernet Switch natif GNS3 |
| Cloud2 | Pont vers les VM VMware | Cloud node GNS3 → VMnet1 (Host-only) |
| ZABBIX_SERVER | Serveur de supervision | VM VMware (Debian 12) |
| Windows | Hôte supervisé | VM VMware |
| Ubuntu Server | Hôte supervisé | VM VMware |

### Principe clé de l'architecture

**GNS3 et VMware Workstation sont deux mondes séparés**, même s'ils tournent sur la même machine physique :

- La topologie réseau (VyOS, pfSense, switches) est **simulée dans la GNS3 VM**
- Les serveurs supervisés (Zabbix, Windows, Ubuntu Server) sont des **VM VMware classiques**, indépendantes de GNS3

Le pont entre les deux mondes se fait via des **Cloud nodes** GNS3, associés à des réseaux **VMnet** de VMware :
- **Cloud1 → VMnet8** (réseau NAT, avec accès Internet réel via VMware)
- **Cloud2 → VMnet1** (réseau Host-only, sans DHCP, dédié au LAN simulé)

Toute VM VMware branchée sur **VMnet1** apparaît automatiquement comme faisant partie du même réseau que la topologie GNS3, sans avoir besoin d'être un nœud GNS3 elle-même.

---

## 3. Prérequis et outils

### Logiciels à installer sur la machine hôte (Windows)

| Outil | Usage | Lien |
|---|---|---|
| **VMware Workstation** | Hyperviseur hébergeant la GNS3 VM et les VM supervisées | déjà installé dans ce projet |
| **GNS3** (logiciel) | Interface de conception de topologies réseau | https://www.gns3.com/software/download |
| **GNS3 VM** | Moteur d'émulation (tourne dans VMware) | https://www.gns3.com/software/download-vm |
| **Npcap** | Pilote de capture/injection réseau nécessaire à GNS3 sous Windows | https://npcap.com/#download |

### Images système à télécharger

| Image | Usage | Source |
|---|---|---|
| **VyOS rolling release (.iso)** | Routeur virtuel | https://downloads.vyos.io/ |
| **pfSense CE (.iso)** | Pare-feu virtuel | https://www.pfsense.org/download/ |

> ⚠️ **Piège fréquent** : le site `vyos.net/get/` redirige vers un système d'abonnement payant (LTS). Le lien `downloads.vyos.io` donne accès à la version **rolling gratuite**, sans compte requis.

### VM VMware à préparer

- 1 VM Debian 12 (ou équivalent) pour Zabbix Server
- 1 VM Windows (10 ou Server) comme hôte à superviser
- 1 VM Ubuntu Server comme hôte à superviser

---

## 4. Installation de la GNS3 VM dans VMware

1. Télécharger la GNS3 VM (fichier `.ova`) depuis le site officiel GNS3
2. Dans VMware Workstation : **Fichier → Ouvrir**, sélectionner le fichier `.ova`
3. Donner un nom (ex: `GNS3 VM`) et un emplacement de stockage
4. Cliquer sur **Importer**
5. **Avant de démarrer**, ajuster les ressources :
   - Clic droit sur la VM → **Modifier les paramètres de la VM**
   - **Mémoire** : au moins 4096 Mo (4 Go), idéalement 6-8 Go
   - **Processeurs** : au moins 2 vCPU

---

## 5. Configuration réseau VMware (VMnet)

### Principe

La GNS3 VM a besoin de **deux cartes réseau** avec des rôles différents :

| Carte | Rôle | Réseau utilisé |
|---|---|---|
| **Network Adapter 1** | Pilotage GNS3 (logiciel) ↔ GNS3 VM | NAT (ou VMnet8) |
| **Network Adapter 2** | Réseau du labo (topologie ↔ VM VMware) | Custom : VMnet1 (Host-only) |

### 5.1 – Configurer VMnet1 (réseau du labo)

1. VMware Workstation → **Édition → Éditeur de réseaux virtuels**
2. Cliquer sur **Change Settings** (droits administrateur requis)
3. Sélectionner (ou créer) **VMnet1**
4. Type : **Host-only**
5. **Décocher** "Use local DHCP service to distribute IP address to VMs" (adressage géré manuellement)
6. Cocher "Connect a host virtual adapter to this network"
7. Subnet IP : `192.168.47.0`, masque `255.255.255.0`
8. **Apply**

> ℹ️ Le PC hôte reçoit automatiquement l'adresse `192.168.47.1` sur ce réseau — à éviter lors de l'attribution d'IP aux autres équipements.

### 5.2 – VMnet8 (réseau NAT, déjà présent par défaut)

VMnet8 est le réseau NAT standard de VMware, avec DHCP activé. Il sert :
- Au pilotage GNS3 ↔ GNS3 VM (carte réseau 1)
- Plus tard, de passerelle Internet réelle pour Cloud1 (voir section 14)

### 5.3 – Configurer les cartes réseau de la GNS3 VM

1. GNS3 VM éteinte → clic droit → **Modifier les paramètres de la VM**
2. **Network Adapter** (carte 1) → **NAT** (ou Custom : VMnet8)
3. **Network Adapter 2** (carte 2, à ajouter si absente via **Add → Network Adapter**) → **Custom : VMnet1**

### 5.4 – Brancher les VM supervisées (Zabbix, Windows, Ubuntu Server) sur VMnet1

Pour chaque VM :
1. VM éteinte → clic droit → **Modifier les paramètres de la VM**
2. Carte réseau → **Custom : VMnet1**

> ⚠️ Conséquence attendue : ces VM **perdent temporairement Internet** après ce changement (VMnet1 est isolé). L'accès Internet sera restauré une fois pfSense et VyOS configurés (section 14), en passant par le pare-feu — exactement comme dans une vraie architecture d'entreprise où les serveurs internes ne sont jamais connectés directement à Internet.

---

## 6. Connexion de GNS3 à la GNS3 VM

1. Démarrer la **GNS3 VM** dans VMware
2. Attendre l'écran de démarrage affichant une IP du type :
   ```
   IP: 192.168.208.129 PORT: 80
   ```
3. Ouvrir **GNS3** (logiciel) → **Edit → Preferences → GNS3 VM**
4. Cocher **Enable the GNS3 VM**
5. **Virtualization engine** : VMware
6. **VM** : sélectionner la GNS3 VM détectée automatiquement
7. **Apply → OK**

Vérifier dans le panneau **Servers Summary** (bas droite de GNS3) que les deux serveurs (local et GNS3 VM) passent au **vert**.

---

## 7. Résolution d'un problème critique : pilote Npcap

### Symptôme rencontré

Lors de la tentative de connexion d'un Cloud node à une interface VMnet, l'erreur suivante apparaît :

```
nio_ethernet_open: unable to open device... Le périphérique spécifié est introuvable
```

Ou, lors du démarrage d'un équipement local (switch/cloud) :

```
dynamips.exe - Erreur système
Impossible d'exécuter le code, car wpcap.dll est introuvable.
```

### Cause

GNS3 nécessite le pilote **Npcap** pour accéder aux interfaces réseau bas niveau sous Windows. L'ancien pilote **WinPcap** (obsolète) est incompatible avec les interfaces réseau virtuelles modernes.

### Solution

1. **Désinstaller WinPcap** si présent (Panneau de configuration → Programmes et fonctionnalités)
2. Télécharger **Npcap** depuis https://npcap.com/#download
3. Lancer l'installateur **en tant qu'administrateur** (clic droit → Exécuter en tant qu'administrateur)
4. **Cocher impérativement** : ☑ *Install Npcap in WinPcap API-compatible Mode*
5. **Redémarrer complètement Windows** (pas juste fermer les applications)

### Vérification

```cmd
dir C:\Windows\System32\wpcap.dll
dir C:\Windows\SysWOW64\wpcap.dll
```

Les deux fichiers doivent exister (System32 pour les composants 64 bits comme GNS3 lui-même, SysWOW64 pour les composants 32 bits comme `dynamips.exe`).

> 💡 Si le fichier manque uniquement dans un des deux dossiers après une première installation, désinstaller Npcap, nettoyer les DLL restantes manuellement, puis réinstaller en s'assurant que l'option de compatibilité WinPcap est bien cochée et que l'installation se termine intégralement avant de fermer la fenêtre.

---

## 8. Création des templates VyOS et pfSense

### 8.1 – Template VyOS

1. **Edit → Preferences → QEMU VMs → New**
2. **Run this QEMU VM on the GNS3 VM**
3. Nom : `VyOS-Router`
4. RAM : 512 Mo
5. **Adapters** (nombre d'interfaces réseau) : mettre **3** (pas 1 par défaut — nécessaire pour avoir plusieurs interfaces disponibles : une vers le WAN simulé, une vers pfSense, une de réserve)
6. Disk image (CD/DVD) : sélectionner le fichier `.iso` VyOS
7. Catégorie : **Routers**
8. Console type : **Telnet** (par défaut, adapté à VyOS qui fonctionne en CLI série)

### 8.2 – Template pfSense

1. **Edit → Preferences → QEMU VMs → New**
2. **Run this QEMU VM on the GNS3 VM**
3. Nom : `pfSense 2.7.2`
4. RAM : 1024 Mo minimum
5. Disk image (CD/DVD) : sélectionner le fichier `.iso` pfSense
6. Catégorie : **Firewalls**
7. **Console type : changer en VNC** (pfSense démarre sur une console graphique VGA, pas série — Telnet afficherait un écran vide/illisible)

### 8.3 – Point d'attention : disque dur virtuel

Par défaut, un template QEMU peut n'avoir que l'ISO monté en disque primaire (HDA), sans véritable disque dur vierge. Cela permet de démarrer en mode "live" mais **empêche toute installation permanente**.

Pour permettre une installation persistante (recommandé pour VyOS et pfSense) :
1. Dans le template (Edit → Preferences → QEMU VMs → *nom du template* → Edit)
2. Onglet **CD/DVD** : y placer le fichier `.iso`
3. Onglet **HDD**, champ **HDA (Primary Master)** : vider le champ ISO s'il y est, puis cliquer **Create...** pour générer un disque virtuel vierge (ex: 4096 Mo, format qcow2)

> ⚠️ Cette configuration doit être faite **avant** de glisser le nœud sur le canevas. Modifier le template après coup ne met pas à jour les nœuds déjà créés — il faut alors éditer le nœud individuellement (clic droit sur le nœud → Configure) ou le supprimer et le recréer.

---

## 9. Construction de la topologie GNS3

### 9.1 – Créer le projet

**File → New blank project** → nommer `Supervision_CDEC`

### 9.2 – Glisser les équipements sur le canevas

Depuis la palette d'équipements (onglet "All devices" à gauche), glisser-déposer dans cet ordre, de gauche à droite :

1. **Cloud** (nommé automatiquement `Cloud1`)
2. **Ethernet switch** (`Switch1`)
3. **VyOS-Router** (`VyOS-Router-1`)
4. **pfSense 2.7.2** (`pfSense2.7.2-2`)
5. **Ethernet switch** (`Switch2`)

### 9.3 – Câbler les équipements

Utiliser l'outil câble (icône lien dans la barre d'outils verticale à gauche) : cliquer sur le premier équipement, choisir l'interface, cliquer sur le second équipement, choisir l'interface.

Connexions à réaliser :

| Depuis | Interface | Vers | Interface |
|---|---|---|---|
| Cloud1 | – | Switch1 | – |
| Switch1 | – | VyOS-Router-1 | `eth0` |
| VyOS-Router-1 | `eth1` | pfSense2.7.2-2 | `em0` (**WAN**) |
| pfSense2.7.2-2 | `em1` (**LAN**) | Switch2 | – |

> ⚠️ **Erreur fréquente à éviter** : pfSense a deux interfaces réseau bien distinctes : `em0` = WAN, `em1` = LAN. Un câblage inversé (ex: brancher VyOS sur `em1` au lieu de `em0`) provoque des échecs de ping de type "Destination Host Unreachable", car les deux extrémités du lien se retrouvent sur des sous-réseaux logiquement incompatibles. **Toujours vérifier l'infobulle du câble** (survol à la souris) pour confirmer les interfaces réellement connectées.

---

## 10. Câblage vers les VM VMware (Cloud nodes)

### 10.1 – Cloud2 (vers le LAN / VMnet1)

1. Glisser un second **Cloud** (`Cloud2`) sur le canevas, à droite de Switch2
2. Double-clic sur Cloud2 → onglet **Ethernet interfaces**
3. Dans le menu déroulant, sélectionner **VMware Network Adapter VMnet1**
4. **Add** → **Apply → OK**
5. Câbler **Switch2 ↔ Cloud2**

### 10.2 – Cloud1 (vers Internet réel / VMnet8)

Même procédure pour Cloud1 :
1. Double-clic sur Cloud1 → onglet **Ethernet interfaces**
2. Sélectionner **VMware Network Adapter VMnet8**
3. **Add → Apply → OK**
4. Vérifier/refaire le câblage **Cloud1 ↔ Switch1**

> ⚠️ Ne jamais sélectionner une interface **Wi-Fi** dans cette liste — GNS3 avertit explicitement que les interfaces Wi-Fi ne fonctionnent pas correctement pour ce type de pont.

---

## 11. Configuration de VyOS

### 11.1 – Démarrage et connexion

1. Clic droit sur **VyOS-Router-1** → **Start**
2. Double-clic pour ouvrir la console Telnet
3. Connexion : login `vyos`, mot de passe `vyos`

### 11.2 – Configuration des interfaces

```
configure
set interfaces ethernet eth0 address 192.168.47.20/24
set interfaces ethernet eth1 address 10.0.0.1/30
commit
save
```

> Ces adresses seront ensuite ajustées (voir section 14) pour permettre l'accès Internet réel.

### 11.3 – Vérification de l'état des interfaces

En mode opérationnel (après `exit` pour sortir du mode `[edit]`) :
```
show interfaces
```
Vérifier que les interfaces utilisées affichent l'état `u/u` (up/up) dans la colonne S/L.

---

## 12. Configuration de pfSense

### 12.1 – Installation initiale (première utilisation)

1. Démarrer **pfSense2.7.2-2**, double-clic pour ouvrir la console **VNC**
2. Suivre l'installeur : accepter la licence, choisir **Install pfSense**
3. Configuration ZFS : sélectionner **Stripe – No Redundancy** (adapté à un disque unique)
4. Sélectionner le disque virtuel proposé (ex: `vtbd0`) en cochant la case avec la **barre Espace**
5. Confirmer l'effacement du disque (**Yes**)
6. Laisser l'installation se dérouler, puis redémarrer

### 12.2 – Configuration IP via le menu console

Au menu principal pfSense, choisir l'option **2) Set interface(s) IP address**.

**Interface WAN (em0)** :
- DHCP ? → `n`
- Adresse IPv4 : `10.0.0.2`
- Masque (bits) : `30`
- Passerelle : `10.0.0.1`
- IPv6 via DHCP6 ? → `n`
- DHCP server sur WAN ? → `n`
- Revert to HTTP ? → `n` (garder HTTPS)

**Interface LAN (em1)** :
- DHCP ? → `n`
- Adresse IPv4 : `192.168.47.254`
- Masque (bits) : `24`
- Passerelle : laisser vide (Entrée)
- IPv6 via DHCP6 ? → `n`
- DHCP server sur LAN ? → `n` (IP fixes gérées manuellement sur chaque VM, pour un mémoire plus prévisible et documentable)
- Revert to HTTP ? → `n`

> ⚠️ Ne pas utiliser `192.168.47.1` pour le LAN pfSense : cette adresse est déjà occupée par le PC hôte sur VMnet1.

### 12.3 – Vérification

Menu pfSense → option **7) Ping host** → tester `10.0.0.1` (VyOS). Un ping réussi confirme la connectivité WAN.

> ℹ️ Un ping échouant dans le sens VyOS → pfSense (avec un simple timeout, pas une erreur "Host Unreachable") est normal : pfSense bloque par défaut tout trafic entrant non sollicité sur son interface WAN, y compris ICMP.

---

## 13. Configuration IP des VM supervisées

### 13.1 – Debian 12 (ZABBIX_SERVER)

Sur Debian, la configuration réseau se fait via `/etc/network/interfaces` (pas Netplan, propre à Ubuntu).

1. Identifier le nom de l'interface :
   ```
   ip a
   ```
   (typiquement `ens33`)

2. Se connecter en **root** si l'utilisateur courant n'a pas les droits `sudo` :
   ```
   su -
   ```

3. Éditer la configuration :
   ```
   nano /etc/network/interfaces
   ```
   Ajouter à la fin du fichier :
   ```
   auto ens33
   iface ens33 inet static
       address 192.168.47.10
       netmask 255.255.255.0
       gateway 192.168.47.254
       dns-nameservers 8.8.8.8 1.1.1.1
   ```

4. Redémarrer le réseau :
   ```
   systemctl restart networking
   ```

5. Vérifier :
   ```
   ip a
   ping -c 4 192.168.47.254
   ```

### 13.2 – Windows et Ubuntu Server

Même principe (adresse IP fixe, masque `255.255.255.0`, passerelle `192.168.47.254`, DNS `8.8.8.8` / `1.1.1.1`), configuré respectivement via les paramètres réseau Windows (IPv4 statique) et via `netplan` (Ubuntu Desktop/Server récent) ou `/etc/network/interfaces` (Ubuntu plus ancien).

---

## 14. Accès Internet complet (NAT + Cloud1)

Une fois le LAN interne fonctionnel (Zabbix ↔ pfSense ↔ VyOS), il reste à donner un accès Internet réel en connectant Cloud1 à une interface réseau ayant elle-même accès à Internet (VMnet8/NAT), puis en activant le NAT sortant sur VyOS.

### 14.1 – Reconfigurer eth0 de VyOS en DHCP

```
configure
delete interfaces ethernet eth0 address 192.168.47.20/24
set interfaces ethernet eth0 address dhcp
commit
save
```

### 14.2 – Activer le NAT source (masquerade) sur VyOS

```
set nat source rule 100 outbound-interface name eth0
set nat source rule 100 source address 192.168.47.0/24
set nat source rule 100 translation address masquerade
set nat source rule 200 outbound-interface name eth0
set nat source rule 200 source address 10.0.0.0/30
set nat source rule 200 translation address masquerade
commit
save
```

> ⚠️ Sur les versions récentes de VyOS (1.4+), la syntaxe correcte est `outbound-interface name eth0` (et non `outbound-interface eth0`).

### 14.3 – Vérification en cascade

```
# Sur VyOS (mode opérationnel)
show interfaces ethernet eth0    # doit afficher une IP de type 192.168.208.x
ping -c 4 8.8.8.8                # doit réussir

# Sur ZABBIX_SERVER
ping -c 4 8.8.8.8                # doit réussir également
```

Chemin complet validé :
```
Zabbix (192.168.47.10) → pfSense (LAN→WAN, NAT) → VyOS (NAT) → Cloud1 → VMnet8 → Internet
```

---

## 15. Installation permanente de VyOS sur disque

Par défaut, VyOS démarre en mode **live-ISO** : toute configuration est perdue au redémarrage tant que le système n'est pas installé sur un disque virtuel.

### 15.1 – Sauvegarder la configuration actuelle (précaution)

```
show configuration commands
```
Copier le résultat avant toute manipulation.

### 15.2 – Pré-requis : un disque virtuel doit être attaché au nœud

Voir section 8.3 pour la procédure. **Si le nœud a été créé avant la correction du template**, il faut éditer le nœud individuellement (clic droit sur le nœud dans le canevas → **Configure** → onglet **HDD**) et non uniquement le template, sous peine que `install image` échoue avec :
```
No suitable disk was found. There must be at least one disk of 2GB or greater size.
```

### 15.3 – Lancer l'installation

```
install image
```

Répondre aux invites :
- Continuer ? → `y` (ou `Yes`)
- Nom de l'image → valeur par défaut (Entrée)
- Mot de passe utilisateur `vyos` → définir un mot de passe
- Console par défaut (K: KVM, S: Serial) → `S`
- Disque cible → accepter le disque détecté (ex: `sda`)
- Taille de la partition racine → valeur par défaut (Entrée)
- Copie de configuration → valeur par défaut (Entrée)
- Disque pour GRUB → valeur par défaut (Entrée)

### 15.4 – Redémarrer et revalider

```
reboot
```

Après redémarrage et reconnexion, vérifier que la configuration réseau est **automatiquement présente sans rien retaper** :
```
show interfaces
```

Si les IP configurées précédemment (`eth0`, `eth1`) apparaissent directement, l'installation permanente est confirmée.

---

## 16. Récapitulatif du plan d'adressage IP

| Équipement | Interface | Adresse IP | Rôle |
|---|---|---|---|
| VyOS-Router | eth0 | DHCP (ex: 192.168.208.x) | Vers Internet (VMnet8) |
| VyOS-Router | eth1 | 10.0.0.1/30 | Vers pfSense (WAN) |
| pfSense | em0 (WAN) | 10.0.0.2/30 | Vers VyOS |
| pfSense | em1 (LAN) | 192.168.47.254/24 | Passerelle du LAN interne |
| ZABBIX_SERVER | ens33 | 192.168.47.10/24 | Serveur de supervision |
| Windows | – | 192.168.47.20/24 | Hôte supervisé |
| Ubuntu Server | – | 192.168.47.30/24 | Hôte supervisé |
| PC hôte (Windows) | VMnet1 | 192.168.47.1 | Interface hôte (réservée) |

Réseaux utilisés :
- **192.168.47.0/24** : LAN interne (derrière pfSense)
- **10.0.0.0/30** : liaison point-à-point VyOS ↔ pfSense
- **192.168.208.0/24** (VMnet8) : réseau NAT, management GNS3 VM + sortie Internet

---

## 17. État actuel du labo

À ce stade, le labo est **entièrement fonctionnel** :

- ✅ GNS3 + GNS3 VM + VMware correctement configurés et connectés
- ✅ Topologie complète câblée : Cloud1 → Switch1 → VyOS → pfSense → Switch2 → Cloud2
- ✅ VyOS configuré et **installé de façon permanente** (interfaces + NAT)
- ✅ pfSense configuré (WAN, LAN)
- ✅ ZABBIX_SERVER, Windows et Ubuntu Server connectés avec IP fixes et accès Internet complet validé

---

<img width="1366" height="768" alt="Capture d’écran (2817)" src="https://github.com/user-attachments/assets/91e38018-7ed4-410c-8b70-7ea3bf0bffd2" />

<img width="1366" height="768" alt="Capture d’écran (2818)" src="https://github.com/user-attachments/assets/17def3d5-9878-4987-8be1-99d2221f3767" />

<img width="1366" height="768" alt="Capture d’écran (2819)" src="https://github.com/user-attachments/assets/28bbe891-792d-4e14-b3ce-429e62254e65" />

<img width="1366" height="768" alt="Capture d’écran (2820)" src="https://github.com/user-attachments/assets/7cfddd30-c524-4666-a0a9-f5a75be95b2f" />


## 18. Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|---|---|---|
| `Timeout... GNS3 VM not connecting` | Pare-feu Windows / antivirus bloquant le port 3080 | Autoriser GNS3 dans le pare-feu Windows, relancer en administrateur |
| VMnet2 disparu après redémarrage | Désynchronisation des services réseau VMware | Recréer le VMnet (ou en utiliser un autre, ex: VMnet1), redémarrer Windows |
| `unable to open device` sur le Cloud node | Pilote Npcap manquant/mal installé | Désinstaller WinPcap, installer Npcap avec le mode compatible WinPcap coché, redémarrer (voir section 7) |
| `wpcap.dll est introuvable` (dynamips.exe) | DLL 32 bits manquante dans `SysWOW64` | Réinstaller Npcap en admin avec l'option de compatibilité, vérifier les deux emplacements DLL |
| Erreur `pfSense-CE-2.7.0-RELEASE-amd64.iso is missing` | Le template pointait vers un nom de fichier différent de l'ISO réellement téléchargée (version 2.7.2 au lieu de 2.7.0) | Éditer le template, pointer manuellement vers le bon fichier ISO local |
| `Can't create the link the destination port is not free` | VyOS n'avait qu'une seule interface réseau (adapter count = 1) | Augmenter le nombre d'adaptateurs réseau du template VyOS à 3 |
| Ping VyOS → pfSense en "Destination Host Unreachable" | Câble tracé sur la mauvaise interface pfSense (`em1`/LAN au lieu de `em0`/WAN) | Supprimer et retracer le câble sur la bonne interface, vérifier via l'infobulle au survol |
| `install image` → "No suitable disk was found" | Le nœud VyOS n'avait qu'un ISO monté en HDA, aucun disque virtuel réel | Ajouter un disque virtuel vierge (HDA) et déplacer l'ISO vers le slot CD/DVD, au niveau du **nœud** (pas seulement du template) |
| `outbound-interface eth0` rejeté par VyOS | Changement de syntaxe entre versions VyOS | Utiliser `outbound-interface name eth0` sur les versions récentes |
| `daniel n'est pas dans le fichier sudoers` | Utilisateur sans droits sudo sur la VM Debian | Se connecter directement avec `su -` (compte root) |

---

# 19. Installation de Zabbix 7.0 LTS — VM ZABBIX_SERVER (Ubuntu Server 24.04 LTS)

 Cette section documente l'installation réellement effectuée, du système d'exploitation jusqu'à l'accès à l'interface web.

**Contexte VM** : Ubuntu Server 24.04.4 LTS, IP `192.168.47.50/24`, utilisateur `mikande`.

---

## 1. Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
```

## 2. Ajout du dépôt officiel Zabbix

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo apt update
```

> Point d'attention : le chemin officiel du dépôt ne contient pas de segment `/release/` (`zabbix/7.0/ubuntu/...`), contrairement à ce que certains tutoriels indiquent.

## 3. Installation des paquets Zabbix

```bash
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent -y
```

## 4. Installation et sécurisation de MySQL

```bash
sudo apt install mysql-server -y
sudo mysql_secure_installation
```

## 5. Création de la base de données et de l'utilisateur

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER zabbix@localhost IDENTIFIED BY 'Riche237@';
GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;
FLUSH PRIVILEGES;
SET GLOBAL log_bin_trust_function_creators = 1;
QUIT;
```

> Identifiants base de données : utilisateur `zabbix`, mot de passe `Riche237@` (la politique de mot de passe MySQL a rejeté un premier essai trop simple).

## 6. Import du schéma initial

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

Import confirmé : **204 tables** créées dans la base `zabbix`.

```bash
sudo mysql -u root -p -e "SET GLOBAL log_bin_trust_function_creators = 0;"
```

## 7. Configuration du serveur Zabbix

```bash
sudo nano /etc/zabbix/zabbix_server.conf
```

```
DBPassword=Riche237@
```

## 8. Configuration PHP (fuseau horaire)

Vérification préalable du module PHP actif dans Apache :

```bash
apache2ctl -M | grep php
```

Résultat : `php_module (shared)` → c'est le bloc `<IfModule mod_php.c>` qui s'applique (pas `mod_php7.c` ni PHP-FPM).

```bash
sudo nano /etc/zabbix/apache.conf
```

Ajout dans le bloc `<IfModule mod_php.c>` existant :

```apache
php_value date.timezone Africa/Douala
```

## 9. Démarrage des services

```bash
sudo systemctl restart apache2
sudo systemctl restart zabbix-server zabbix-agent
sudo systemctl enable zabbix-server zabbix-agent apache2
sudo systemctl status zabbix-server
```

## 10. Assistant de configuration web

Accès : `http://192.168.47.50/zabbix`

- Tous les prérequis PHP validés (PHP 8.3.6, timezone Africa/Douala détectée automatiquement)
- Connexion base de données : MySQL / localhost / `zabbix` / `zabbix` / `Riche237@`
- Nom du serveur Zabbix : `CDEC-Zabbix-Server`

Installation terminée avec succès.

## 11. Première connexion

```
URL     : http://192.168.47.50/zabbix
Login   : Admin
Password: zabbix (à changer immédiatement après connexion)
```

---

<img width="1366" height="768" alt="Capture d’écran (2879)" src="https://github.com/user-attachments/assets/8769d0da-851e-45b8-84eb-f732ba063bfa" />
<img width="1366" height="768" alt="Capture d’écran (2878)" src="https://github.com/user-attachments/assets/dade7df3-c038-41d6-99cd-46aadad2c2fa" />
<img width="1366" height="768" alt="Capture d’écran (2882)" src="https://github.com/user-attachments/assets/db4d862c-259a-4a6a-9966-e648aca96bb9" />


## Statut

✅ Serveur Zabbix 7.0 LTS opérationnel sur `192.168.47.50`, interface web accessible, base de données fonctionnelle.

**Prochaine étape** : installation des agents Zabbix sur les hôtes supervisés (Windows et Ubuntu Server) et rattachement des templates de supervision.


#20. Supervision des hôtes et équipements réseau — Agents Zabbix & SNMP

 Documente le déploiement des agents sur les postes/serveurs et l'activation SNMP sur les équipements réseau.

**Périmètre supervisé** : Ubuntu Desktop, Windows Server, routeur VyOS, pare-feu pfSense. Le switch GNS3 (`Ethernet switch` simple, sans OS ni pile IP) est hors périmètre technique de supervision — sa disponibilité est vérifiée indirectement via la connectivité de bout en bout des hôtes qu'il interconnecte.

---

## 1. Agent Zabbix — Ubuntu Desktop

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo apt update
sudo apt install zabbix-agent2 -y
```

```bash
sudo nano /etc/zabbix/zabbix_agent2.conf
```

```
Server=192.168.47.50
ServerActive=192.168.47.50
Hostname=Ubuntu-Desktop-CDEC
```

```bash
sudo systemctl restart zabbix-agent2
sudo systemctl enable zabbix-agent2
sudo ufw allow from 192.168.47.50 to any port 10050
```

## 2. Agent Zabbix — Windows Server

1. Télécharger le MSI : https://www.zabbix.com/download_agents (Zabbix 7.0, agent2, Windows)
2. Installer avec :
   - Host name : `WinServer-CDEC`
   - Zabbix server IP/DNS : `192.168.47.50`
   - Server or Proxy for active checks : `192.168.47.50` (⚠️ pas `127.0.0.1`, valeur par défaut à corriger)
   - Enable PSK : décoché
   - Add agent location to the PATH : coché
3. Vérifier le service :

```powershell
Get-Service Zabbix*
```

4. Ouvrir le port firewall entrant :

```powershell
New-NetFirewallRule -DisplayName "Zabbix Agent" -Direction Inbound -LocalPort 10050 -Protocol TCP -Action Allow
```

## 3. Déclaration des hôtes agent dans Zabbix

**Data collection → Hosts → Create host**, pour chaque machine :

| Champ | Ubuntu Desktop | Windows Server |
|---|---|---|
| Nom de l'hôte | Ubuntu-Desktop-CDEC | WinServer-CDEC |
| Groupe d'hôtes | CDEC - Postes | CDEC - Serveurs |
| Interface | Agent, IP de la VM, port 10050 | Agent, IP de la VM, port 10050 |
| Modèle | Linux by Zabbix agent | Windows by Zabbix agent |

⚠️ N'associer **qu'un seul** template par hôte (actif *ou* passif, pas les deux) — sinon conflit sur la clé `agent.hostname`.

---

## 4. SNMP — Routeur VyOS

```
configure
set service snmp community public authorization ro
set service snmp listen-address 10.0.0.1 port 161
set service snmp listen-address 127.0.0.1 port 161
commit
save
exit
```

Vérification :

```
show snmp community public
show configuration commands | match snmp
```

## 5. SNMP — Pare-feu pfSense

Interface web `https://<IP_LAN_pfSense>` → **Services → SNMP** :
- Enable : coché
- Polling Port : `161`
- Community String : `public`
- Save

Vérifier qu'aucune règle **Firewall → Rules → LAN** ne bloque le port 161 en entrée (la règle par défaut "Default allow LAN to any" suffit en labo).

## 6. Déclaration des hôtes SNMP dans Zabbix

**Data collection → Hosts → Create host**, pour chaque équipement :

| Champ | VyOS | pfSense |
|---|---|---|
| Nom de l'hôte | VyOS-Router-CDEC | pfSense-Firewall-CDEC |
| Groupe d'hôtes | CDEC - Réseau | CDEC - Réseau |
| Interface | SNMP, IP `10.0.0.1`, port 161 | SNMP, IP LAN pfSense, port 161 |
| Communauté SNMP | `public` (valeur réelle, pas la macro `{$SNMP_COMMUNITY}`) | `public` |

Aucun template SNMP générique officiel n'étant disponible par défaut dans Zabbix 7.0, deux items SNMP sont créés manuellement sur chaque hôte :

| Nom | Clé | OID | Type | Unités |
|---|---|---|---|---|
| System uptime | snmp.uptime | `.1.3.6.1.2.1.1.3.0` | Numérique (non signé) | `uptime` |
| System description | snmp.sysdescr | `.1.3.6.1.2.1.1.1.0` | Texte | — |

Toujours valider avec le bouton **Test → Obtenir la valeur** avant d'ajouter l'élément — permet de confirmer la communication SNMP sans attendre le premier cycle de polling.

---

<img width="1366" height="768" alt="Capture d’écran (2916)" src="https://github.com/user-attachments/assets/cd47db5f-3d17-49a2-afd4-ac3b1697b055" />


## 7. Vérification finale

**Surveillance → Hosts** : les 5 hôtes doivent afficher un badge de disponibilité vert (ZBX pour les agents, SNMP pour les équipements réseau).

| Hôte | Méthode | Résultat |
|---|---|---|
| Zabbix server | Agent (local) | ✅ |
| Ubuntu-Desktop-CDEC | Agent | ✅ |
| WinServer-CDEC | Agent | ✅ |
| VyOS-Router-CDEC | SNMP | ✅ |
| pfSense-Firewall-CDEC | SNMP | ✅ |

**Statut** : supervision de bout en bout opérationnelle sur l'ensemble du périmètre technique retenu.

**Prochaine étape** : configuration des déclencheurs (triggers) et des notifications d'alerte, puis tests de charge/panne (iperf3, hping3, stress-ng) pour valider le déclenchement des alertes.


## 21. Prochaines étapes

1. Simuler des scénarios de panne/charge (`iperf3`, `hping3`, `stress-ng`) pour valider la réactivité des alertes
2. Documenter les résultats en vue du chapitre "Mise en œuvre" du mémoire
3. Préparer la phase d'implémentation réelle sur l'infrastructure de la CDEC, en réutilisant les configurations validées ici

---

## Annexe — Commandes de référence rapide

### VyOS – configuration réseau + NAT complète
```
configure
set interfaces ethernet eth0 address dhcp
set interfaces ethernet eth1 address 10.0.0.1/30
set nat source rule 100 outbound-interface name eth0
set nat source rule 100 source address 192.168.47.0/24
set nat source rule 100 translation address masquerade
set nat source rule 200 outbound-interface name eth0
set nat source rule 200 source address 10.0.0.0/30
set nat source rule 200 translation address masquerade
commit
save
```

### Debian – configuration IP statique
```
auto ens33
iface ens33 inet static
    address 192.168.47.10
    netmask 255.255.255.0
    gateway 192.168.47.254
    dns-nameservers 8.8.8.8 1.1.1.1
```

### Vérification Npcap (Windows)
```cmd
dir C:\Windows\System32\wpcap.dll
dir C:\Windows\SysWOW64\wpcap.dll
```

---

*Document rédigé dans le cadre du mémoire d'ingénieur en Management des Systèmes d'Information — Mise en place d'une solution de supervision réseau Zabbix à la CDEC.*
