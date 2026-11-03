# 03 — Diagnostic : chaque problème, sa cause racine, son fix

Huit problèmes réels, dans l'ordre chronologique. Chacun : **symptôme → cause →
correctif → leçon**. C'est le cœur du retour d'expérience du projet.

## 1. Syntaxe iptables refusée dans `cluster.fw`

- **Symptôme** : `pve-firewall status` → « unable to parse rule parameters » sur
  toutes les règles écrites en syntaxe iptables (`-s`, `-d`, `--dport`,
  `-m conntrack`...).
- **Cause** : le parser de PVE n'accepte pas la syntaxe brute ; il attend
  `-source`, `-dest`, `-p`, `-dport`, `-log`. La règle conntrack ESTABLISHED
  est injectée automatiquement, inutile de l'écrire.
- **Fix** : réécriture des règles dans la syntaxe PVE (via GUI ou `pvesh`).
- **Leçon** : ne jamais écrire des règles iptables dans les fichiers `.fw` —
  GUI ou API, point final.

## 2. Pas de routage : `ip_forward = 0`

- **Symptôme** : les VMs n'avaient pas d'Internet même avec SNAT.
- **Cause** : `net.ipv4.ip_forward = 0` par défaut.
- **Fix** : `sysctl -w net.ipv4.ip_forward=1` + persistance via
  `/etc/sysctl.d/99-ip-forward.conf`.
- **Leçon** : sans ça, rien ne traverse le nœud, et aucun message d'erreur ne
  le dit.

## 3. L'IPSet `management` ouvrait tout le LAN

- **Symptôme** : l'ipset auto-généré `PVEFW-0-management-v4` contenait
  `192.168.1.0/24` + l'IP admin → les ports 22, 8006, 5900-5999, 3128,
  60000-60050 étaient ouverts à **tout le LAN** sans qu'aucune règle ne le dise.
- **Cause** : PVE remplit automatiquement cet ipset avec les réseaux détectés.
- **Fix** : alias `local_network`/`cluster_network` restreints à l'IP du nœud
  en `/32`, puis suppression des usages de l'ipset : IP en dur dans les règles.
- **Leçon** : c'est LA backdoor silencieuse d'une install Proxmox. À auditer
  systématiquement (voir `04-audit-securite.md`).

## 4. Le backend iptables ignorait les règles FORWARD

- **Symptôme** : les règles de blocage inter-zones semblaient enregistrées mais
  n'étaient compilées nulle part.
- **Cause** : le pare-feu iptables classique de PVE n'applique pas les règles
  FORWARD — confirmé par la doc officielle PVE 9.2.
- **Fix** : bascule vers le backend **nftables** : `pvesh set
  /nodes/$NODE/firewall/options -enable 1 -nftables 1`, `iptables-nft` via
  `update-alternatives`, service `proxmox-firewall`.
- **Leçon** : décision d'architecture majeure. Toute isolation inter-VMs sous
  PVE 9 passe par le backend nftables.
- **Piège annexe** : `pvesh` refusait le nom d'hôte `pve` (lookup failed) —
  toujours `NODE=$(hostname -s)`.

## 5. Les règles FORWARD étaient **sautées** à cause du champ interface

- **Symptôme** : log `skipping rule ... dir: Forward ... cannot define
  interfaces for forward direction` ; la chaîne `cluster-forward` ne contenait
  que le `drop` final, les 8 règles étaient ignorées.
- **Cause** : le backend nftables refuse les interfaces sur les règles forward.
- **Fix** : retirer tout `-i` des règles FORWARD de `cluster.fw` → les 6 DROP
  + 2 ACCEPT se compilent, l'isolation devient réelle.
- **Leçon** : jamais d'interface sur une règle forward ; et **regarder les logs
  du service** (`journalctl -u proxmox-firewall`), pas seulement la GUI.

## 6. `policy_forward` hérité à ACCEPT

- **Symptôme** : un trace montrait `policy_forward: ACCEPT` alors que la GUI
  affichait DROP... hérité d'un état précédent.
- **Fix** : re-défini explicitement à DROP dans Datacenter → Pare-feu → Options,
  vérifié via `pvesh get /cluster/firewall/options`.
- **Leçon** : vérifier les options par API après chaque reconfiguration.

## 7. Le DNS du nœud bloquait `apt`

- **Symptôme** : `Temporary failure resolving 'deb.debian.org'` — plus aucune
  mise à jour possible.
- **Cause** : le DNS du nœud pointait sur `192.168.1.53` (DNS interne du LAN),
  bloqué par la règle `OUT DROP vers 192.168.1.0/24`.
- **Fix retenu** : garder le DNS interne + 2 règles `OUT ACCEPT` (udp+tcp 53)
  vers `192.168.1.53/32` **avant** le DROP.
- **Leçon** : quand on coupe la sortie vers le LAN, penser à tout ce que l'hôte
  contacte sur ce LAN : DNS, NTP, dépôts miroirs.

## 8. Le DHCP bloqué (2 jours de diagnostic) — le grand final

- **Symptôme** : la VM Windows reste en APIPA (`169.254.x.x`), « Impossible de
  contacter votre serveur DHCP ». Pourtant tout semble bon : IPAM ✔, ethers ✔,
  dnsmasq à l'écoute ✔, règle DHCP cochée ✔.
- **Méthode qui a débloqué** : tcpdump synchronisé sur **toutes** les
  interfaces (`timeout 60 tcpdump -ni any -e udp port 67 or udp port 68`) +
  `journalctl -u dnsmasq@zone1 -f` + `ipconfig /renew` pendant la capture.
  → les DISCOVER **atteignent le bridge** (10 paquets capturés) mais le journal
  dnsmasq reste muet : le paquet meurt dans netfilter.
- **Cause racine (double)** :
  1. la règle DHCP compilée en nft exigeait `ip daddr 10.10.1.254` — or un
     DISCOVER part vers `255.255.255.255` : jamais de match, DROP de la policy ;
  2. piège UI : le champ « Source » de l'éditeur graphique se compilait en
     filtre **`daddr`** sur cette règle.
- **Fix** : règle `in ACCEPT udp sport 68 dport 67` sur le VNet avec **Source
  et Destination VIDES**. Compilé : `iifname "LANlab" udp sport 68 udp dport 67
  accept` → `DHCPOFFER` + `DHCPACK` en quelques secondes.
- **Leçon (la plus rentable du projet)** : une règle DHCP doit matcher le
  broadcast ; et **la vérité est dans le nft compilé**, jamais dans la GUI.
  Reproduire le fix sur chaque nouvelle zone.

## Matrice de validation finale

| Depuis | Test | Attendu | Résultat |
|---|---|---|---|
| Poste admin `.10` | `https://192.168.1.100:8006`, SSH | ✅ | ✅ |
| Autre appareil LAN (téléphone) | ping/8006 vers `.100` | ❌ + DROP loggé | ✅ |
| VM Lab | `ping 8.8.8.8` / `nslookup` | ✅ (SNAT/DNS) | ✅ |
| VM Lab | ping VM Games / LAN | ❌ | ✅ (100 % perte) |
| Nœud | `ping` LAN / `apt update` | ❌ / ✅ | ✅ |
| — | `reboot` du nœud, tout re-tester | tout persiste | ✅ |
