# TP 1 : VPN IPsec site à site sous GNS3

Travail pratique de réseau (RSI3) : tunnel VPN IPsec entre deux routeurs d'extrémité (R1 et R3) à travers un routeur intermédiaire (R2) qui simule Internet. Réalisé sous **GNS3** avec des routeurs Cisco **c7200**, des PC **VPCS** et **Wireshark**.

Les commandes complètes sont dans [`commandes.md`](commandes.md).

## Contenu du dépôt

```
.
├── README.md
├── commandes.md
├── .gitignore
├── projet/       # fichier .gns3 ou .gns3project
├── captures/     # fichiers .pcap Wireshark
└── images/       # captures d'écran du rapport
```

## Prérequis

- GNS3 (client Windows) et la VM GNS3 sous VirtualBox
- Image IOS Cisco **c7200** avec support de la commande `crypto` (non fournie dans ce dépôt)
- Wireshark

## Objectif

Chiffrer le trafic entre le LAN de PC-A (192.168.1.0/24) et celui de PC-C (192.168.3.0/24).

## Table d'adressage

| Équipement | Interface | Adresse IP | Masque |
|---|---|---|---|
| R1 | FastEthernet0/0 | 192.168.1.1 | 255.255.255.0 |
| R1 | Serial1/0 | 10.1.1.2 | 255.255.255.252 |
| R2 | Serial1/0 | 10.1.1.1 | 255.255.255.252 |
| R2 | Serial1/1 | 10.2.2.1 | 255.255.255.252 |
| R3 | FastEthernet0/0 | 192.168.3.1 | 255.255.255.0 |
| R3 | Serial1/1 | 10.2.2.2 | 255.255.255.252 |
| PC-A | NIC | 192.168.1.3 | 255.255.255.0 (passerelle 192.168.1.1) |
| PC-C | NIC | 192.168.3.3 | 255.255.255.0 (passerelle 192.168.3.1) |

## Paramètres IPsec

| Élément | Valeur |
|---|---|
| ISAKMP (phase 1) | AES, SHA-1 (défaut), pre-share, Diffie-Hellman groupe 2, durée de vie 86400 s |
| IPsec (phase 2) | transform-set `VPN-SET` : `esp-3des esp-sha-hmac` |
| Trafic chiffré | ACL 110 : `permit ip` entre 192.168.1.0/24 et 192.168.3.0/24 (inversée sur R3) |
| Crypto map | `VPN-MAP` (ipsec-isakmp), appliquée sur l'interface série de sortie |
| Pairs | R1 = 10.1.1.2, R3 = 10.2.2.2 |

## Point d'attention

En plus du routage vers les LAN, R1 et R3 ont besoin d'une route vers le réseau `/30` situé de l'autre côté (10.2.2.0/30 pour R1, 10.1.1.0/30 pour R3). Les pairs IPsec négocient directement entre 10.1.1.2 et 10.2.2.2 : sans ces routes, IKE ne s'établit pas, même si PC-A joint PC-C.

## Vérifications attendues

- Avant le trafic : `show crypto ipsec sa` avec tous les compteurs (`pkts encaps`, `encrypt`, `decaps`, `decrypt`) à 0.
- Après `ping 192.168.3.3` depuis PC-A : compteurs supérieurs à 0 et `show crypto isakmp sa` à l'état `QM_IDLE`.
- Wireshark (lien R1–R2) : filtre `isakmp` (Main Mode puis Quick Mode), filtre `esp` (protocole IP 50), filtre `icmp` vide.

## Encapsulation observée (mode tunnel)

| Couche | Contenu |
|---|---|
| Liaison | trame du lien série |
| Nouvel en-tête IP | 10.1.1.2 → 10.2.2.2, protocole 50 (ESP) |
| ESP | SPI et numéro de séquence |
| Données chiffrées (3DES) | IP 192.168.1.3 → 192.168.3.3 + ICMP |
| Trailer ESP + ICV | padding et authentification SHA-1-HMAC |

## Ouvrir le projet

1. Démarre la VM GNS3 sous VirtualBox, puis GNS3 sous Windows.
2. *File → Open project* (ou *Import portable project* pour un `.gns3project`).
3. Associe tes propres images c7200 aux routeurs.
4. Démarre les nœuds et colle les commandes de `commandes.md`.

## Sécurité du dépôt

- La clé pré-partagée `cisco` est une clé de **laboratoire**. Ne la réutilise jamais en production.
- Les images IOS Cisco ne sont pas publiées (licence).
