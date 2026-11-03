# Audit du pare-feu — Chasse aux backdoors (3 novembre 2026)

Méthode : 8 écrans de l'interface Proxmox + lecture du Log + règles compilées en nft.

## Verdict : AUCUNE BACKDOOR DÉTECTÉE

| Écran / point de contrôle | Attendu | Constat |
|---|---|---|
| Datacenter → Options | enable=1, in/out/forward = DROP/ACCEPT/DROP | ✅ conforme, ebtables=Oui, aucun « bypass firewall on interface » |
| Datacenter → IPSet | vide | ✅ **vide** — l'IPSet `management` (qui ouvrait 22/8006/5900-5999/3128/60000-60050 à tout le LAN) n'est **pas** réapparu |
| Datacenter → Alias | `.103/32` uniquement | ✅ `local_network` et `cluster_network` = 192.168.1.100 |
| Datacenter → Security Groups | vide | ✅ vide, aucun groupe inséré |
| Nœud → Règles | vide | ✅ « No firewall rule configured here » — aucune règle cachée au niveau hôte |
| Nœud → Options | enable + nftables | ✅ Pare-feu Oui, nftables Oui, SMURFS Oui |
| VM 100 → Options | — | ℹ️ Pare-feu VM = No (voir observation 1) |
| Log | seulement du légitime | ✅ uniquement ACCEPT depuis .20 (8006) et DNS des VMs vers .254 ; aucune IP inconnue |

## Ordre des règles vérifié (cluster.fw)

- IN : ACCEPT SSH/8006 depuis `.20/32` **avant** le DROP `192.168.1.0/24 → .103` ✅
- OUT : ACCEPT DNS `.99` / NTP `.1` **avant** le DROP vers `192.168.1.0/24` ✅
- FORWARD : 6 DROP (LANlab↔Glab, LAN↔zones) **avant** les 2 ACCEPT vers Internet ✅
  (critique : sinon les VMs pourraient atteindre le LAN ou l'autre zone)

## Observations (non-bloquantes)

1. **Pare-feu VM 100 désactivé** : pas une faille (l'isolation vient des règles
   FORWARD datacenter), mais le `firewall=1` de net0 est inerte. La défense en
   profondeur consisterait à l'activer avec in=DROP / out=ACCEPT.
2. **DROP du téléphone non visible dans la fenêtre de log examinée** : le ping d'un
   appareil LAN doit générer `DROP: IN=vmbr0 SRC=<IP téléphone>` (règles DROP en
   `-log warning`) — à confirmer en refaisant le test pour la traçabilité.
3. **Règles inutiles supprimées** : les 2 règles « DHCP descendant / renew unicast »
   (dest `10.10.1.0/24` et `10.20.1.0/24`) — le trafic concerné ne passe pas par
   INPUT ; la règle DHCP sans destination couvre tout.
4. **Fix Glab appliqué** : règle DHCP Glab éditée avec Source et Destination vides,
   compilée `iifname "Glab" udp sport 68 udp dport 67 accept` (sans `daddr`) ✅
5. **Niveaux de log** : `warning` (limité 1/s burst 5) = bon choix en production ;
   `debug` journalise chaque paquet sans limite de débit — ponctuel uniquement.
   Le `log_level_in=nolog` du nœud peut rester : les logs par règle suffisent.

## État final des règles cluster (après audit)

- IN : SSH(.20), 8006+3128(.20), DHCP LANlab (udp 68→67, sans dest), DNS LANlab
  (udp 53 → .254), DHCP Glab, DNS Glab, DROP LAN→.103
- OUT : DNS(.99), NTP(.1), DROP vers LAN
- FORWARD : 6 DROP inter-zones/LAN, 2 ACCEPT zones→Internet (SNAT)
- Policies : in=DROP, out=ACCEPT, forward=DROP, ebtables=Oui
