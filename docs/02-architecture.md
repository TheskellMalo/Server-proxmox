# 02 — L'architecture retenue

## Vue d'ensemble

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
```

## Les briques et pourquoi

| Brique | Choix | Pourquoi |
|---|---|---|
| Réseau hôte | `vmbr0` ponté sur l'interface physique, `192.168.1.100/24` | seul lien vers la box/Internet |
| Zones VM | SDN **zones Simple** (pas VLAN) | la box ne gère pas le 802.1Q |
| VNet Lab | `10.10.1.0/24`, gw `.254`, SNAT coché, DHCP `.225-.253` | groupe « test d'applications » |
| VNet Games | `10.20.1.0/24`, gw `.254`, SNAT coché, DHCP `.225-.253` | groupe isolé du premier |
| DHCP/DNS | `dnsmasq` par zone (`dnsmasq@zone1`, `@zone2`) + IPAM `pve` | instances par zone, IPs statiques maîtrisées |
| Backend pare-feu | **nftables** (`proxmox-firewall`) | l'ancien backend iptables **ignore les règles FORWARD** — sans lui, aucune isolation inter-zones |
| Policies | in `DROP` · out `ACCEPT` · forward `DROP` · ebtables activé | tout interdit par défaut, ouverture au cas par cas |

## Les règles finales (datacenter)

**IN** (accès à l'hôte) :
- ACCEPT SSH depuis `192.168.1.10/32` → `192.168.1.100/32`
- ACCEPT tcp 8006 (+ 3128 proxy SPICE) depuis `192.168.1.10/32` → `.100`
- ACCEPT udp 68→67 sur chaque VNet, **Source et Destination vides** (DHCP broadcast)
- ACCEPT udp 53 vers la passerelle de chaque VNet (DNS des VMs via dnsmasq)
- DROP tout le LAN `192.168.1.0/24` → `.100` (loggé)

**OUT** (le nœud vers le monde) :
- ACCEPT DNS vers `192.168.1.53` (DNS interne) et NTP vers `192.168.1.1`
- DROP vers `192.168.1.0/24` — le reste (Internet) passe en policy ACCEPT

**FORWARD** (le trafic qui traverse) :
- DROP Lab ↔ Games, Lab → LAN, LAN → Lab, Games → LAN, LAN → Games
- ACCEPT Lab → (Internet), ACCEPT Games → (Internet) — via SNAT
- **Jamais d'interface sur les règles forward** (le backend les saute sinon)

## Ce qui a marché du premier coup

- Le SNAT du SDN : les VMs avaient Internet dès la création du premier VNet.
- L'isolation FORWARD : une fois le backend nftables actif et les interfaces
  retirées des règles forward, les 6 DROP se sont compilés et le blocage a été
  immédiat et vérifiable.
- La persistance : `/etc/sysctl.d/` + services systemd + config SDN → tout a
  tenu au reboot du premier coup.

## Points de fragilité connus

- Le filtrage des VMs repose sur les **règles datacenter forward** ; le pare-feu
  au niveau VM est désactivé (défense en profondeur possible mais non retenue).
- L'IPSet `management` peut être régénéré par PVE : à vérifier à chaque audit
  (voir `04-audit-securite.md`).
