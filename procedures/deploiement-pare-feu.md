# Déployer le pare-feu sur Proxmox VE (9.x)

Procédure générique, applicable à toute installation Proxmox VE 9.x (backend
nftables). Elle ne présuppose aucun cahier des charges : les valeurs d'exemple
sont à remplacer par les vôtres. Chaque « ⚠️ Piège » signale un écueil réel
rencontré en production — voir `../archive/conversation.md` et `../docs/03-diagnostic.md` section 8.

Conventions d'exemple utilisées ci-dessous :
- LAN physique / box : `192.168.1.0/24`, box `192.168.1.1`
- Poste admin : `192.168.1.10` (réservation DHCP recommandée)
- Nœud Proxmox : `192.168.1.100` (vmbr0, interface physique `enp1s0`)

---

## Étape 1 — Socle réseau du nœud

```bash
# Confirmer l'interface et le nom court du nœud (à réutiliser partout)
ip -br link
NODE=$(hostname -s); echo $NODE
```

`/etc/network/interfaces` (au minimum) :
```
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.100/24
    gateway 192.168.1.1
    bridge-ports enp1s0
    bridge-stp off
    bridge-fd 0
```

⚠️ **Piège (vécu)** : le nom DNS du nœud n'est pas toujours celui qu'on croit
(`pve` ≠ `proxmox`). Toujours utiliser `$(hostname -s)` dans les commandes
`pvesh set /nodes/$NODE/...`.

## Étape 2 — Activer le pare-feu avec le backend nftables

```bash
pvesh set /nodes/$(hostname -s)/firewall/options -enable 1 -nftables 1
pvesh set /cluster/firewall/options -enable 1 -ebtables 1
```

⚠️ **Piège (vécu, décision d'architecture)** : le backend **iptables classique
ignore les règles FORWARD** — seules celles du nœud sont appliquées. Si on veut
filtrer le trafic entre VMs/zones, le backend **nftables** (`proxmox-firewall`)
est indispensable.

Vérification :
```bash
systemctl status proxmox-firewall     # active (running)
nft list tables                        # doit contenir "table inet proxmox-firewall"
```

⚠️ **Piège (vécu)** : après un `restart`, la table n'existe que quelques
secondes plus tard — un `nft list chain ...` juste après peut renvoyer
« No such file or directory » sans que ce soit une erreur. Attendre ~5 s.

⚠️ **Piège (vécu)** : le backend nftables n'utilise **plus les chaînes
`PVEFW-*`**. `nft list ruleset | grep PVEFW` ne trouvant rien est NORMAL.
Lire avec `nft list table inet proxmox-firewall`.

## Étape 3 — Politiques du datacenter

```
Datacenter → Pare-feu → Options :
  Pare-feu activé, ebtables activé
  Politique entrée  : DROP
  Politique sortie  : ACCEPT
  Politique forward : DROP
```

⚠️ **Piège (vécu)** : `policy_forward` peut être hérité d'une valeur ACCEPT
précédente — vérifier explicitement après chaque changement majeur :
```bash
pvesh get /cluster/firewall/options
```

## Étape 4 — Règles de gestion de l'hôte (IN/OUT)

Principe : n'ouvrir **que** le nécessaire, avec des adresses explicites.

IN (via l'UI Datacenter → Pare-feu → Règles) :
| # | Direction | Action | Interface | Proto | D.Port | Source | Destination |
|---|---|---|---|---|---|---|---|
| 1 | in | ACCEPT | vmbr0 | macro SSH | — | 192.168.1.10/32 | 192.168.1.100/32 |
| 2 | in | ACCEPT | vmbr0 | tcp | 8006 | 192.168.1.10/32 | 192.168.1.100/32 |
| 3 | in | DROP | vmbr0 | — | — | 192.168.1.0/24 | 192.168.1.100/32 |

⚠️ **Piège (vécu)** : l'IPSet `management` est **auto-généré** par PVE et ouvre
22/8006/5900-5999/3128/60000-60050 à tout ce qu'il contient. S'il récupère un
jour `192.168.1.0/24`, tout le LAN accède à l'hôte. Après installation :
```
Datacenter → Pare-feu → IPSet : vérifier que "management" (s'il existe)
ne contient QUE l'IP admin et/ou l'IP du nœud — sinon le vider.
```
Idem pour les alias : `local_network`/`cluster_network` doivent pointer sur
l'IP **du nœud seul** (`/32`), pas sur le subnet.

OUT (si le nœud doit contacter le LAN malgré la policy) :
- des ACCEPT ciblés **avant** le DROP général (ex. DNS interne, NTP box),
- puis `OUT DROP -dest 192.168.1.0/24`.

⚠️ **Piège (vécu)** : si le DNS du nœud pointe sur une IP du LAN non autorisée
en OUT, `apt update` échoue avec « Temporary failure resolving ». Ouvrir
explicitement `OUT ACCEPT udp/tcp 53` vers ce DNS **avant** le DROP.

## Étape 5 — Zones SDN pour les VMs (optionnel mais recommandé)

Objectif : donner aux VMs un réseau IP privé, du DHCP/DNS, et un accès Internet
via SNAT — tout en filtrant les flux entre zones.

```
Datacenter → SDN → Zones : créer une zone Simple par groupe logique
Datacenter → SDN → VNets : un VNet par zone
  Sous-réseau : 10.X.Y.0/24, passerelle .254, DHCP range .225-.253, SNAT coché
Zone → DHCP : dnsmasq
```

```bash
apt install -y dnsmasq
systemctl disable --now dnsmasq                 # ⚠️ le SDN lance des instances PAR ZONE
pvesh set /cluster/sdn/zones/<zone> -dhcp dnsmasq -ipam pve
pvesh set /cluster/sdn                           # apply
pvesh set /cluster/sdn --apply 1
```

Vérifications :
```bash
systemctl is-active dnsmasq@<zone>
ss -lunp | grep :67                              # écoute sur %<vnet>:67
ls /etc/dnsmasq.d/<zone>/                        # conf + ethers générés
```

⚠️ **Piège (vécu)** : `dnsmasq.service` générique doit rester **désactivé** —
seules les instances `dnsmasq@<zone>` servent le DHCP.

⚠️ **Piège (vécu)** : avec l'IPAM PVE, dnsmasq démarre en « static leases only »
(`dhcp-ignore=tag:!known`) : seules les VMs **enregistrées dans l'IPAM**
obtiennent un bail. Pour chaque VM : créer son entrée (IPAM : MAC, IP, VMID)
puis re-Apply le SDN et `systemctl restart dnsmasq@<zone>`.

⚠️ **Piège (vécu, 2 jours de diagnostic)** : la règle DHCP générée/éditée se
compile avec un filtre `ip daddr <passerelle ou subnet>` alors qu'un DISCOVER
part vers `255.255.255.255` — **elle ne matche jamais** et le bail n'arrive pas.
Correctif éprouvé : règle DHCP avec **Source VIDE et Destination VIDE** :
```
Direction: in  Action: ACCEPT  Interface: <vnet>
Protocol: udp  S.Port: 68  D.Port: 67  Source: —  Destination: —
```
Compilé attendu : `iifname "<vnet>" udp sport 68 udp dport 67 accept`
(**sans** `daddr`/`saddr`). Ajouter aussi l'équivalent DNS si les VMs doivent
résoudre via la passerelle : `in ACCEPT udp dport 53 -dest <passerelle>/32`.

⚠️ **Piège (vécu, UI)** : sur ce backend, le champ « Source » de certaines
règles éditées en GUI s'est compilé en filtre de **destination** — si une règle
n'applique pas ce qu'elle semble dire, **toujours vérifier le nft compilé**,
pas la GUI.

## Étape 6 — Routage et FORWARD

```bash
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/99-ip-forward.conf
```

⚠️ **Piège (vécu)** : le backend nftables **refuse les interfaces sur les
règles forward** (`cannot define interfaces for forward direction`) — une
règle FORWARD avec `-i` est **silencieusement sautée**. Ne jamais mettre
d'interface sur les règles forward.

Règles FORWARD génériques (Datacenter → Pare-feu → Règles, Direction forward) —
**l'ordre est capital** :
| # | Action | Source | Destination | Rôle |
|---|---|---|---|---|
| 1..n | DROP | zone A | zone B (et inversement) | interdire inter-zones |
| n+1.. | DROP | zones | LAN | interdire VMs → LAN |
| n+2.. | DROP | LAN | zones | interdire LAN → VMs |
| fin | ACCEPT | zone A | — | zone A → Internet (via SNAT) |
| fin | ACCEPT | zone B | — | zone B → Internet |

Vérification du compilé :
```bash
nft list chain inet proxmox-firewall cluster-forward
# les DROP puis les ACCEPT, un "drop" final — sinon l'ordre est faux
```

## Étape 7 — Matrice de validation

À exécuter après déploiement, dans cet ordre :

1. **Gestion** : depuis le poste admin → `https://<nœud>:8006` et SSH ✅ ;
   depuis un autre appareil du LAN → 8006/SSH ❌ **et** une ligne `DROP` visible
   dans Datacenter → Pare-feu → Log (règles DROP en niveau `warning`).
2. **Hôte** : `apt update` ✅ depuis le nœud ; ping du nœud vers le LAN ❌
   (sauf exceptions explicites).
3. **DHCP d'une VM** : `ipconfig /renew` (ou dhclient) → bail obtenu ;
   `journalctl -u dnsmasq@<zone>` montre DISCOVER → OFFER → REQUEST → ACK.
   En cas d'échec : tcpdump synchronisé (`timeout 40 tcpdump -ni any -e udp
   port 67 or udp port 68`) + renew pendant la capture — la position du dernier
   paquet vu désigne la couche fautive (VM / bridge / netfilter / dnsmasq).
4. **VM → Internet** : ping 8.8.8.8 ✅ (SNAT) ; résolution DNS ✅.
5. **Isolation** : VM zone A → VM zone B ❌ ; VM → LAN ❌ ; LAN → VM ❌.
6. **Persistance** : `reboot` le nœud, refaire 1–5. Tout doit tenir sans
   intervention (ip_forward, proxmox-firewall, dnsmasq@zones, SDN).

## Étape 8 — Audit final (anti-backdoor)

Checklist rapide à refaire périodiquement :
- IPSet : `management` absent ou restreint à l'admin/nœud uniquement
- Alias : uniquement des `/32` voulus
- Security Groups : vide ou maîtrisé
- Nœud → Pare-feu → Règles : vide (tout au datacenter)
- Log : aucun ACCEPT depuis une IP inattendue
- `nft list table inet proxmox-firewall` relu intégralement après tout changement
  (la GUI peut compiler différemment de ce qu'elle affiche)

## Niveaux de log recommandés

- Règles : `warning` (limité par le log rate limit, 1/s burst 5) — bon défaut.
- `debug` : journalise chaque paquet **sans limite de débit** — uniquement
  pour un diagnostic ponctuel, puis revenir à `warning`.
