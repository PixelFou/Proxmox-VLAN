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
```

## 4. Procédures de mise en œuvre

### Étape 1 : Configuration réseau sur pfSense
Création des VLANs : Sur l'interface parente vtnet1, déclaration des tags 99 (MGMT), 100 (ADMIN), 200 (EMPLOYES) et 300 (WAITINGROOM).

Assignation et adressage : Activation des interfaces virtuelles avec attribution des passerelles statiques :

VLAN 99 : 192.168.99.1/24

VLAN 100 : 192.168.10.1/24

VLAN 200 : 192.168.20.1/24

VLAN 300 : 192.168.30.1/24

Services DHCP et Pare-feu : Activation des plages DHCP (.100 à .199) sur chaque sous-réseau, création d'un bail statique pour FreeRADIUS (192.168.99.50) et ouverture des règles de filtrage pour les tests.


### Étape 2 : Configuration d'Open vSwitch (OVS)

Création du commutateur virtuel br0, configuration du port Trunk vers pfSense, placement des ports d'accès en VLAN d'attente (300) et création de l'interface interne de management pour joindre le serveur RADIUS :

ovs-vsctl add-br br0
ovs-vsctl add-port br0 ens18
ovs-vsctl set port ens18 trunks=99,100,200,300

ovs-vsctl add-port br0 ens19
ovs-vsctl set port ens19 tag=300
ovs-vsctl add-port br0 ens20
ovs-vsctl set port ens20 tag=300

ovs-vsctl add-port br0 int-mgmt -- set interface int-mgmt type=internal
ovs-vsctl set port int-mgmt tag=99
ip link set int-mgmt up
ip addr add 192.168.99.10/24 dev int-mgmt

### Étape 3 : Configuration de FreeRADIUS

Déclaration des clients (/etc/freeradius/3.0/clients.conf) : Autorisation du commutateur OVS et des sous-réseaux avec le secret partagé SuperSecretRadiusKey.

Activation de PEAP (/etc/freeradius/3.0/mods-enabled/eap) : Configuration de default_eap_type = peap avec authentification interne mschapv2.

Définition des profils et VLANs (/etc/freeradius/3.0/mods-config/files/authorize) :

admin Cleartext-Password := "adminpass"
    Tunnel-Type = VLAN,
    Tunnel-Medium-Type = IEEE-802,
    Tunnel-Private-Group-Id = "100"

employes Cleartext-Password := "employespass"
    Tunnel-Type = VLAN,
    Tunnel-Medium-Type = IEEE-802,
    Tunnel-Private-Group-Id = "200"


### Étape 4 : Déploiement de Hostapd sur l'OVS

Désactivation du service global (systemctl disable --now hostapd) et création de deux fichiers de configuration distincts (/etc/hostapd/hostapd-ens19.conf et hostapd-ens20.conf) pour piloter chaque port d'accès indépendamment :

interface=ens19
driver=wired
ieee8021x=1
eapol_version=2
eapol_key_index_workaround=0
auth_server_addr=192.168.99.50
auth_server_port=1812
auth_server_shared_secret=SuperSecretRadiusKey
acct_server_addr=192.168.99.50
acct_server_port=1813
acct_server_shared_secret=SuperSecretRadiusKey

# Attribution dynamique de VLAN sur le bridge OVS
dynamic_vlan=1
vlan_naming=1
vlan_tagged_interface=br0

Lancement en mode débogage dans deux terminaux distincts :

hostapd -dd /etc/hostapd/hostapd-ens19.conf
hostapd -dd /etc/hostapd/hostapd-ens20.conf


### Étape 5 : Tests et validation côté Client

Sur le poste admin Debian, création du profil /root/eapol.conf et lancement de l'authentification filaire :
wpa_supplicant -i ens18 -c /root/eapol.conf -D wired -dd

Résultats attendus :

Échange EAPOL (0x888e) visible via tcpdump.

Réception d'un paquet Access-Accept depuis FreeRADIUS contenant l'attribut Tunnel-Private-Group-Id.

Bascule automatique du port client vers le VLAN 100 (ou 200) et attribution d'une adresse IP par le serveur DHCP de pfSense dans le bon sous-réseau.


### 5. Axes d'amélioration
Sécurisation des identifiants et certificats :

Remplacer les mots de passe en clair (Cleartext-Password) dans FreeRADIUS par des empreintes chiffrées (NT-Hash) ou raccorder FreeRADIUS à un annuaire d'entreprise (LDAP / Active Directory).

Remplacer les certificats auto-signés par défaut de FreeRADIUS par une infrastructure à clés publiques (PKI) interne afin que les clients vérifient l'identité du serveur RADIUS (protection contre les attaques Man-in-the-Middle).

urcissement du pare-feu (pfSense) :

Remplacer les règles permissives (Any-to-Any) utilisées durant la phase de test par une matrice de flux stricte : interdire aux EMPLOYES (VLAN 200) de joindre le réseau d'administration ADMIN (VLAN 100) et le réseau de gestion MGMT (VLAN 99).