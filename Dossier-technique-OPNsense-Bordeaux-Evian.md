# Dossier-technique-OPNsense-Evian-Bordeaux
## Dossier technique — Interconnexion des sites de Bordeaux et d'Evian

**Configuration des pare-feu OPNsense : VLAN, VPN IPsec site à site, relais DHCP et détection d'intrusion**

|  |  |
| --- | --- |
| **Projet** | Infrastructure réseau — sites de Bordeaux et d'Evian |
| **Périmètre** | OPNsense R1 (Bordeaux, cœur de réseau), OPNsense R2 (Evian) |
| **Domaine** | it.lan |
| **Version** | 1.0 |
| **Date** | 15/10/2026 |
| **Auteur** | Juninho Loungila |
| **Classification** | Diffusion restreinte |

### Contrôle du document

#### Historique des versions

| Version | Date | Auteur | Description |
| --- | --- | --- | --- |
| 1.0 | 15/10/2026 | Juninho Loungila | Création du document |

#### Classification

Ce document décrit l'architecture et la configuration d'équipements d'infrastructure. Sa diffusion est limitée aux personnes en charge de l'exploitation et de la validation du projet. Les mots de passe et secrets partagés n'y figurent pas : ils sont remplacés par la mention `<CLE_PRE_PARTAGEE>` ou `<MOT_DE_PASSE>` et conservés dans le coffre-fort de mots de passe de l'équipe.

### Sommaire

1.  Introduction
2.  Architecture
3.  Configuration des pare-feu OPNsense
4.  Synthèse des mesures de sécurité
5.  Cahier de recette
6.  Annexe A — Fiche de configuration consolidée de R2
7.  Annexe B — Points à valider et corrections apportées
8.  Annexe C — Glossaire

## 1\. Introduction

### 1.1 Objet du document

Ce document constitue le dossier technique de l'interconnexion des sites de Bordeaux et d'Evian. Il décrit les choix d'architecture, la configuration appliquée sur les pare-feu OPNsense et les informations associées. Il sert de référence pour l'exploitation, la maintenance et les évolutions futures.

Les deux pare-feu sont désignés ainsi dans l'ensemble du document :

| Désignation | Site | Rôle |
| --- | --- | --- |
| **R1** | Bordeaux | Routeur au cœur du réseau, passerelle des VLAN du site, relais DHCP |
| **R2** | Evian | Routeur du site distant, passerelle des VLAN d'Evian, extrémité du tunnel IPsec, sonde de détection d'intrusion |

### 1.2 Périmètre

| Inclus | Exclus (traités dans d'autres documents) |
| --- | --- |
| Déclaration des VLAN et adressage des interfaces de R2 | Commutateurs Cisco BDX-SW01, BDX-SW02 et EVN-SW01 \[R3\] |
| Tunnel IPsec IKEv2 route-based entre R2 et R1 (côté R2) | Serveur Windows BDX-SRV01 : AD DS, DNS, DHCP, NPS, AD CS \[R3\] |
| Passerelles, route statique et règles de filtrage associées | Détail des règles de filtrage par flux : matrice des flux \[R2\] |
| Relais DHCP vers BDX-SRV01 | Configuration complète de R1 côté tunnel IPsec (symétrique de celle de R2) |
| Détection d'intrusion Suricata (IDS) sur R2 |  |

### 1.3 Documents de référence

| Référence | Document |
| --- | --- |
| \[R1\] | Plan d'adressage IP du projet |
| \[R2\] | Matrice des flux |
| \[R3\] | Dossier technique — Infrastructure réseau du site de Bordeaux (commutateurs et serveur BDX-SRV01) |

## 2\. Architecture

### 2.1 Vue d'ensemble

```
                                    Internet
                                        |
              +-------------------------+--------------------------+
              |                                                    |
 +-------------------------+                          +-------------------------+
 |       OPNsense R1       |    Tunnel IPsec IKEv2    |       OPNsense R2       |
 |    Bordeaux (coeur)     |==========================|          Evian          |
 |    WAN 188.231.29.14    |  ESP AES-256 / SHA-256   |    WAN 188.231.29.20    |
 |   reseaux 10.0.0.0/16   |  GW tunnel 172.16.100.1  |   reseaux 10.1.0.0/16   |
 |       relais DHCP       |                          |   IDS Suricata (WAN)    |
 +-------------------------+                          +-------------------------+
              |                                                    |
        trunk 802.1Q                                         trunk 802.1Q
              |                                                    |
   +--------------------+                               +--------------------+
   |  BDX-SW01 / SW02   |                               |      EVN-SW01      |
   +--------------------+                               +--------------------+
              |                                                    |
   BDX-SRV01 10.0.30.253                                Postes et equipements
   (DHCP, DNS, AD)                                      VLAN 10 a 90
```

L'architecture repose sur quatre principes :

*   **Segmentation par VLAN.** Chaque catégorie d'usage (utilisateurs, imprimantes, serveurs, VPN, Wi-Fi, administration) dispose de son propre réseau, porté par une interface OPNsense dédiée. Les flux entre VLAN passent obligatoirement par le pare-feu.
*   **Interconnexion route-based.** Le tunnel IPsec est une interface virtuelle routée : le trafic inter-sites est dirigé par une route statique, et non par des politiques de sécurité figées dans la configuration IPsec.
*   **Services centralisés à Bordeaux.** Les adresses sont distribuées par BDX-SRV01. Les pare-feu ne servent pas d'adresses : ils relaient les requêtes DHCP.
*   **Détection des menaces.** Un moteur de détection d'intrusion analyse le trafic de R2 avec des listes de signatures mises à jour chaque jour.

### 2.2 Plan d'adressage

#### Site de Bordeaux (R1)

Le plan d'adressage de Bordeaux est décrit dans \[R3\]. Seul le préfixe agrégé utilisé pour le routage inter-sites est rappelé ici.

| Réseau | Usage |
| --- | --- |
| 10.0.0.0/16 | Ensemble des VLAN de Bordeaux (10.0.\<VLAN\>.0/24, passerelle .254) |

#### Site d'Evian (R2)

| VLAN | Description du périphérique | Interface OPNsense | Réseau | Passerelle (R2) | Usage |
| --- | --- | --- | --- | --- | --- |
| 10 | vlan10Utilisateur | Interface utilisateur | 10.1.10.0/24 | 10.1.10.254 | Postes utilisateurs |
| 20 | vlan20Imprimante | Interface Imprimante | 10.1.20.0/24 | 10.1.20.254 | Imprimantes |
| 30 | vlan30Serveur | Interface Serveur | 10.1.30.0/24 | 10.1.30.254 | Serveurs |
| 40 | vlan40VPN | Interface VPN | 10.1.40.0/24 | 10.1.40.254 | Accès VPN |
| 50 | vlan50WifiUtilisateur | Interface Wifi Utilisateur | 10.1.50.0/24 | 10.1.50.254 | Wi-Fi des employés |
| 60 | vlan60WifiMobile | Interface Wifi Mobile | 10.1.60.0/24 | 10.1.60.254 | Wi-Fi des terminaux mobiles |
| 70 | vlan70WifiInvite | Interface Wifi Invite | 10.1.70.0/24 | 10.1.70.254 | Wi-Fi invités |
| 90 | vlan90Admin | Interface Admin | 10.1.90.0/24 | 10.1.90.254 | Administration des équipements |

**Info :**

*   Les passerelles suivent la convention du site de Bordeaux (adresse .254 de chaque VLAN). Sur les captures de configuration, seule l'adresse 10.1.20.254/24 (VLAN 20) est visible ; les autres adresses appliquent la même règle et sont à contrôler lors de la recette.
*   Le VLAN 80 (Black Hole) n'est pas déclaré sur le pare-feu : il n'est ni routé ni relayé, ce qui le maintient isolé.
*   Les réseaux de Bordeaux (10.0.0.0/16) et d'Evian (10.1.0.0/16) ne se chevauchent pas, condition nécessaire au routage inter-sites.

#### Adressage de l'interconnexion

| Élément | Valeur |
| --- | --- |
| IP publique de R1 (site A, Bordeaux) | 188.231.29.14 |
| IP publique de R2 (site B, Evian) | 188.231.29.20 |
| Passerelle WAN de R2 | 188.231.29.3 |
| Passerelle du tunnel (interface TunnelR2R1) | 172.16.100.1 |

### 2.3 Inventaire des équipements

| Équipement | Modèle / système | Adresse IP | Rôle |
| --- | --- | --- | --- |
| R1 | Pare-feu OPNsense | WAN 188.231.29.14 ; .254 de chaque VLAN de Bordeaux | Cœur de réseau, routage inter-VLAN, relais DHCP, extrémité du tunnel |
| R2 | Pare-feu OPNsense | WAN 188.231.29.20 ; .254 de chaque VLAN d'Evian | Routage inter-VLAN, extrémité du tunnel, IDS |
| BDX-SRV01 | Windows Server 2025 | 10.0.30.253 | Serveur DHCP, DNS et annuaire (voir \[R3\]) |
| EVN-SW01 | Commutateur Cisco | Voir \[R3\] | Commutation du site d'Evian |

Sur R2, les cartes physiques sont `re0` (affectée à l'interface WAN) et `re1` (affectée à l'interface LAN). Les interfaces VLAN sont créées sur `re0` (voir le point de vigilance en 3.2.1).

### 2.4 Répartition des rôles réseau

| Fonction | Porteur | Remarque |
| --- | --- | --- |
| Routage entre VLAN d'un site | R1 (Bordeaux) / R2 (Evian) | Passerelle .254 de chaque VLAN |
| Filtrage des flux | R1 et R2 | Règles par interface, voir \[R2\] |
| Interconnexion des sites | R1 et R2 | IPsec IKEv2, mode routé |
| Relais DHCP | OPNsense | Requêtes transmises à BDX-SRV01 (10.0.30.253) |
| Attribution des adresses | BDX-SRV01 | Serveur DHCP unique |
| Détection d'intrusion | R2 | Suricata, signatures abuse.ch |

## 3\. Configuration des pare-feu OPNsense

### 3.1 Principes retenus

| Domaine | Choix | Info |
| --- | --- | --- |
| Segmentation | Un VLAN, une interface, une passerelle .254 | Chaque catégorie d'usage est isolée et filtrée |
| Tunnel | IPsec IKEv2, route-based, clé pré-partagée | Chiffrement ESP AES-256, intégrité SHA-256, groupe MODP-2048 |
| Acheminement inter-sites | Route statique vers 10.0.0.0/16 via la passerelle du tunnel | Le trafic est dirigé par le routage |
| Distribution des adresses | Relais DHCP vers BDX-SRV01 | Un serveur DHCP unique, aucune étendue sur les pare-feu |
| Détection des menaces | Suricata en mode IDS sur l'interface WAN | Signatures abuse.ch, mises à jour quotidiennes |
| Secrets | Clé pré-partagée générée dans le gestionnaire de mots de passe | Jamais stockée dans ce document |

Après chaque enregistrement dans l'interface web, il faut cliquer sur **Apply** (ou **Apply changes**) pour que la modification soit active.

### 3.2 Segmentation réseau : VLAN

#### 3.2.1 Déclaration des interfaces VLAN

**Objectif :** créer sur R2 un périphérique VLAN par réseau logique, au-dessus de l'interface physique qui porte le trunk.

**Chemin :** Interfaces → Devices → VLAN → bouton **+**

| Paramètre | Valeur retenue |
| --- | --- |
| Device | Vide (nom généré automatiquement : vlan01, vlan02...) |
| Parent | `re0` |
| VLAN tag | Identifiant du VLAN (10, 20, 30, 40, 50, 60, 70, 90) |
| VLAN priority | Best Effort (0, valeur par défaut) |
| Description | `vlan<tag><Usage>`, par exemple `vlan10Utilisateur` |

Enregistrer avec **Save**, puis **Apply**.

**Résultat :** huit périphériques VLAN sont listés (vlan01 à vlan08), associés aux tags 10, 20, 30, 40, 50, 60, 70 et 90.

**Info :**

*   Les VLAN doivent être déclarés dans les deux équipements qui les transportent : les tags du pare-feu et ceux des commutateurs doivent être identiques.
*   La description suit une convention sans espace ni accent, comme les noms de groupes de \[R3\].

**Point de vigilance :** sur les captures, le parent des VLAN est `re0`, carte rattachée à l'interface WAN. Le trunk vers le commutateur est normalement porté par la carte du côté LAN (`re1`). Ce choix est à confirmer avant la remise du dossier (voir l'annexe B).

#### 3.2.2 Assignation logique des interfaces

**Objectif :** transformer chaque périphérique VLAN en interface OPNsense configurable.

**Chemin :** Interfaces → Assignments → bouton **+**

| Paramètre | Valeur retenue |
| --- | --- |
| Identifier | Attribué automatiquement (opt1, opt2...) |
| Lock | Coché (empêche la suppression accidentelle) |
| Description | Nom explicite, par exemple `Interface utilisateur` |
| Device | Le périphérique VLAN correspondant, par exemple `vlan01 vlan10Utilisateur` |

Cliquer sur **Save**, puis **Apply**.

**Résultat :** huit interfaces apparaissent dans le menu _Interfaces_ : Admin, Imprimante, Serveur, utilisateur, VPN, Wifi Invite, Wifi Mobile et Wifi Utilisateur, en plus des interfaces LAN et WAN d'origine.

#### 3.2.3 Configuration de l'adressage IP

**Objectif :** activer chaque interface et lui donner l'adresse de passerelle de son VLAN.

**Chemin :** Interfaces → \[nom de l'interface\]

| Paramètre | Valeur retenue |
| --- | --- |
| Enable | Coché (_Enable Interface_) |
| Lock | Coché (_Prevent interface removal_) |
| Description | Nom de l'interface, par exemple `Interface Imprimante` |
| Block private networks | Décoché |
| Block bogon networks | Décoché |
| IPv4 Configuration Type | Static IPv4 |
| IPv6 Configuration Type | None |
| IPv4 address | Adresse de la passerelle et masque, par exemple `10.1.20.254` / `24` |
| IPv4 gateway rules | Disabled |

Cliquer sur **Save**, puis **Apply changes**.

**Info :**

*   Les options _Block private networks_ et _Block bogon networks_ sont décochées : elles n'ont de sens que sur l'interface WAN. Sur une interface interne, elles bloqueraient le trafic légitime des réseaux privés.
*   Le type d'adressage IPv6 est réglé sur _None_ : le projet n'utilise pas IPv6.
*   La passerelle de l'interface reste vide (_Disabled_) : une interface interne est la passerelle des postes de son VLAN et n'a pas de passerelle amont.

#### 3.2.4 Règles de filtrage des interfaces VLAN

**Objectif :** autoriser les flux nécessaires en sortie de chaque VLAN, le trafic entrant sur une interface étant bloqué par défaut.

**Chemin :** Firewall → Rules → \[nom de l'interface\] → bouton **+**

| Paramètre | Valeur retenue |
| --- | --- |
| Enabled | Coché |
| Description | Nom de la règle, par exemple `Admin` |
| Interface | Interface du VLAN concerné, par exemple `InterfaceAdmin` |
| Action | Pass |
| Protocole | Selon le flux autorisé |
| Source | `<nom de l'interface> net` |
| Destination | Selon la politique de filtrage (\[R2\]) |

Cliquer sur **Save**, puis **Apply changes**.

**Info :** OPNsense applique une politique de refus par défaut sur les interfaces ajoutées : tant qu'aucune règle _Pass_ n'existe, le VLAN n'a aucun accès. Les règles sont à restreindre aux flux listés dans la matrice \[R2\] ; une règle « tout autoriser » ne doit servir que de point de départ pendant les tests.

### 3.3 VPN IPsec site à site (route-based)

#### 3.3.1 Paramètres de l'interconnexion

| Paramètre | Site A : R1 (Bordeaux) | Site B : R2 (Evian) |
| --- | --- | --- |
| IP publique | 188.231.29.14 | 188.231.29.20 |
| Identifiant IKE | 188.231.29.14 | 188.231.29.20 |
| Version IKE | IKEv2 | IKEv2 |
| Authentification | Clé pré-partagée | Clé pré-partagée |
| Chiffrement ESP | aes256-sha256-modp2048 | aes256-sha256-modp2048 |
| Mode | Tunnel, routé | Tunnel, routé |

La configuration décrite ci-dessous est celle de **R2**. R1 applique la configuration symétrique : adresses locale et distante inversées, même clé pré-partagée.

#### 3.3.2 Définition de la connexion

**Chemin :** VPN → IPsec → Connections → bouton **+**

| Paramètre | Valeur retenue |
| --- | --- |
| Enabled | Coché |
| Proposals | `default` |
| Version | IKEv2 |
| MOBIKE | Coché |
| Local addresses | 188.231.29.20 |
| Remote addresses | 188.231.29.14 |
| DPD delay (s) | Vide (valeur par défaut) |
| Pools | Aucun |
| Description | `IPSEC R2-R1` |

**Info :**

*   IKEv2 est le protocole d'échange de clés retenu : plus robuste et plus rapide à rétablir que IKEv1.
*   Les adresses locale et distante sont les adresses publiques des deux pare-feu.
*   MOBIKE est coché sur la configuration relevée. Entre deux sites à adresse fixe, cette option n'est pas nécessaire.

#### 3.3.3 Authentification locale et distante

**Chemin :** VPN → IPsec → Connections → connexion `IPSEC R2-R1` → sections _Local Authentication_ et _Remote Authentication_

| Section | Paramètre | Valeur retenue |
| --- | --- | --- |
| Local Authentication | Enabled | Coché |
|  | Connection | `IPSEC R2-R1` |
|  | Round | 0 |
|  | Authentication | Pre-Shared Key |
|  | ID (Local Identifier) | 188.231.29.20 |
|  | Certificates | Aucun |
|  | Description | `Evian` |
| Remote Authentication | Enabled | Coché |
|  | Round | 0 |
|  | Authentication | Pre-Shared Key |
|  | ID (Remote Identifier) | 188.231.29.14 |
|  | Description | `BDX` |

**Info :** chaque pare-feu s'identifie par son adresse IP publique. La clé pré-partagée associée à ces deux identifiants est déclarée en 3.3.5.

#### 3.3.4 Association de sécurité enfant (Children, phase 2)

**Chemin :** VPN → IPsec → Connections → connexion `IPSEC R2-R1` → section _Children_ → bouton **+**

| Paramètre | Valeur retenue |
| --- | --- |
| Enabled | Coché |
| Connection | `IPSEC R2-R1` |
| Mode | Tunnel |
| Policies | Décoché |
| Start action | Trap |
| DPD action | Clear |
| Reqid | 10 |
| ESP proposals | `aes256-sha256-modp2048` |
| Local | `0.0.0.0/0` |
| Remote | `0.0.0.0/0` |
| Description | `Lan R2 vers R1` |

**Résultat :** une ligne _Lan R2 vers R1_ apparaît dans le tableau _Children_ avec les réseaux locaux et distants `0.0.0.0/0`.

**Info :**

*   Les sélecteurs de trafic `0.0.0.0/0` combinés à _Policies_ décoché définissent un tunnel **route-based** : toutes les destinations sont admises dans le tunnel et c'est la table de routage (3.3.7) qui choisit ce qui y passe.
*   _Start action = Trap_ : le tunnel s'établit à la première demande de trafic.
*   _DPD action = Clear_ : en cas de pair injoignable, l'association de sécurité est supprimée puis renégociée.
*   _Reqid_ identifie l'association de sécurité et doit correspondre à celui de l'interface de tunnel virtuelle (VTI).
*   Les algorithmes ESP doivent être identiques sur les deux pare-feu, faute de quoi la phase 2 ne s'établit pas.

**Prérequis :** l'interface de tunnel virtuelle `TunnelR2R1` (VPN → IPsec → Virtual Tunnel Interfaces, puis assignation dans Interfaces → Assignments) doit exister pour être sélectionnée comme interface de la passerelle en 3.3.6. Sa création n'est pas illustrée dans la procédure de départ : ses paramètres sont à documenter lors de la validation (annexe B).

#### 3.3.5 Clé pré-partagée

**Chemin :** VPN → IPsec → Pre-Shared Keys → bouton **+**

| Paramètre | Valeur retenue |
| --- | --- |
| Local Identifier | 188.231.29.20 |
| Remote Identifier | 188.231.29.14 |
| Pre-Shared Key | `<CLE_PRE_PARTAGEE>` |
| Type | PSK |
| Description | `PSK R2-R1` |

**Info :**

*   La clé est générée par le gestionnaire de mots de passe de l'équipe, avec une longueur et une entropie élevées, puis saisie à l'identique sur R1 et R2.
*   Les identifiants local et distant doivent correspondre à ceux de l'authentification (3.3.3), dans le sens propre à chaque pare-feu.

#### 3.3.6 Passerelles

**Chemin :** System → Gateways → Configuration → bouton **+**

| Paramètre | Passerelle WAN | Passerelle du tunnel |
| --- | --- | --- |
| Name | `WAN_GW` | `GW-IPSEC-R1` |
| Interface | WAN | `TunnelR2R1` |
| Address Family | IPv4 | IPv4 |
| Priority | 255 | 255 |
| IP Address | 188.231.29.3 | 172.16.100.1 |
| Upstream Gateway | Coché | Décoché |
| Far Gateway | Décoché | Décoché |
| Disable Gateway Monitoring | Coché | Coché |
| Description | `Interface WAN Gateway` | `GW` |

Cliquer sur **Save**, puis **Apply**.

**Info :**

*   La passerelle WAN est la sortie par défaut vers Internet (_Upstream Gateway_) : elle est marquée _active_ dans la liste.
*   La passerelle du tunnel n'est pas une passerelle amont : elle sert uniquement de prochain saut pour la route inter-sites.
*   La supervision des passerelles est désactivée : l'état des passerelles n'est pas basculé automatiquement en cas de perte du lien.

#### 3.3.7 Routage statique inter-sites

**Chemin :** System → Routes → Configuration → bouton **+**

| Paramètre | Valeur retenue |
| --- | --- |
| Enabled | Coché |
| Network Address | `10.0.0.0/16` |
| Gateway | `GW-IPSEC-R1 - 172.16.100.1` |
| Description | `Route vers site routeur 1` |

Cliquer sur **Save**, puis **Apply**.

**Info :**

*   Tout paquet de R2 à destination de Bordeaux (10.0.0.0/16) est envoyé dans le tunnel.
*   OPNsense affiche un rappel : une route statique ne doit pas viser un réseau déjà affecté à une interface du pare-feu, mais seulement un réseau joignable via un autre routeur. C'est bien le cas de 10.0.0.0/16 vu depuis R2.

#### 3.3.8 Validation de la connexion IPsec

**Chemin :** VPN → IPsec → Status Overview

| Phase | Élément contrôlé | Résultat constaté |
| --- | --- | --- |
| Phase 1 | Connexion `IPSEC R2-R1`, version | IKEv2, icône verte (établie) |
| Phase 1 | Local ID / Remote ID | 188.231.29.20 / 188.231.29.14 |
| Phase 2 | Réseaux locaux / distants | 0.0.0.0/0 / 0.0.0.0/0 |
| Phase 2 | État | INSTALLED |
| Phase 2 | Volumes échangés | Octets entrants et sortants non nuls |

#### 3.3.9 Règles de filtrage sur l'interface IPsec

**Objectif :** autoriser le trafic de Bordeaux vers Evian à sortir du tunnel et à entrer dans le réseau d'Evian.

**Chemin :** Firewall → Rules → IPsec → bouton **+**

| Paramètre | Valeur retenue |
| --- | --- |
| Interface | IPsec |
| Quick | Coché |
| Action | Pass |
| Direction | In |
| Version | IPv4 |
| Protocol | any |
| Source | Single host or Network : `10.0.0.0/16` |
| Destination | Single host or Network : `10.1.0.0/16` |
| Log | Décoché |
| Description | `IPSEC Encapsulation` |

Cliquer sur **Save**, puis **Apply changes**.

**Info :**

*   La règle est limitée aux deux réseaux de l'interconnexion : le tunnel n'accepte pas d'autre source.
*   Cette règle couvre le sens Bordeaux → Evian. Le trafic initié depuis Evian est filtré par les règles des interfaces VLAN de R2 (3.2.4), avec 10.0.0.0/16 comme destination.
*   Le protocole est réglé sur _any_ : à restreindre aux services nécessaires selon \[R2\].

### 3.4 Relais DHCP

#### Principe

Les requêtes DHCP sont émises en diffusion et ne franchissent pas les routeurs. Le pare-feu, passerelle de chaque VLAN, relaie donc les requêtes vers le serveur DHCP BDX-SRV01 (10.0.30.253). Il n'attribue lui-même aucune adresse.

#### 3.4.1 Déclaration du serveur DHCP (destination)

**Chemin :** Services → DHCRelay → Configuration → section _Destinations_ → bouton **+**

| Paramètre | Valeur retenue |
| --- | --- |
| Name | `BDX-SRV01` |
| Server | `10.0.30.253` |

Cliquer sur **Save**.

**Résultat :** l'adresse 10.0.30.253 est listée dans le tableau _Destinations_.

#### 3.4.2 Agent de relais (un par VLAN)

**Chemin :** Services → DHCRelay → Configuration → section _Relays_ → bouton **+**

| Paramètre | Valeur retenue |
| --- | --- |
| Enabled | Coché |
| Interface | Interface du VLAN à desservir, par exemple `Vlan10Utilisateur` |
| Destination | `BDX-SRV01` |
| Agent Information | Décoché |
| Depend on (CARP) | None |

Cliquer sur **Save** puis **Apply**. Un relais est créé pour chaque VLAN qui reçoit ses adresses par DHCP.

**Résultat :** sept relais sont listés avec un voyant vert, dont `Vlan10Utilisateur`, `Vlan20Imprimante`, `Vlan40VPN` et `Vlan50Wifiutilisateur`, tous associés à la destination `BDX-SRV01`.

**Info :**

*   _Agent Information_ est décoché : le relais n'ajoute pas l'option 82. Ce réglage est cohérent avec les commutateurs, sur lesquels l'insertion de l'option 82 est désactivée (\[R3\]).
*   _Depend on (CARP)_ reste à _None_ : le projet n'utilise pas de haute disponibilité.
*   Les VLAN 30 et 90 n'ont pas de relais : leurs équipements utilisent des adresses statiques.
*   Le serveur DHCP doit disposer d'une étendue pour chaque réseau relayé : sans étendue correspondant au réseau de la passerelle, aucune adresse n'est délivrée.

### 3.5 Détection d'intrusion (IDS) sur R2

#### 3.5.1 Activation du service

**Chemin :** Services → Intrusion Detection → Administration → onglet _Settings_

| Paramètre | Valeur retenue |
| --- | --- |
| Enabled | Coché |
| Capture mode | PCAP live mode (IDS) |
| Promiscuous mode | Décoché |
| Interfaces | WAN |
| Pattern matcher | Hyperscan |
| Enable syslog alerts | Décoché |
| Enable eve syslog output | Décoché |
| Rotate log | Weekly |
| Save logs | 4 |

Cliquer sur **Apply**.

**Info :**

*   Le mode _PCAP live mode_ fonctionne en **détection** : le moteur Suricata observe une copie du trafic et génère des alertes, mais il ne bloque rien. Le mode de blocage (IPS) repose sur _Netmap_ et dépend de la compatibilité de la carte réseau.
*   L'interface surveillée est le WAN : le moteur analyse le trafic qui entre et sort du site.
*   _Hyperscan_ est le moteur de recherche de motifs le plus rapide ; il accélère l'analyse des signatures.
*   Les journaux sont pivotés chaque semaine et quatre rotations sont conservées.

#### 3.5.2 Jeux de règles

**Chemin :** Services → Intrusion Detection → Administration → onglet _Download_

| Paramètre | Valeur retenue |
| --- | --- |
| Jeux de règles cochés | abuse.ch/Feodo Tracker, abuse.ch/SSL Fingerprint Blacklist, abuse.ch/SSL IP Blacklist, abuse.ch/ThreatFox, abuse.ch/URLhaus |
| Action | **Download & Update Rules**, puis **enable** pour la sélection |

**Résultat :** l'onglet _Rules_ liste plus de 123 000 signatures issues d'abuse.ch, avec l'action _alert_ par défaut.

**Info :** les jeux abuse.ch ciblent les serveurs de commande et de contrôle de logiciels malveillants, les certificats frauduleux et les URL de diffusion de malwares : peu de faux positifs et un coût de traitement limité.

#### 3.5.3 Politique de gestion des règles

**Chemin :** Services → Intrusion Detection → Policy → onglet _Policies_ → bouton **+**

| Paramètre | Valeur retenue |
| --- | --- |
| Enabled | Coché |
| Priority | 0 |
| Rulesets | Les cinq jeux abuse.ch téléchargés |
| Action | Drop |
| Rules : classtype | bad-unknown, trojan-activity |
| Rules : confidence\_level | Aucune sélection |
| Rules : first\_seen | Aucune sélection |
| New action | Drop |
| Description | Vide |

Cliquer sur **Save**, puis **Apply**.

**Info :**

*   Une politique modifie l'action de toutes les règles qui correspondent aux critères, et reste appliquée après chaque mise à jour des signatures. C'est plus durable qu'une modification règle par règle dans l'onglet _Rules_.
*   La politique cible les classes `bad-unknown` et `trojan-activity` : les menaces les plus probables et les plus critiques.

**Point de vigilance :** la nouvelle action est _Drop_, alors que le moteur fonctionne en mode IDS (3.5.1). Dans ce mode, les règles _Drop_ génèrent une alerte sans bloquer le trafic. Pour obtenir un blocage effectif, il faut passer en mode IPS (_Capture mode_ = Netmap) après avoir vérifié la compatibilité de la carte réseau.

#### 3.5.4 Mise à jour automatique des signatures

**Chemin :** System → Settings → Cron → onglet _Jobs_ → bouton **+** (ou Services → Intrusion Detection → Administration → onglet _Schedule_)

| Paramètre | Valeur retenue |
| --- | --- |
| Enabled | Coché |
| Description | `ids rule updates` |
| Command | Update and reload intrusion detection rules |
| Minutes | 0 |
| Hours | 0 |
| Day of the month | `*` |
| Months | `*` |
| Days of the week | `*` |
| Parameters | Vide |

Cliquer sur **Save**, puis **Apply**.

**Résultat :** le job _ids rule updates_ apparaît dans la liste des tâches planifiées. Il s'exécute chaque jour à 00h00.

**Info :**

*   La mise à jour nocturne limite l'impact du téléchargement sur la bande passante pendant les heures de production.
*   La commande _Update and reload intrusion detection rules_ télécharge les signatures puis recharge le moteur sans le redémarrer.
*   Il faut sélectionner la commande dans la liste déroulante : la valeur affichée par défaut (_Automatic firmware update_) lancerait la mise à jour du pare-feu et non celle des signatures.

### 3.6 Sauvegarde et contrôles

La configuration d'OPNsense est stockée dans un fichier unique. Une sauvegarde est exportée après chaque modification importante :

**Chemin :** System → Configuration → Backups → **Download configuration**

Le fichier contient la clé pré-partagée : il est conservé dans le coffre-fort de l'équipe, jamais sur un partage ordinaire.

| Contrôle | Chemin | Information vérifiée |
| --- | --- | --- |
| Périphériques VLAN | Interfaces → Devices → VLAN | Huit VLAN, tags et parent corrects |
| Interfaces assignées | Interfaces → Assignments | Chaque VLAN est associé à une interface |
| Adressage | Interfaces → Overview | Adresse de chaque interface, état _up_ |
| État du tunnel | VPN → IPsec → Status Overview | Phase 1 établie, phase 2 _INSTALLED_ |
| Passerelles | System → Gateways → Configuration | `WAN_GW` active, `GW-IPSEC-R1` présente |
| Routes | System → Routes → Configuration | Route 10.0.0.0/16 via le tunnel |
| Règles de filtrage | Firewall → Rules | Règles par interface et sur IPsec |
| Relais DHCP | Services → DHCRelay → Configuration | Relais actifs (voyant vert) |
| Alertes IDS | Services → Intrusion Detection → Administration → onglet _Alerts_ | Alertes générées par les signatures |
| Tâches planifiées | System → Settings → Cron | Job de mise à jour des signatures actif |
| Test de joignabilité | Interfaces → Diagnostics → Ping | Réponse de 10.0.30.253 depuis R2 |

## 4\. Synthèse des mesures de sécurité

| Mesure | Équipement | Risque couvert |
| --- | --- | --- |
| Segmentation en VLAN et filtrage par interface | R1, R2 | Propagation latérale entre catégories de postes |
| VLAN 80 non routé et non relayé | R1, R2 | Connexion d'un équipement non autorisé |
| Tunnel IPsec IKEv2, ESP AES-256 / SHA-256 / MODP-2048 | R1, R2 | Interception et altération du trafic inter-sites |
| Clé pré-partagée générée et stockée dans un coffre-fort | R1, R2 | Divulgation ou devinette de la clé |
| Règle IPsec limitée à 10.0.0.0/16 → 10.1.0.0/16 | R2 | Accès au tunnel depuis un réseau tiers |
| Route statique explicite vers le tunnel | R2 | Trafic inter-sites hors du tunnel |
| Relais DHCP vers un serveur unique | R1, R2 | Serveurs DHCP non maîtrisés sur les VLAN |
| Détection d'intrusion Suricata (abuse.ch) | R2 | Communication avec des serveurs malveillants connus |
| Politique _Drop_ sur les classes à risque | R2 | Menaces actives (effective en mode IPS) |
| Mise à jour quotidienne des signatures | R2 | Signatures obsolètes |
| Verrouillage des interfaces (_Lock_) | R1, R2 | Suppression accidentelle d'une interface |

## 5\. Cahier de recette

| N° | Test | Méthode | Résultat attendu | Statut |
| --- | --- | --- | --- | --- |
| R01 | Périphériques VLAN | Interfaces → Devices → VLAN | Huit entrées, tags 10, 20, 30, 40, 50, 60, 70, 90 | Conforme |
| R02 | Interfaces assignées | Interfaces → Assignments | Chaque VLAN associé à une interface | Conforme |
| R03 | Adressage d'une interface | Interfaces → Interface Imprimante | 10.1.20.254/24, Static IPv4 | Conforme |
| R04 | Adressage des autres interfaces | Interfaces → Overview | Passerelle .254 de chaque VLAN, état _up_ | À réaliser |
| R05 | Phase 1 IPsec | VPN → IPsec → Status Overview | IKEv2 établie (icône verte) | Conforme |
| R06 | Phase 2 IPsec | VPN → IPsec → Status Overview | État _INSTALLED_, octets entrants et sortants non nuls | Conforme |
| R07 | Passerelles | System → Gateways → Configuration | `WAN_GW` active, `GW-IPSEC-R1` présente | Conforme |
| R08 | Route inter-sites | System → Routes → Configuration | 10.0.0.0/16 via `GW-IPSEC-R1` | Conforme |
| R09 | Règle sur l'interface IPsec | Firewall → Rules → IPsec | Règle _IPSEC Encapsulation_ présente et active | Conforme |
| R10 | Joignabilité inter-sites | Ping d'un poste du VLAN 10 d'Evian vers 10.0.30.253 | Réponses reçues | À réaliser |
| R11 | Relais DHCP | Services → DHCRelay → Configuration | Relais actifs avec voyant vert, destination `BDX-SRV01` | Conforme |
| R12 | Bail DHCP d'un poste d'Evian | `ipconfig /renew` sur un poste du VLAN 10 | Adresse du réseau 10.1.10.0/24, passerelle .254 | À réaliser |
| R13 | Signatures IDS | Services → Intrusion Detection → onglet _Rules_ | Signatures abuse.ch chargées (plus de 123 000) | Conforme |
| R14 | Politique de gestion des règles | Services → Intrusion Detection → Policy | Politique active, action _Drop_, jeux abuse.ch | Conforme |
| R15 | Mise à jour planifiée | System → Settings → Cron | Job _ids rule updates_ actif, tous les jours à 00h00 | Conforme |
| R16 | Génération d'une alerte | Requête vers un domaine de test abuse.ch | Alerte visible dans l'onglet _Alerts_ | À réaliser |

## 6\. Annexe A — Fiche de configuration consolidée de R2

Les pare-feu OPNsense se configurent par l'interface web : la configuration est présentée sous forme de fiche récapitulative.

```
INTERFACES (parent re0)
  vlan01  tag 10   vlan10Utilisateur       Static IPv4  10.1.10.254/24
  vlan02  tag 20   vlan20Imprimante        Static IPv4  10.1.20.254/24
  vlan03  tag 30   vlan30Serveur           Static IPv4  10.1.30.254/24
  vlan04  tag 40   vlan40VPN               Static IPv4  10.1.40.254/24
  vlan05  tag 50   vlan50WifiUtilisateur   Static IPv4  10.1.50.254/24
  vlan06  tag 60   vlan60WifiMobile        Static IPv4  10.1.60.254/24
  vlan07  tag 70   vlan70WifiInvite        Static IPv4  10.1.70.254/24
  vlan08  tag 90   vlan90Admin             Static IPv4  10.1.90.254/24

IPSEC : connexion "IPSEC R2-R1"
  IKEv2, proposals default, MOBIKE active
  local 188.231.29.20 / remote 188.231.29.14
  auth locale  : PSK, ID 188.231.29.20 (Evian)
  auth distante: PSK, ID 188.231.29.14 (BDX)
  PSK          : <CLE_PRE_PARTAGEE> (description "PSK R2-R1")
  child "Lan R2 vers R1" : mode Tunnel, policies off, start Trap, DPD Clear,
                           reqid 10, ESP aes256-sha256-modp2048,
                           local 0.0.0.0/0, remote 0.0.0.0/0

PASSERELLES
  WAN_GW       WAN         188.231.29.3   priorite 255   upstream, monitoring off
  GW-IPSEC-R1  TunnelR2R1  172.16.100.1   priorite 255   monitoring off

ROUTES
  10.0.0.0/16  via GW-IPSEC-R1 (172.16.100.1)   "Route vers site routeur 1"

PARE-FEU
  Interface IPsec : Pass, In, IPv4, any, 10.0.0.0/16 -> 10.1.0.0/16, quick
  Interfaces VLAN : une regle Pass par interface (a restreindre selon [R2])

RELAIS DHCP
  destination BDX-SRV01 = 10.0.30.253
  7 relais actifs, agent information desactive

IDS (Suricata)
  mode PCAP live (IDS), interface WAN, Hyperscan
  jeux : abuse.ch Feodo, SSL Fingerprint, SSL IP, ThreatFox, URLhaus
  policy : priorite 0, action Drop, classtype bad-unknown + trojan-activity
  cron  : "ids rule updates", tous les jours a 00h00
```

## 7\. Annexe B — Points à valider et corrections apportées

### 7.1 Corrections apportées par rapport à la procédure de départ

| Élément | Procédure de départ | Dossier |
| --- | --- | --- |
| Chemin des VLAN | Interfaces → Other Types → VLAN | Interfaces → Devices → VLAN (menu constaté sur les captures) |
| Chemin du relais DHCP | Services → ISC DHCPv4 (ou Dnsmasq / Relay) → Relay | Services → DHCRelay → Configuration |
| Nom du serveur DHCP | `BDX-SAV01` (faute de frappe) | `BDX-SRV01`, nom du serveur dans \[R3\] |
| Réseau de destination d'une route | Libellé « Disabled Network / Network Address » | Champ _Network Address_ |
| Gestion des règles IDS | Onglet _Rules_, action _Change to Drop_ | Politique (_Policy_), constatée sur les captures et plus durable |
| Mode du moteur IDS | « IPS mode si le blocage est requis » | Mode constaté : _PCAP live mode (IDS)_, sans blocage |
| Commande de la tâche planifiée | Non précisée | _Update and reload intrusion detection rules_ |
| Noms d'interfaces | Orthographes mêlées (`Interfaceutilisateur`, `Interface Imprimante`) | Convention homogène de description des périphériques |

### 7.2 Points à valider avant la remise

| N° | Point | Constat | Action proposée |
| --- | --- | --- | --- |
| V1 | Interface parente des VLAN | Les VLAN sont créés sur `re0`, rattachée à l'interface WAN | Confirmer la carte qui porte le trunk vers EVN-SW01 (normalement `re1`) |
| V2 | Équipement du relais DHCP | La procédure de départ place le relais sur R1 (10.0.0.254), mais les interfaces relayées sur les captures portent les noms des VLAN d'Evian | Confirmer sur quel pare-feu est configuré le relais et corriger le rattachement en 3.4 |
| V3 | Plan d'adressage d'Evian | \[R3\] référence EVN-SW01 en 10.0.90.4 (réseau de Bordeaux), alors que R2 route 10.1.0.0/16 | Aligner \[R1\] et \[R3\] sur le plan 10.1.x d'Evian |
| V4 | Étendues DHCP d'Evian | \[R3\] ne décrit que les étendues 10.0.x de Bordeaux | Créer les étendues 10.1.x sur BDX-SRV01 et vérifier la route de retour vers 10.1.0.0/16 |
| V5 | Mode IDS et action _Drop_ | Politique en _Drop_ mais moteur en mode IDS | Rester en détection, ou passer en IPS après vérification de la carte réseau |
| V6 | Interface de tunnel virtuelle | La création de `TunnelR2R1` (VTI) n'est pas illustrée | Documenter ses paramètres et vérifier que son _Reqid_ vaut 10 |
| V7 | Nom de la connexion IPsec | Le nom `SBEB` apparaît sur une capture, `IPSEC R2-R1` sur les autres | Conserver un seul nom |
| V8 | MOBIKE | Option cochée entre deux sites à adresse fixe | La décocher si elle n'est pas requise |
| V9 | Supervision des passerelles | _Disable Gateway Monitoring_ coché sur les deux passerelles | Activer la supervision de `GW-IPSEC-R1` si un basculement est attendu |
| V10 | Identifiants des interfaces | Nom d'hôte par défaut `OPNsense.internal` | Nommer les pare-feu selon la convention du projet (par exemple BDX-FW01 et EVN-FW01) |

## 8\. Annexe C — Glossaire

| Terme | Définition |
| --- | --- |
| abuse.ch | Organisme publiant des listes de menaces (serveurs de commande et de contrôle, URL de malwares, certificats frauduleux) |
| CARP | Protocole de haute disponibilité entre pare-feu OPNsense |
| Child SA | Association de sécurité enfant d'IPsec (phase 2) qui protège les données échangées |
| Cron | Planificateur de tâches du système |
| DPD | _Dead Peer Detection_ : détection d'un pair IPsec devenu injoignable |
| ESP | _Encapsulating Security Payload_ : protocole IPsec qui chiffre et authentifie les paquets |
| Hyperscan | Moteur de recherche de motifs à haute performance utilisé par Suricata |
| IDS / IPS | Système de détection / de prévention d'intrusion : il alerte (IDS) ou bloque (IPS) |
| IKEv2 | Protocole d'échange de clés d'IPsec, version 2 |
| MOBIKE | Extension d'IKEv2 qui permet de changer d'adresse IP sans rompre le tunnel |
| Netmap | Technique d'accès rapide aux cartes réseau, requise par le mode IPS de Suricata |
| Option 82 | Option DHCP d'information d'agent de relais |
| PSK | _Pre-Shared Key_ : clé secrète partagée entre les deux extrémités d'un tunnel |
| Relais DHCP | Service qui transmet les requêtes DHCP d'un VLAN à un serveur situé sur un autre réseau |
| Reqid | Identifiant de requête reliant une association de sécurité IPsec à son interface de tunnel |
| Route-based | Tunnel IPsec représenté par une interface virtuelle, dont le trafic est dirigé par le routage |
| Suricata | Moteur open source de détection et de prévention d'intrusion intégré à OPNsense |
| Trunk 802.1Q | Lien transportant plusieurs VLAN étiquetés |
| VLAN | Réseau local virtuel : segmentation logique d'un réseau physique |
| VTI | _Virtual Tunnel Interface_ : interface virtuelle associée à un tunnel IPsec routé |