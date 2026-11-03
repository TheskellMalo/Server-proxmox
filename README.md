# Server-proxmox — Isoler un hyperviseur Proxmox VE 9 avec pare-feu nftables et zones SDN

Projet réel de sécurisation d'un serveur Proxmox VE 9.2 : un hyperviseur installé sur
un réseau local domestique, rendu **inaccessible du LAN** (sauf depuis le poste admin),
avec des **zones de VM isolées** entre elles et **accès Internet** pour toutes.

> Réalisé sur Proxmox VE 9.2.25, backend pare-feu **nftables** (`proxmox-firewall`),
> SDN avec zones Simple + dnsmasq, IPAM PVE. Les adresses du dépôt sont des valeurs
> d'exemple anonymisées.

## L'objectif initial

1. Le serveur Proxmox ne doit pas communiquer avec le réseau local — sauf depuis le
   poste de l'administrateur (web 8006 / SSH 22).
2. Un premier groupe de VMs simule un LAN de test d'applications.
3. Un deuxième groupe de VMs ne doit **jamais** communiquer avec le premier.
4. Toutes les VMs doivent avoir **accès à Internet**.
5. L'ensemble doit **survivre à un redémarrage** sans intervention.

## Le résultat final

| Exigence | Statut |
|---|---|
| Accès web/SSH uniquement depuis le poste admin (`.10`) | ✅ validé (les autres appareils sont DROP-loggés) |
| VMs de la zone Lab (`10.10.1.0/24`) isolées de la zone Games (`10.20.1.0/24`) et du LAN | ✅ validé (100 % perte inter-zones et vers le LAN) |
| Internet pour toutes les VMs via SNAT | ✅ validé (ping 8.8.8.8, résolution DNS) |
| DHCP/DNS par zone (dnsmasq + IPAM PVE) | ✅ validé |
| Persistance après reboot (ip_forward, firewall, SDN, dnsmasq) | ✅ validé |
| Audit anti-backdoor (IPSet, alias, règles de nœud, logs) | ✅ aucun vecteur trouvé |

## Architecture déployée

```
                 ┌─────────────────────────────────────────────┐
                 │              BOX / PASSERELLE              │
                 │              192.168.1.1                   │
                 └───────┬─────────────────────────▲───────────┘
                         │ vmbr0 (192.168.1.100)   │ SNAT
                 ┌───────┴─────────────────────────┴───────────┐
                 │              NŒUD PROXMOX VE 9.2            │
                 │   pare-feu nftables (in DROP / fwd DROP)   │
                 └───────┬─────────────────────┬───────────────┘
                         │ VNet Lab            │ VNet Games
                  10.10.1.0/24           10.20.1.0/24
                  gw 10.10.1.254         gw 10.20.1.254
                  DHCP dnsmasq@zone1     DHCP dnsmasq@zone2
                         │                     │
                     VMs Lab  ◄── DROP ──►  VMs Games
                   (isolées entre elles et du LAN — Internet OK)
```

## Contenu du dépôt

```
docs/
  01-objectif.md          — le cahier des charges complet, reformulé
  02-architecture.md      — l'architecture retenue et pourquoi
  03-diagnostic.md        — LA SAGA : chaque problème → cause racine → fix → leçon
  04-audit-securite.md    — l'audit anti-backdoor final (verdict : clean)
  05-retour-experience.md — leçons transversales pour Proxmox VE 9 / nftables
procedures/
  deploiement-pare-feu.md — procédure générique réutilisable, pas à pas
  test-dhcp.md            — procédure de test DHCP synchronisée (méthode universelle)
archive/
  conversation.md         — résumé chronologique complet du projet
  transcript-brut.md      — transcript brut des échanges
```

## Pour commencer

- Vous voulez **reproduire cette configuration** ? → [`procedures/deploiement-pare-feu.md`](procedures/deploiement-pare-feu.md)
- Vous voulez comprendre **pourquoi ça a bloqué** ? → [`docs/03-diagnostic.md`](docs/03-diagnostic.md)
- Vous voulez juste **la carte finale** ? → [`docs/02-architecture.md`](docs/02-architecture.md)
