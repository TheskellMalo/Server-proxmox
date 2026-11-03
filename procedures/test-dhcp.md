# Procédure de test DHCP — VM 100 (Windows 10) sur LANlab

Objectif : faire tomber le dernier point ouvert — la VM 100 reste en APIPA
(`169.254.196.216`) alors que dnsmasq@zones1 écoute sur `%LANlab:67` et que
sa MAC (`AA:BB:CC:DD:EE:01`) est dans l'IPAM (`10.10.1.225`).

Principe : on ne relance **rien au hasard**. Chaque test produit une lecture
unique (tcpdump + logs + nft) qui désigne une seule cause possible.

---

## Étape 0 — Remise à plat (2 min, à faire une seule fois)

Sur le nœud Proxmox :

```bash
# 1. Redémarrer dnsmasq pour forcer la relecture d'ethers
systemctl restart dnsmasq@zones1

# 2. Vérifier la conf effective : le mot "static" dans dhcp-range est le suspect n°1
grep dhcp-range /etc/dnsmasq.d/zones1/*.conf
# Attendu si Automatic DHCP actif :
#   dhcp-range=set:zones1-10.10.1.0-24,10.10.1.225,10.10.1.253,255.255.255.0,<durée>
# Si "static" est TOUJOURS présent → la VM sans tag "known" n'obtiendra JAMAIS
# de lease → aller au cas D de l'interprétation.

# 3. Confirmer l'écoute DHCP
ss -lunp | grep :67
# Attendu : 0.0.0.0%LANlab:67 (dnsmasq@zones1)

# 4. Confirmer que la MAC est dans ethers
cat /etc/dnsmasq.d/zones1/ethers
# Attendu : AA:BB:CC:DD:EE:01,10.10.1.225
```

Si le `static` persiste malgré « Automatic DHCP » coché et un Apply SDN
(`pvesh set /cluster/sdn`) : forcer une IP fixe par VM dans l'IPAM
(SDN → IPAM → VM 100 → 10.10.1.225), re-Apply, re-vérifier le dhcp-range.

---

## Étape 1 — Test synchronisé (le test de référence)

**Hôte — terminal 1** (à lancer EN PREMIER, avant le renew) :

```bash
# Capture tout le trafic DHCP sur le bridge LANlab pendant 40 s
timeout 40 tcpdump -ni LANlab udp port 67 or udp port 68 -vv
```

**Hôte — terminal 2** :

```bash
journalctl -u dnsmasq@zones1 -f
```

**VM Windows — cmd (attendre ~2 s après le lancement du tcpdump)** :

```bat
ipconfig /release
ipconfig /renew
```

Ne pas interrompre : laisser le renew rendre la main tout seul (erreur ou succès).

Puis côté hôte : récupérer l'état du filtrage pendant la fenêtre du test :

```bash
nft list chain inet proxmox-firewall cluster-in  | grep 'dport 6[78]'
nft list table bridge                             2>/dev/null | grep -E '67|68|drop'
```

---

## Étape 2 — Interprétation (une lecture = une cause)

| Lecture tcpdump / logs | Diagnostic | Correctif |
|---|---|---|
| **A.** 0 paquet vu côté hôte, et Windows affiche l'erreur DHCP | La VM n'émet pas ou les paquets n'atteignent pas le bridge | Dans la VM : `sc query dhcp` (service Dhcp RUNNING ?), désactiver/réactiver la carte réseau, vérifier pare-feu Windows ; côté hôte : `ip link show tap100i0` (UP, master LANlab) |
| **B.** DISCOVER vu (`0.0.0.0.68 > 255.255.255.255.67`), **aucun OFFER**, rien dans le journal dnsmasq | dnsmasq ne reçoit pas ou ignore la requête (filtre INPUT/bridge, ou tag `!known`) | Règle INPUT udp 67 absente de `cluster-in` → l'ajouter (Firewall → règles in : ACCEPT udp 67/68) ; sinon vérifier la table `bridge` (un drop y court-circuite le DHCP) |
| **C.** DISCOVER vu + log dnsmasq `DHCPDISCOVER(LANlab) ...AA:BB:CC:DD:EE:01` **sans** `DHCPOFFER` | dnsmasq reçoit mais refuse : MAC non `known` (mode static/IPAM) | Cas D : fixer l'IP dans l'IPAM ou activer Automatic DHCP, re-Apply, restart dnsmasq@zones1 |
| **D.** Log dnsmasq `DHCPOFFER(LANlab) 10.10.1.225 ...` mais tcpdump ne montre pas d'OFFER sortant, ou l'OFFER sort et la VM n'ACK pas | Problère côté VM (pare-feu Windows) ou OFFER perdu en L2 | Désactiver temporairement le pare-feu Windows (`netsh advfirewall set allprofiles state off`), refaire le test ; vérifier que le veth/tap n'est pas dans un état down |
| **E.** DISCOVER → OFFER → REQUEST → ACK complets, IP `10.10.1.225` | ✅ Résolu | Passer à l'étape 3 |

Rappel utile : dans le journal dnsmasq, une ligne
`DHCPDISCOVER(LANlab) 00:00:00:00:00:00 ignored` (ou `not giving ... to ...`)
signifie explicitement le rejet par `dhcp-ignore=tag:!known` → cas C/D.

---

## Étape 3 — Validation réseau complète de la VM (une fois l'IP obtenue)

```bat
:: VM — vérifier l'IP et la passerelle
ipconfig /all                    :: attendu 10.10.1.225, gw 10.10.1.254, DNS via dnsmasq

:: Connectivité montante
ping 10.10.1.254                 :: gateway SDN (LANlab)
ping 8.8.8.8                     :: Internet via SNAT → attendu OK
nslookup deb.debian.org          :: résolution via dnsmasq → hôte → DNS .99

:: Isolation (attendu : ÉCHEC = TIMEOUT, pas de réponse)
ping 10.20.1.254                 :: Glab → doit échouer
ping 192.168.1.10              :: PC admin → doit échouer
ping 192.168.1.1               :: box → doit échouer
```

Puis depuis le PC admin (`192.168.1.10`) et un autre appareil LAN (téléphone) :

| Test | Attendu |
|---|---|
| PC .20 → `https://192.168.1.100:8006` et `ssh root@192.168.1.100` | ✅ |
| Téléphone → ping `.103` / port 8006 | ❌ (vérifier le log : Datacenter → Firewall → Log, DROP loggé) |
| Téléphone → ping `10.10.1.225` | ❌ (règle FORWARD LAN → LANlab) |
| VM LANlab → ping une 2e VM LANlab (même zone) | ✅ (L2 intra-bridge) |
| Hôte → `ping 192.168.1.10` | ❌ (OUT DROP LAN) |
| Hôte → `apt update` | ✅ (DNS .99 autorisé) |

## Étape 4 — Test de persistance

```bash
reboot
# après redémarrage, re-vérifier :
sysctl net.ipv4.ip_forward                # = 1 (99-ip-forward.conf)
systemctl is-active proxmox-firewall dnsmasq@zones1 dnsmasq@zones2
nft list table inet proxmox-firewall | head
ss -lunp | grep :67
```

Puis refaire `ipconfig /renew` dans la VM et re-passer la matrice de l'étape 3.
Tout persiste → projet bouclé. 🎉
