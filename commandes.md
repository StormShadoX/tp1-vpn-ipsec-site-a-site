# TP 1 : commandes (VPN IPsec site à site)

Noms d'interfaces du c7200 utilisés : `FastEthernet0/0`, `Serial1/0`, `Serial1/1`. Vérifie les tiens avec `show ip interface brief` et adapte-les, surtout dans `crypto map`.

## 1. PC (VPCS)

```
PC-A> ip 192.168.1.3/24 192.168.1.1
PC-A> save

PC-C> ip 192.168.3.3/24 192.168.3.1
PC-C> save
```

## 2. Adressage et routage statique

### R1
```
enable
configure terminal
hostname R1
interface FastEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit
interface Serial1/0
 ip address 10.1.1.2 255.255.255.252
 no shutdown
 exit
ip route 192.168.3.0 255.255.255.0 10.1.1.1
ip route 10.2.2.0 255.255.255.252 10.1.1.1
end
write memory
```

### R2
```
enable
configure terminal
hostname R2
interface Serial1/0
 ip address 10.1.1.1 255.255.255.252
 no shutdown
 exit
interface Serial1/1
 ip address 10.2.2.1 255.255.255.252
 no shutdown
 exit
ip route 192.168.1.0 255.255.255.0 10.1.1.2
ip route 192.168.3.0 255.255.255.0 10.2.2.2
end
write memory
```

### R3
```
enable
configure terminal
hostname R3
interface FastEthernet0/0
 ip address 192.168.3.1 255.255.255.0
 no shutdown
 exit
interface Serial1/1
 ip address 10.2.2.2 255.255.255.252
 no shutdown
 exit
ip route 192.168.1.0 255.255.255.0 10.2.2.1
ip route 10.1.1.0 255.255.255.252 10.2.2.1
end
write memory
```

### Tests avant IPsec
```
PC-A> ping 192.168.3.3
R1# show ip route
R1# ping 10.2.2.2
R3# ping 10.1.1.2
```
Les routes vers `10.2.2.0/30` (R1) et `10.1.1.0/30` (R3) sont indispensables : les pairs IPsec négocient directement entre 10.1.1.2 et 10.2.2.2.

## 3. IPsec sur R1

```
configure terminal
crypto isakmp enable
access-list 110 permit ip 192.168.1.0 0.0.0.255 192.168.3.0 0.0.0.255

crypto isakmp policy 10
 encryption aes
 authentication pre-share
 group 2
 exit
crypto isakmp key cisco address 10.2.2.2

crypto ipsec transform-set VPN-SET esp-3des esp-sha-hmac
crypto map VPN-MAP 10 ipsec-isakmp
 description VPN connection to R3
 set peer 10.2.2.2
 set transform-set VPN-SET
 match address 110
 exit

interface Serial1/0
 crypto map VPN-MAP
 end
write memory
```

## 4. IPsec sur R3 (symétrique)

```
configure terminal
crypto isakmp enable
access-list 110 permit ip 192.168.3.0 0.0.0.255 192.168.1.0 0.0.0.255

crypto isakmp policy 10
 encryption aes
 authentication pre-share
 group 2
 exit
crypto isakmp key cisco address 10.1.1.2

crypto ipsec transform-set VPN-SET esp-3des esp-sha-hmac
crypto map VPN-MAP 10 ipsec-isakmp
 description VPN connection to R1
 set peer 10.1.1.2
 set transform-set VPN-SET
 match address 110
 exit

interface Serial1/1
 crypto map VPN-MAP
 end
write memory
```

Sur certains IOS, `crypto isakmp enable` est déjà actif par défaut ou renvoie une erreur sans gravité. Vérifie avec `show crypto isakmp policy`.

## 5. Vérification

Avant le trafic (compteurs à 0) :
```
R1# show crypto isakmp policy
R1# show crypto ipsec sa
```

Génération du trafic, avec une capture Wireshark sur le lien R1–R2 démarrée avant le ping :
```
PC-A> ping 192.168.3.3
```
Filtres Wireshark : `isakmp`, `esp`, `icmp`.

Après le trafic (compteurs supérieurs à 0) :
```
R1# show crypto isakmp sa
R1# show crypto ipsec sa
```
État attendu : `QM_IDLE` entre 10.1.1.2 et 10.2.2.2.

Pour revoir la négociation ISAKMP, efface d'abord les SA :
```
R1# clear crypto isakmp
R1# clear crypto sa
R3# clear crypto isakmp
R3# clear crypto sa
```

## Dépannage

| Symptôme | Piste |
|---|---|
| Pas de `QM_IDLE` | clé, adresse du pair ou stratégie ISAKMP différente. `debug crypto isakmp` pendant un ping, puis `undebug all` |
| Compteurs à 0 | le trafic ne correspond pas à l'ACL 110, ou la crypto map n'est pas sur la bonne interface. Le ping doit venir de PC-A |
| ACL non symétriques | l'ACL 110 de R3 doit être exactement inversée par rapport à celle de R1 |
| Ping LAN à LAN OK mais IKE échoue | routes manquantes vers 10.2.2.0/30 (R1) ou 10.1.1.0/30 (R3) |
