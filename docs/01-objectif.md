# 01 — L'objectif initial

Reformulation générique du cahier des charges de départ. Les adresses sont anonymisées.

## Contexte

Un serveur Proxmox VE est installé sur le réseau local d'un particulier
(`192.168.1.0/24`, box en `192.168.1.1`). Il héberge des VMs de test
d'applications et des VMs de jeu. Le réseau local n'est pas contrôlé
(autres appareils, Wi-Fi) : l'hyperviseur ne doit pas leur être accessible.

## Exigences

1. **Isolation de l'hyperviseur** — le serveur Proxmox ne doit pas communiquer
   avec le réseau local, sauf depuis le **poste de l'administrateur**
   (`192.168.1.10`, IP réservée) qui garde l'accès web (8006) et SSH (22).
2. **Groupe de VMs « Lab »** — un premier ensemble de VMs simulant un LAN de
   test d'applications, avec DHCP et DNS.
3. **Groupe de VMs « Games »** — un deuxième ensemble qui ne doit **jamais**
   communiquer avec le premier.
4. **Accès Internet** — toutes les VMs, des deux groupes, doivent pouvoir
   sortir sur Internet.
5. **Persistance** — tout doit tenir après un redémarrage, sans intervention
   manuelle.

## Contraintes découvertes en cours de route

- La box ne gère pas les VLAN 802.1Q → pas de zones VLAN, il faut des zones
  SDN de type **Simple**.
- Le nœud doit pouvoir se mettre à jour (`apt`) tout en étant coupé du LAN :
  le DNS et les dépôts doivent rester joignables.
- Les IP des VMs doivent être **prévisibles** (IPAM statique), pas aléatoires.

## Ce qui a été validé en fin de projet

Chaque exigence est couverte par un test documenté dans `docs/03-diagnostic.md`
(section « Matrice de validation ») et confirmée après redémarrage du nœud.
