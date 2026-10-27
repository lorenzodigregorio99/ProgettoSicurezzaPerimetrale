# ProgettoSicurezzaPerimetrale

Infrastruttura di rete virtuale con **firewall iptables**, **DMZ** con server **OpenVPN** e **server SSH protetto da certificati X.509** emessi da una CA privata. Tutto realizzato in laboratorio su VMware Workstation.

## Obiettivo

- Un firewall Ubuntu che separa tre reti: WAN, DMZ e LAN, con politica predefinita restrittiva (`INPUT` e `FORWARD` in `DROP`).
- Un server VPN in DMZ raggiungibile dall'esterno tramite DNAT.
- Un server SSH nella LAN accessibile **solo con certificato**, senza password.
- Amministrazione di tutte le macchine tramite SSH passando dal firewall (bastion host).

## Schema di rete

```
                     PC fisico (WSL + OpenVPN)
                              | 10.0.2.1
                              |
                    VMnet8 - NAT - 10.0.2.0/24
                              | ens33  10.0.2.5
                     +--------+--------+
                     |    FIREWALL     |
                     |  Ubuntu+iptables|
                     +---+---------+---+
              ens37      |         |      ens38
          192.168.56.2   |         |   192.168.57.2
                         |         |
        VMnet1 - DMZ     |         |    VMnet2 - LAN
        192.168.56.0/24  |         |    192.168.57.0/24
                         |         |
                  +------+---+  +--+---------+
                  |  DMZ_VPN |  |  ssh-lan   |
                  | OpenVPN  |  | PKIX-SSH   |
                  |192.168.  |  |192.168.    |
                  |  56.10   |  |  57.10     |
                  | tun0     |  | porta 2223 |
                  | 10.8.0.1 |  +------------+
                  +----------+
```

Se disponibile, lo schema disegnato a mano è in `schema-rete.png`.

## Macchine

| Macchina | Ruolo | Rete VMware | Indirizzo |
|---|---|---|---|
| Firewall | Router e firewall | VMnet8 (NAT) | `10.0.2.5` (ens33) |
| Firewall | | VMnet1 (host-only) | `192.168.56.2` (ens37) |
| Firewall | | VMnet2 (host-only) | `192.168.57.2` (ens38) |
| DMZ_VPN | Server OpenVPN | VMnet1 | `192.168.56.10`, tunnel `tun0` `10.8.0.1` |
| ssh-lan | Server PKIX-SSH con certificati | VMnet2 | `192.168.57.10`, porta `2223` |
| PC fisico | Client (WSL + OpenVPN) | VMnet8 | `10.0.2.1` |

Il DHCP è disattivato su tutte le reti virtuali e le macchine usano indirizzi statici (netplan).

## Flussi principali

| Servizio | Percorso |
|---|---|
| VPN | PC → port forwarding VMware UDP 1194 → firewall `10.0.2.5` → DNAT → `192.168.56.10:1194` |
| SSH con certificato | PC → port forwarding VMware TCP 2223 → firewall → DNAT → `192.168.57.10:2223` |
| Amministrazione | WSL → `ssh firewall` → `ProxyJump` verso `dmz` e `lorenzo` |

## Contenuto del repository

| Cartella | Contenuto |
|---|---|
| [`firewall/`](firewall/README.md) | Reti VMware, indirizzi statici, regole iptables, DNAT, port forwarding |
| [`dmz/`](dmz/README.md) | Server OpenVPN, client Windows e verifica del tunnel |
| [`ssh-certificati/`](ssh-certificati/README.md) | CA privata, certificato client, PKIX-SSH e accesso senza password |

## Risultati

- Tunnel OpenVPN stabilito dal PC fisico attraverso firewall e DNAT (client `10.8.0.2`).
- Login SSH con certificato X.509 su `ssh-lan`, passando dal firewall.
- Accesso con password rifiutato: il server risponde solo `Permission denied (publickey)`.

## Limiti noti e sviluppi futuri

- Dalla VPN la LAN non è ancora raggiungibile: il server spinge la rotta verso `192.168.57.0/24`, ma sul firewall manca la regola di `FORWARD` da `ens37` verso `ens38`.
- Le reti DMZ e LAN non hanno accesso a internet: manca una regola di `MASQUERADE` sul firewall.
- I certificati SSH hanno validità di 365 giorni e vanno rinnovati.

## Tecnologie

VMware Workstation, Ubuntu Server, iptables / iptables-persistent, netplan, OpenVPN, OpenSSL, PKIX-SSH, WSL.

## Sicurezza del repository

Chiavi private, certificati e profili VPN non sono inclusi (vedi `.gitignore`). Gli indirizzi IP sono tutti privati e appartengono al laboratorio.
