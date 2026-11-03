# 05 — Retour d'expérience : Proxmox VE 9 + pare-feu nftables

Leçons transversales tirées du projet, utiles pour toute installation similaire.

## Les 7 leçons à retenir

1. **Le backend compte plus que les règles.** Sous PVE 9, sans le backend
   nftables, les règles FORWARD n'existent pas — aucune isolation inter-VMs
   n'est possible. C'est la première décision à prendre, pas un détail.
2. **La GUI n'est pas la vérité ; le nft compilé l'est.** Champ « Source »
   compilé en `daddr`, règles silencieusement sautées, IPSet régénéré : après
   chaque changement, `nft list table inet proxmox-firewall` se lit intégralement.
3. **Jamais d'interface sur une règle forward.** Le backend la saute sans lever
   d'erreur bloquante — juste une ligne de log.
4. **Une règle DHCP doit matcher le broadcast.** Source et Destination vides,
   restriction par interface + ports uniquement.
5. **Vérifier les ouverture auto-générées** (IPSet `management`, alias) : la
   backdoor d'une install Proxmox ne vient pas des règles qu'on écrit, mais de
   celles qu'on n'écrit pas.
6. **Diagnostiquer par couches, avec une capture synchronisée.** tcpdump sur
   `-i any` + `journalctl -f` + l'action pendant la fenêtre : la position du
   dernier paquet vu désigne la couche fautive (émission / L2 / netfilter /
   service). C'est la méthode qui a débloqué le DHCP en 40 secondes après deux
   jours d'errance.
7. **Le niveau de log `warning` est le bon équilibre.** Limité en débit
   (1/s burst 5), il trace les drops sans inonder. `debug` : ponctuel uniquement.

## Ce qui a bien marché

- **SDN + zones Simple** : parfaite réponse à une box sans VLAN. Chaque zone a
  son bridge, son gateway, son dnsmasq, sans toucher au LAN.
- **SNAT coché dans le subnet** : Internet pour les VMs dès la création, zéro
  configuration manuelle de NAT.
- **IPAM PVE + ethers** : les VMs ont des IP fixes prévisibles ; le mode
  « static leases only » évite les clients surprises.
- **La persistence** : tout (sysctl, services, SDN, règles) a survécu au reboot
  sans rien re-saisir.
- **L'ordre des règles** : ACCEPT ciblés avant DROP, DROP forward avant ACCEPT
  Internet — cette discipline a rendu la matrice de validation lisible et
  prévisible.

## Ce qu'on ferait différemment

- Activer le pare-feu **dès l'installation**, avant de créer des VMs : on
  aurait évité la période où l'hôte était joignable du LAN entier.
- Écrire la matrice de validation **avant** de commencer : elle aurait orienté
  les tests et évité des allers-retours.
- Activer le pare-feu au niveau VM (in=DROP / out=ACCEPT) pour la défense en
  profondeur — l'isolation repose aujourd'hui uniquement sur le datacenter.
- Prévoir d'emblée la correction DHCP sur **chaque** zone (le fix LANlab a été
  reproduit sur Games a posteriori).

## Outils de la boîte à diagnostic (à garder sous la main)

```bash
nft list table inet proxmox-firewall            # la vérité compilée
nft list chain inet proxmox-firewall cluster-in | grep 'dport 6[78]'   # DHCP
journalctl -u proxmox-firewall -f               # règles sautées, erreurs
journalctl -u dnsmasq@zone1 -f                  # DISCOVER/OFFER/ACK
ss -lunp | grep :67                             # dnsmasq à l'écoute ?
timeout 60 tcpdump -ni any -e udp port 67 or udp port 68   # capture synchronisée
pvesh get /cluster/firewall/options             # policies réelles
```
