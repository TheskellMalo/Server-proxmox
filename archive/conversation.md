# Compilation complète — Projet pare-feu Proxmox VE 9.2.25 (isolation LAN de test)

Date : 1–3 novembre 2026. Toutes les images sont dans `images/`, nommées `<message-id>_<nom-original>.png`.

---

## 1. Cahier des charges initial (fd4d2afe)

- Proxmox installé sur le réseau local d'un particulier.
- Le serveur Proxmox **ne doit pas communiquer avec le réseau local** (sauf depuis le PC de l'admin).
- Des VM vont **simuler le LAN** (environnement de test d'applications).
- Un **deuxième groupe de VM** ne doit **jamais** communiquer avec le premier.
- **Toutes les VMs doivent avoir accès à Internet.**
- Seul moyen d'accès à Proxmox : **interface web (8006) ou SSH (22)**, uniquement depuis le PC admin.
- Schéma fourni : `fd4d2afe..._Pc5CF5D5-image.png` (BOX en haut, « isolé du LAN », Mon PC → Proxmox « Accès WEB + SSH », BOX → Proxmox « accès internet »).

**Adressage identifié (ad232aa8)** :
- LAN de la box : `192.168.1.0/24`, box/passerelle : `192.168.1.1`, plage DHCP dynamique `.150–.254`.
- PC admin `poste-admin` (réservation DHCP statique, MAC `11:22:33:44:55:66`) : **`192.168.1.10`**.
- Proxmox : **`192.168.1.100`** (ensuite confirmé sur `vmbr0`, port `nic0`).
- Le PC est connecté en Ethernet **uniquement via la box** (pas de lien dédié).

## 2. Conception retenue

- `vmbr0` (ponté sur `nic0`) : `192.168.1.100/24`, gateway `192.168.1.1` — seul lien vers la box/Internet.
- **SDN** (zones `Simple`, pas VLAN car la box ne gère pas le 802.1Q) :
  - zone `zones1` → VNet **`LANlab`** (« home network labo ») → subnet **`10.10.1.0/24`**, gateway `10.10.1.254`, **SNAT coché**, DHCP range `10.10.1.225–10.10.1.253`.
  - zone `zones2` → VNet **`Glab`** (« VM games ») → subnet **`10.20.1.0/24`**, gateway `10.20.1.254`, **SNAT coché**, DHCP range `10.20.1.225–10.20.1.253`.
- Pare-feu datacenter : `policy_in DROP`, `policy_out ACCEPT`, `policy_forward DROP`.
- Règles prévues (IP en dur, **pas d'IPSet**) :
  - IN : ACCEPT tcp 22 et 8006 depuis `192.168.1.10/32` → `192.168.1.100/32` ; DROP `192.168.1.0/24` → `.103` (log warning).
  - OUT : ACCEPT DNS (udp+tcp 53) et NTP (udp 123) vers `192.168.1.1/32` ; ACCEPT DNS vers `192.168.1.53/32` (DNS interne de l'utilisateur, ajouté plus tard) ; DROP vers `192.168.1.0/24`.
  - FORWARD : 6 DROP (LANlab↔Glab, LANlab/Glab→LAN, vmbr0→LANlab/Glab) puis 2 ACCEPT (`10.10.1.0/24`, `10.20.1.0/24` → Internet via SNAT) ; policy DROP.
- Décision : **Option A = backend nftables (`proxmox-firewall`)** car c'est le seul qui applique les règles FORWARD (le pare-feu iptables classique les ignore — confirmé par la doc officielle PVE 9.2).

## 3. Problèmes rencontrés et résolutions (chronologie)

### 3.1 Syntaxe `cluster.fw` rejetée (02dc7968, c579a6b8)
- `pve-firewall status` : `unable to parse rule parameters` sur toutes les règles avec `-s`, `-d`, `--dport`, `-m conntrack`, `--log-prefix`.
- Cause : le parser PVE 9.2 n'accepte pas la syntaxe iptables brute. Syntaxe correcte (vue via `pvesh create`) : `-source`, `-dest`, `-p`, `-dport`, `-i`, `-o`, `-log warning`. La règle conntrack ESTABLISHED est injectée automatiquement.
- `pvesh set ... -in-interface/-out-interface` → `Unknown option` (400). Dans l'UI, les règles `forward` n'ont qu'un champ **Interface** (= in) ; l'"out" n'existe pas en GUI.

### 3.2 `ip_forward` (c579a6b8)
- `net.ipv4.ip_forward = 0` → corrigé : `sysctl -w net.ipv4.ip_forward=1` + `/etc/sysctl.d/99-ip-forward.conf` → persistant = 1.

### 3.3 IPSet `management` trop large (b96ec404, 91826c30)
- L'ipset auto `PVEFW-0-management-v4` contenait `192.168.1.0/24` + `.20` → ouvrait 22/8006/5900-5999/3128/60000-60050 à **tout le LAN**.
- Corrigé via alias : `[ALIASES] local_network 192.168.1.100/32` + `cluster_network 192.168.1.100/32` → le set ne contient plus que `.103` (et `.20` ajouté manuellement, puis décision finale : **supprimer l'IPSet management**, IP en dur dans les règles).
- `grep -rn "management" /etc/pve/firewall/` → rien ; c'est un ipset système auto-généré.

### 3.4 Bascule vers le backend nftables (81d51f90, a63655cb, a74256cd)
- `pvesh set /nodes/pve/...` échouait : `hostname lookup 'pve' failed` → le nœud s'appelle **`proxmox`** (`hostname` = proxmox). Utiliser `NODE=$(hostname -s)`.
- `host.fw` avait `enable: 0` → rien n'était généré. Corrigé : `pvesh set /nodes/$NODE/firewall/options -enable 1 -nftables 1`.
- `systemctl stop/disable pve-firewall`, `update-alternatives --set iptables iptables-nft` (+ ip6tables), `systemctl restart proxmox-firewall`.
- Résultat : `table inet proxmox-firewall` existe ; `proxmox-firewall 1.2.3`, `pve-manager 9.2.25`.
- ⚠️ Le nouveau backend n'utilise plus les noms de chaînes `PVEFW-*` : `nft list ruleset | grep -c PVEFW` = 0 est **normal**. Lire avec `nft list table inet proxmox-firewall`.

### 3.5 Règles FORWARD ignorées à cause du champ `-i` (82a076e4, 792498a7)
- Log clé : `skipping rule ... dir: Forward ... iface: Some("LANlab") ... : cannot define interfaces for forward direction`.
- Le backend Rust **refuse les interfaces sur les règles forward** → toutes les règles FORWARD étaient sautées (`cluster-forward` = `drop` seul).
- Correction : `sed -i '/^FORWARD/ s/ -i [A-Za-z0-9]*//' /etc/pve/firewall/cluster.fw` (backup `cluster.fw.bak2`) → après restart, `cluster-forward` contient les 6 DROP loggés + 2 ACCEPT + `drop` final. ✅ (vérifié, sortie complète dans la conversation)

### 3.6 `policy_forward` (fff481be)
- Le TRACE montrait `policy_forward: ACCEPT` (hérité) → remis à **DROP** dans `Datacenter → Firewall → Options`. Vérifié : `pvesh get /cluster/firewall/options` → DROP.

### 3.7 DNS du nœud bloquait `apt` (3ad13edc)
- `pve → System → DNS` pointait sur `192.168.1.53` (DNS interne) ; la règle `OUT DROP 192.168.1.0/24` bloquait → `Temporary failure resolving 'deb.debian.org'`.
- L'utilisateur a choisi de **garder `.99`** → ajout de 2 règles OUT ACCEPT (udp+tcp 53) vers `192.168.1.53/32` **avant** le DROP. Vérifié en nft : `oifname "vmbr0" udp dport 53 ip daddr 192.168.1.53 accept` (+ tcp). `apt update` repasse ✅.

### 3.8 dnsmasq / DHCP SDN (6d1dc365, a130a1ed, 3ad13edc, af180f13, 19649550, 395a8c60, 51400b20, e441325b, d08c61bc)
- Symptôme : VM Windows 100 en APIPA `169.254.196.216`, `ipconfig /renew` = « Impossible de contacter votre serveur DHCP » (b59875a0, 395a8c60_WNSsHGjv).
- `dnsmasq.service does not exist` → `apt install -y dnsmasq` (après correction DNS) ✅, mais le service **générique** doit être **désactivé** : le SDN utilise des instances par zone (`dnsmasq@zones1`, `dnsmasq@zones2`).
- `pvesh set /cluster/sdn/zones/zones1 -dhcp dnsmasq` (+ zones2) → `zones.cfg` : `dhcp dnsmasq` + `ipam pve` ✅.
- `pvesh set /cluster/sdn` (apply) → génère `/etc/dnsmasq.d/zones1/`, `zones2/` ; `ss -lunp | grep :67` → écoute sur `%LANlab:67` et `%Glab:67` ✅.
- Conf générée : `dhcp-range=set:zones1-10.10.1.0-24,10.10.1.0,static,...` + `dhcp-ignore=tag:!known` → **mode static** : seules les MAC dans l'IPAM reçoivent un lease.
- MAC de la VM 100 enregistrée : `/etc/dnsmasq.d/zones1/ethers` = `AA:BB:CC:DD:EE:01,10.10.1.225` ✅.
- « Automatic DHCP » sur la zone = choix **DHCP dynamique vs static par VM** (IPAM : VM sans IP fixée = dynamique dans la plage ; VM avec IP = statique).
- L2 vérifié OK : `tap100i0 ... master LANlab state UP`, `qm config 100` → `net0: e1000=AA:BB:CC:DD:EE:01,bridge=LANlab,firewall=1` (pas de `link=0`).
- tcpdump sur LANlab pendant renew : **0 paquet** (premier test, renew interrompu par ^C) → à refaire proprement (voir procédure finale).
- Règles firewall « DHCP DISCOVER » présentes et cochées (e441325b_vzkU1mkm).

## 4. État final constaté

| Élément | État |
|---|---|
| `vmbr0` | `192.168.1.100/24`, gw `192.168.1.1`, port `nic0` ✅ |
| `ip_forward` | 1 persistant ✅ |
| SDN | `LANlab`/`Glab` créés, IPs gateway `.254/24` sur les bridges, SNAT → `192.168.1.100` en POSTROUTING (x2, doublon inoffensif) ✅ |
| Backend pare-feu | nftables (`proxmox-firewall` actif, `pve-firewall` disabled, iptables→nft) ✅ |
| Options DC | enable=1, in=DROP, out=ACCEPT, forward=DROP ✅ |
| Règles IN/OUT hôte | 6 règles actives, IP en dur (`.20/32` pour 22/8006; DNS/NTP vers `.1`; DNS vers `.99`; DROP LAN) ✅ compilées en nft (`cluster-in` lignes 389-394) |
| Règles FORWARD | 8 règles sans `-i`, compilées dans `cluster-forward` (6 DROP loggés + 2 ACCEPT + drop final) ✅ |
| Aliases | `local_network`/`cluster_network` = `192.168.1.100/32` ✅ |
| IPSet management | supprimé (plus d'ouverture auto du LAN) ✅ |
| dnsmasq SDN | `dnsmasq@zones1`/`@zones2` actifs, UDP 67 sur LANlab/Glab, ethers contient la VM 100 ✅ |
| VM 100 (Windows 10) | `net0: bridge=LANlab, firewall=1`, MAC `AA:BB:CC:DD:EE:01` — **en attente de lease DHCP** (dernier point ouvert) |

## 5. Point ouvert à la fin

La VM Windows 100 ne recevait pas encore de lease au moment de la dernière capture (`/renew` bloqué). Dernières pistes vérifiées : ethers OK, dnsmasq zones actif, L2 OK, règles firewall cochées. Procédure de test restante : tcpdump synchronisé (40s) + `ipconfig /release && /renew` sans Ctrl+C + `journalctl -u dnsmasq@zones1` + `nft ... cluster-in | grep 'dport 6[78]'` + table bridge. Interprétation dans `conversation.md` section 3.8 et dernier message.

## 6. Tests finaux prévus (une fois DHCP OK)

| Depuis | Test | Attendu |
|---|---|---|
| PC `.20` | `ssh root@192.168.1.100`, `https://192.168.1.100:8006` | ✅ |
| Autre appareil LAN (téléphone Wi-Fi) | `ping 192.168.1.100`, port 8006 | ❌ (log Firewall → Log) |
| VM LANlab | `ping 8.8.8.8` | ✅ SNAT |
| VM LANlab | `ping 10.20.1.x` (Glab), `ping 192.168.1.10` | ❌ ❌ |
| VM Glab | `ping 8.8.8.8` puis `ping 10.10.1.x` | ✅ puis ❌ |
| VM LANlab | `ping 10.10.1.y` (même groupe) | ✅ L2 intra-bridge |
| Autre appareil LAN | `ping 10.10.1.x` | ❌ |
| Hôte | `ping 192.168.1.10` / `apt update` | ❌ / ✅ |
| Reboot | tout persiste | ✅ |
