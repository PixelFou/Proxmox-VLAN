# Compte Rendu de TP : Architecture Réseau Dynamique (802.1X, OVS, FreeRADIUS, pfSense)

---

## 1. Intérêt de l'infrastructure

L'objectif de cette architecture est d'automatiser la segmentation réseau de niveau 2 (VLAN) en se basant sur **l'identité de l'utilisateur** plutôt que sur le port physique de connexion ou une configuration IP statique.

* **Sécurité renforcée (802.1X / PEAP) :** Le port réseau reste bloqué tant que l'utilisateur n'est pas authentifié auprès du serveur RADIUS. Les échanges d'identifiants sont encapsulés et chiffrés via un tunnel TLS (PEAP).
* **Isolation et mobilité :** Un utilisateur (Administrateur ou Employé) peut se connecter depuis n'importe quel port géré par le commutateur. Après authentification, il est basculé dynamiquement dans son VLAN dédié (`VLAN 100` pour les Admins, `VLAN 200` pour les Employés) et reçoit automatiquement sa configuration réseau.
* **Mise en quarantaine (Salle d'attente) :** Les équipements non authentifiés ou en cours de connexion sont isolés par défaut dans un VLAN restreint (`VLAN 300 - WAITINGROOM`), empêchant tout accès latéral au reste du système d'information.
* **Approche SDN (Software-Defined Networking) :** L'utilisation d'Open vSwitch permet de piloter virtuellement les règles de commutation et l'attribution des tags VLAN à la volée.

---

## 2. Architecture et Matériel

L'environnement est virtualisé sous **Proxmox** avec des ponts réseau Linux configurés en mode **VLAN aware** (`vmbr1` à `vmbr5`) pour autoriser le transit des trames taguées 802.1Q.

| Composant | Rôle dans l'infrastructure | Détails techniques |
| :--- | :--- | :--- |
| **pfSense** | Routeur, Pare-feu et Serveur DHCP | Gère le routage inter-VLAN, le filtrage des flux et distribue les baux IP via Kea DHCP sur les sous-réseaux (`99`, `100`, `200`, `300`). |
| **Open vSwitch (OVS)** | Commutateur virtuel intelligent (NAS) | Héberge le bridge principal `br0`, gère le lien Trunk vers pfSense, maintient l'interface de gestion `int-mgmt` (VLAN 99) et exécute les daemons `hostapd`. |
| **hostapd** | Authentificateur (Authenticator 802.1X) | Écoute les trames EAPOL sur les interfaces clients (`ens19`/`ens20`), relaie les demandes vers FreeRADIUS et applique le tag VLAN dynamique sur `br0`. |
| **FreeRADIUS** | Serveur d'authentification (AAA) | Centralise la base des utilisateurs, vérifie les identifiants (PEAP/MSCHAPv2) et retourne les attributs RADIUS d'attribution de VLAN. |
| **Client Debian** | Supplicant (Poste utilisateur) | Initie la demande d'authentification filaire 802.1X via `wpa_supplicant` (pilote `wired`) ou `nmcli`. |

---

## 3. Schéma de l'architecture

```text
+-----------------------------------------------------------------------+
|                           Routeur pfSense                             |
|   [WAN: vmbr0]         [LAN / Trunk: vmbr1 (VLANs 99, 100, 200, 300)] |
+-----------------------------------+-----------------------------------+
                                    |
                                    | Lien Trunk 802.1Q
                                    v
+-----------------------------------------------------------------------+
|                       Nœud Debian Open vSwitch                        |
|                         Bridge Virtuel (br0)                          |
|                                                                       |
|  [int-mgmt: 192.168.99.10]                                            |
|        |                                                              |
|        |            +-----------------------+-----------------------+ |
|        |            |                                               | |
|        |   +--------v---------+                           +---------v-+
|        |   | hostapd (ens19)  |                           | hostapd (ens20)
|        |   +--------+---------+                           +---------+ |
+--------|------------|-----------------------------------------------|----+
         |            | Port 802.1X                                   | Port 802.1X
         |            | (Défaut: VLAN 300)                            | (Défaut: VLAN 300)
         |            v                                               v
         |   +-----------------+                             +-----------------+
         |   |  Client Admin   |                             | Client Employé  |
         |   |  -> VLAN 100    |                             |  -> VLAN 200    |
         |   | 192.168.10.0/24 |                             | 192.168.20.0/24 |
         |   +-----------------+                             +-----------------+
         |
         | Requêtes RADIUS (UDP 1812 / 1813)
         v
+-----------------------------------+
|        Serveur FreeRADIUS         |
|      VLAN 99 (192.168.99.50)      |
+-----------------------------------+