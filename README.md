# Réseau d'entreprise segmenté — Cisco Packet Tracer

> **EN** — Segmented SME network lab built with Cisco Packet Tracer: 802.1Q VLANs, router-on-a-stick inter-VLAN routing, OSPF, DHCP, NAT/PAT, extended ACLs, SSH-only management and switch port security. Full device configurations are in [`configs/`](configs/).

Labo réseau d'une PME de quatre services, construit pour mettre en pratique le cours **CCNA3 — Enterprise Networking, Security and Automation**. L'objectif : un réseau segmenté par service, routé, qui sort sur Internet par NAT et dont le réseau invités est isolé du reste.

## Topologie

```
                      198.51.100.1 (hôte "Internet" simulé)
                              │
                         ┌────┴────┐
                         │   ISP   │
                         └────┬────┘  203.0.113.0/30
                         G0/0 │
                         ┌────┴────┐
                         │ R1-EDGE │  NAT/PAT · route par défaut
                         └────┬────┘
                         G0/1 │  10.0.0.0/30 — OSPF area 0
                         ┌────┴────┐
                         │ R2-CORE │  inter-VLAN · DHCP · ACL
                         └────┬────┘
                         G0/0 │  trunk 802.1Q (natif 999)
                         ┌────┴────┐
                         │   SW1   │  VLAN · port-security
                         └────┬────┘
          ┌──────────────┬────┴─────────┬──────────────┐
      PC-ADMIN         PC-RH          SRV-01        PC-INVITE
      VLAN 10          VLAN 20        VLAN 30        VLAN 40
```

## Plan d'adressage

| VLAN | Nom | Réseau | Passerelle | Adressage |
|---|---|---|---|---|
| 10 | ADMIN | 192.168.10.0/24 | 192.168.10.1 | DHCP (.11 → .254) |
| 20 | RH | 192.168.20.0/24 | 192.168.20.1 | DHCP (.11 → .254) |
| 30 | SERVEURS | 192.168.30.0/24 | 192.168.30.1 | Statique (SRV-01 = .10) |
| 40 | INVITES | 192.168.40.0/24 | 192.168.40.1 | DHCP (.11 → .254) |
| 99 | GESTION | 192.168.99.0/24 | 192.168.99.1 | SW1 = .2 |
| 666 | PARKING | — | — | Ports inutilisés |
| 999 | NATIVE | — | — | VLAN natif du trunk |

| Lien | Réseau | Adresses |
|---|---|---|
| R1-EDGE ↔ R2-CORE | 10.0.0.0/30 | .1 / .2 |
| ISP ↔ R1-EDGE | 203.0.113.0/30 | .1 / .2 |

## Ce qui est configuré

- **Segmentation** : 4 VLAN métier + 1 VLAN de gestion, trunk 802.1Q avec VLAN natif dédié et liste de VLAN autorisés.
- **Routage inter-VLAN** en *router-on-a-stick* (sous-interfaces `dot1Q` sur R2-CORE).
- **OSPF** (area 0) entre R1 et R2, interfaces utilisateurs en `passive-interface`, route par défaut propagée par `default-information originate`.
- **DHCP** sur R2-CORE pour les VLAN ADMIN, RH et INVITES, avec plages exclues.
- **NAT/PAT** sur R1-EDGE : tous les VLAN sortent avec l'adresse publique 203.0.113.2.

## Sécurité

- **Réseau invités isolé** : l'ACL étendue `INVITES-IN` ne laisse passer que DHCP, DNS vers le serveur interne et Internet.
- **Administration en SSH v2 uniquement**, comptes locaux, `access-class` qui limite l'accès aux VLAN ADMIN et GESTION, déconnexion après 10 min.
- **Port-security** (2 adresses MAC max, apprentissage *sticky*), **PortFast + BPDU Guard** sur les ports utilisateurs.
- **Ports inutilisés** désactivés et placés dans le VLAN 666, **DTP désactivé** sur le trunk.
- Aucun mot de passe réel dans ce dépôt : les configurations utilisent `CHANGE_ME`.

## Reconstruire le labo

1. Ouvrir **Cisco Packet Tracer 8.x** et placer 3 routeurs **2911** (ISP, R1-EDGE, R2-CORE), 1 commutateur **2960** (SW1), 3 PC et 1 serveur.
2. Câbler :

   | De | Vers |
   |---|---|
   | ISP G0/0 | R1-EDGE G0/0 |
   | R1-EDGE G0/1 | R2-CORE G0/1 |
   | R2-CORE G0/0 | SW1 G0/1 |
   | SW1 Fa0/1 | PC-ADMIN |
   | SW1 Fa0/6 | PC-RH |
   | SW1 Fa0/11 | SRV-01 (192.168.30.10/24, passerelle .1, services DNS + HTTP) |
   | SW1 Fa0/13 | PC-INVITE |

3. Remplacer `CHANGE_ME` dans les fichiers de [`configs/`](configs/), puis coller chaque fichier dans le CLI de l'équipement correspondant (`configure terminal`).
4. Passer les PC en DHCP et lancer les vérifications ci-dessous.

## Vérifications

| Test | Commande / action | Résultat attendu |
|---|---|---|
| VLAN et trunk | `show vlan brief`, `show interfaces trunk` sur SW1 | VLAN 10-99 présents, trunk actif, natif 999 |
| DHCP | `ipconfig` sur PC-ADMIN | Adresse en 192.168.10.x |
| OSPF | `show ip ospf neighbor` sur R1 et R2 | Voisin en état FULL |
| Route par défaut | `show ip route` sur R2-CORE | Route `O*E2 0.0.0.0/0` via 10.0.0.1 |
| Inter-VLAN | ping PC-ADMIN → SRV-01 | Succès |
| Isolation invités | ping PC-INVITE → PC-RH | Échec (bloqué par `INVITES-IN`) |
| Internet | ping PC-INVITE → 198.51.100.1 | Succès |
| NAT | `show ip nat translations` sur R1 | Traductions vers 203.0.113.2 |
| SSH | `ssh -l admin 192.168.99.2` depuis PC-ADMIN | Connexion acceptée ; refusée depuis PC-INVITE |

## Structure du dépôt

```
lab-reseau-entreprise/
├── README.md
└── configs/
    ├── ISP.txt
    ├── R1-EDGE.txt
    ├── R2-CORE.txt
    └── SW1.txt
```

## Feuille de route

- [ ] Ajouter le fichier Packet Tracer `lab.pkt`
- [ ] Ajouter des captures des vérifications (`docs/`)
- [ ] Redondance de passerelle avec HSRP et un second commutateur
- [ ] Double pile IPv6
- [ ] Supervision SNMP du labo (voir [supervision-zabbix](https://github.com/zouloupapers/supervision-zabbix))

## Auteur

**Bien De Grace KABITOU ONDONGO** — étudiant en 3ᵉ année de licence Ingénierie des réseaux et systèmes, Université Centrale (Tunis) · CCNA1 & CCNA2.
Je recherche un **stage de fin d'études du 1ᵉʳ février au 31 mars 2027**.
