<p align="center">
  <img src="https://raw.githubusercontent.com/proxmox/pve-docs/master/images/proxmox-logo.svg" alt="Proxmox VE" width="360">
</p>

<h1 align="center"> Pare-feu Proxmox nftables</h1>

<p align="center">
  <b>Isolation complète d'un serveur Proxmox VE 9 : pare-feu nftables, zones SDN étanches, Internet pour les VMs.</b><br>
  <i>Projet réel, mené de bout en bout — à la validation après redémarrage.</i>
</p>

---

## Ce que j'ai fait

Parti d'un hyperviseur Proxmox VE 9.2 installé « nu » sur le réseau local de la box, j'ai construit une infrastructure où :

- **l'hyperviseur est invisible du LAN** : plus personne ne peut ouvrir l'interface web (8006) ou SSH (22), à l'exception d'un unique poste admin (IP fixe) — les tentatives sont bloquées **et journalisées** ;
- **deux zones de VMs coexistent sans jamais se voir** : un « Lab » (`10.10.1.0/24`) pour les tests d'applications et une zone « Games » (`10.20.1.0/24`), étanches l'une à l'autre **et** au réseau local ;
- **toutes les VMs ont Internet** via SNAT, avec DHCP et DNS servis par zone (dnsmasq + IPAM), sans jamais exposer le LAN ;
- **tout survit à un redémarrage** : sysctl, services, SDN — rien à re-saisir.

Le parcours n'a pas été linéaire : **8 problèmes réels** ont été diagnostiqués et résolus, dont deux qui valent le détour —
un IPSet `management` auto-généré qui rouvrait silencieusement les ports d'admin à tout le LAN, et un DHCP bloqué pendant
deux jours parce que la règle pare-feu filtrait sur une adresse que le broadcast DHCP n'utilise jamais. Chacun est
documenté en **symptôme → cause racine → correctif → leçon** dans [`docs/03-diagnostic.md`](docs/03-diagnostic.md).

L'ensemble a été terminé par un **audit anti-backdoor** complet ([`docs/04-audit-securite.md`](docs/04-audit-securite.md)) :
IPSet, alias, groupes de sécurité, règles de nœud, lecture des logs — verdict : aucun vecteur.

| Exigence initiale | Résultat |
|---|---|
| Proxmox inaccessible du LAN (sauf poste admin `.10`) | ✅ validé — DROP loggés pour tout le reste |
| Zones Lab / Games étanches entre elles | ✅ validé — 100 % perte inter-zones |
| Internet pour toutes les VMs (SNAT + DHCP + DNS) | ✅ validé — ping 8.8.8.8 et résolution OK |
| Persistance après reboot | ✅ validé — re-test complet sans intervention |

## L'architecture obtenue

<img width="375.9375" height="457.5" alt="Architecture" src="https://github.com/user-attachments/assets/e99aeda6-b6a8-4cd1-8797-08f61b808324" />

## Comment y parvenir

La procédure complète, pas à pas et réutilisable, est dans
**[`procedures/deploiement-pare-feu.md`](procedures/deploiement-pare-feu.md)**. Le chemin en résumé :

1. **Socle réseau** — un pont `vmbr0` vers la box, l'IP du nœud dessus. *(§1)*
2. **Backend nftables** — décision d'architecture clé : l'ancien backend iptables **ignore les règles FORWARD** ;
   sans nftables, aucune isolation inter-VMs n'est possible. *(§2)*
3. **Politiques strictes** — entrée DROP, forward DROP : tout interdit, puis on ouvre au cas par cas. *(§3)*
4. **Accès admin** — ACCEPT SSH/8006 uniquement depuis l'IP du poste admin, DROP du LAN derrière, et surtout :
   vérifier l'**IPSet `management`** auto-généré (la backdoor classique d'une install Proxmox). *(§4)*
5. **Zones SDN** — un VNet par groupe de VMs, SNAT coché, dnsmasq par zone, IPAM pour des IP fixes, et la règle
   DHCP avec **Source et Destination vides** (le broadcast `255.255.255.255` ne matche rien d'autre). *(§5)*
6. **Filtrage FORWARD** — `ip_forward=1` persistant, puis les DROP inter-zones/LAN **avant** les ACCEPT vers
   Internet, et **jamais d'interface** sur ces règles (sinon elles sont silencieusement sautées). *(§6)*
7. **Validation** — la matrice de tests complète, poste admin / téléphone / VM / reboot. *(§7)*
8. **Audit** — la checklist anti-backdoor à rejouer périodiquement. *(§8)*

Chaque « ⚠️ Piège » de la procédure correspond à un problème réellement rencontré — le détail complet des
diagnostics est dans [`docs/03-diagnostic.md`](docs/03-diagnostic.md).

## Organisation du dépôt

```
docs/
  01-objectif.md          — le cahier des charges initial
  02-architecture.md      — l'architecture retenue et pourquoi
  03-diagnostic.md       — les 8 problèmes : symptôme → cause → fix → leçon
  04-audit-securite.md    — l'audit anti-backdoor final
  05-retour-experience.md — les 7 leçons transversales + boîte à outils
procedures/
  deploiement-pare-feu.md — LA procédure réutilisable, pas à pas
  test-dhcp.md            — déboguer un DHCP récalcitrant (méthode universelle)
```

## Lire ce dépôt selon votre besoin

| Vous voulez… | Allez à |
|---|---|
| Reproduire cette configuration | [`procedures/deploiement-pare-feu.md`](procedures/deploiement-pare-feu.md) |
| Comprendre pourquoi ça a bloqué | [`docs/03-diagnostic.md`](docs/03-diagnostic.md) |
| Voir la carte finale et les choix | [`docs/02-architecture.md`](docs/02-architecture.md) |
| Auditer votre propre Proxmox | [`docs/04-audit-securite.md`](docs/04-audit-securite.md) |
| Éviter les mêmes erreurs | [`docs/05-retour-experience.md`](docs/05-retour-experience.md) |

---

*Adresses et identifiants du dépôt anonymisés (valeurs d'exemple `192.168.1.x`).*
*Proxmox® est une marque déposée de Proxmox Server Solutions GmbH — ce projet personnel n'est pas affilié à Proxmox.*
