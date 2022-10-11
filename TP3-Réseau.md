# TP3 : On va router des trucs


## I. ARP

Première partie simple, on va avoir besoin de 2 VMs.

| Machine  | `10.3.1.0/24` |
|----------|---------------|
| `john`   | `10.3.1.11`   |
| `marcel` | `10.3.1.12`   |

```schema
   john               marcel
  ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     │
  └─────┘    └───┘    └─────┘
```

> Référez-vous au [mémo Réseau Rocky](../../cours/memo/rocky_network.md) pour connaître les commandes nécessaire à la réalisation de cette partie.

### 1. Echange ARP

🌞**Générer des requêtes ARP**

- effectuer un `ping` d'une machine à l'autre

```
[root@localhost ~]$ ping 10.3.1.11
PING 10.3.1.11 (10.3.1.11) 56(84) bytes of data.
64 bytes from 10.3.1.11: icmp_seq=1 ttl=64 time=1.53 ms
64 bytes from 10.3.1.11: icmp_seq=2 ttl=64 time=0.762 ms
```
- observer les tables ARP des deux machines

Côté john
```
[root@localhost ~]$ ip neigh show dev enp0s8
10.3.1.12 lladdr 08:00:27:f9:97:33 STALE
```

Côté marcel
```
[root@localhost ~]$ ip neigh show dev enp0s8
10.3.1.11 lladdr 08:00:27:63:f7:af STALE
```
- repérer l'adresse MAC de `john` dans la table ARP de `marcel` et vice-versa

Adresse MAC de `john` : 08:00:27:63:f7:af

Adresse MAC de `marcel` : 08:00:27:f9:97:33

- prouvez que l'info est correcte (que l'adresse MAC que vous voyez dans la table est bien celle de la machine correspondante)
  - une commande pour voir la MAC de `marcel` dans la table ARP de `john`
```
[root@localhost ~]$ ip neigh show 10.3.1.12
```
  - et une commande pour afficher la MAC de `marcel`, depuis `marcel`

```
[root@localhost ~]$ ip link show

2: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000ot found
    link/ether 08:00:27:f9:97:33 brd ff:ff:ff:ff:ff:ff
```

### 2. Analyse de trames

🌞**Analyse de trames**

- utilisez la commande `tcpdump` pour réaliser une capture de trame

```
[root@localhost ~]$ tcpdump -i enp0s8 -c 10 -w tp3_arp.pcap not port 22
```
- videz vos tables ARP, sur les deux machines, puis effectuez un `ping`

```
[root@localhost ~]$ ip neigh flush all
```

🦈 **Capture réseau `tp2_arp.pcapng`** qui contient un ARP request et un ARP reply

**[tp3_arp.pcapng](wireshark/tp3_arp.pcap)**

> **Si vous ne savez pas comment récupérer votre fichier `.pcapng`** sur votre hôte afin de l'ouvrir dans Wireshark, et me le livrer en rendu, demandez-moi.

## II. Routage

Vous aurez besoin de 3 VMs pour cette partie. **Réutilisez les deux VMs précédentes.**

| Machine  | `10.3.1.0/24` | `10.3.2.0/24` |
|----------|---------------|---------------|
| `router` | `10.3.1.254`  | `10.3.2.254`  |
| `john`   | `10.3.1.11`   | no            |
| `marcel` | no            | `10.3.2.12`   |

> Je les appelés `marcel` et `john` PASKON EN A MAR des noms nuls en réseau 🌻

```schema
   john                router              marcel
  ┌─────┐             ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     ├────┤ho2├────┤     │
  └─────┘    └───┘    └─────┘    └───┘    └─────┘
```

### 1. Mise en place du routage

🌞**Activer le routage sur le noeud `router`**

```
[root@localhost ~]$ firewall-cmd --add-masquerade --zone=public
success

[root@localhost ~]$ firewall-cmd --list-all
  masquerade: yes
```

🌞**Ajouter les routes statiques nécessaires pour que `john` et `marcel` puissent se `ping`**

- il faut ajouter une seule route des deux côtés

Côté `john`
```
[root@localhost ~]$ ip route add 10.3.2.0/24 via 10.3.1.254 dev enp0s8
```

Côté `marcel`
```
[root@localhost ~]$ ip route add 10.3.1.0/24 via 10.3.2.254 dev enp0s8
```
- une fois les routes en place, vérifiez avec un `ping` que les deux machines peuvent se joindre

```
[root@localhost ~]$ ping 10.3.2.12
PING 10.3.2.12 (10.3.2.12) 56(84) bytes of data.
64 bytes from 10.3.2.12: icmp_seq=1 ttl=63 time=2.60 ms
64 bytes from 10.3.2.12: icmp_seq=2 ttl=63 time=1.89 ms
```

```
[root@localhost ~]$ ping 10.3.1.11
PING 10.3.1.11 (10.3.1.11) 56(84) bytes of data.
64 bytes from 10.3.1.11: icmp_seq=1 ttl=63 time=1.20 ms
```


### 2. Analyse de trames

🌞**Analyse des échanges ARP**

- videz les tables ARP des trois noeuds
- effectuez un `ping` de `john` vers `marcel`
- regardez les tables ARP des trois noeuds
- essayez de déduire un peu les échanges ARP qui ont eu lieu
- répétez l'opération précédente (vider les tables, puis `ping`), en lançant `tcpdump` sur `marcel`
- **écrivez, dans l'ordre, les échanges ARP qui ont eu lieu, puis le ping et le pong, je veux TOUTES les trames** utiles pour l'échange


| ordre | type trame  | IP source | MAC source                   | IP destination | MAC destination               |
|-------|-------------|-----------|------------------------------|----------------|-------------------------------|
| 1     | Requête ARP | x         | `router` `08:00:27:94:24:b6` | x              | Broadcast `ff:ff:ff:ff:ff:ff` |
| 2     | Réponse ARP | x         | `marcel` `08:00:27:f9:97:33` | x              | `router` `08:00:27:94:24:b6`  |
| 3     | Ping        | 10.3.2.254| 08:00:27:94:24:b6            | 10.3.2.12      | 08:00:27:f9:97:33             |
| 4     | Pong        | 10.3.2.12 | 08:00:27:f9:97:33            | 10.3.2.254     | 08:00:27:94:24:b6             |
| 5     | Requête ARP | x         | `marcel` `08:00:27:f9:97:33` | x              | `router` `08:00:27:94:24:b6`  |
| 6     | Réponse ARP | x         | `router` `08:00:27:94:24:b6` | x              | `marcel` `08:00:27:f9:97:33`  |


> Vous pourriez, par curiosité, lancer la capture sur `john` aussi, pour voir l'échange qu'il a effectué de son côté.

🦈 **Capture réseau `tp2_routage_marcel.pcapng`**

**[TP3_ROUTAGE_MARCEL](wireshark/tp3_routage_marcel.pcap)**

### 3. Accès internet

🌞**Donnez un accès internet à vos machines**

- ajoutez une carte NAT en 3ème inteface sur le `router` pour qu'il ait un accès internet

```
[root@localhost ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000

2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:7d:ad:70 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute enp0s3
       valid_lft 84739sec preferred_lft 84739sec
    inet6 fe80::a00:27ff:fe7d:ad70/64 scope link noprefixroute
       valid_lft forever preferred_lft forever

3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000

4: enp0s9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
```

- ajoutez une route par défaut à `john` et `marcel`

Côté `john`
```
[root@localhost ~]$ ip route add default via 10.3.1.254 dev enp0s8
```

Côté `marcel`
```
[root@localhost ~]$ ip route add default via 10.3.2.254 dev enp0s8
```
  - vérifiez que vous avez accès internet avec un `ping`

Côté `john`
```
[root@localhost ~]$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=112 time=24.4 ms
```

Côté `marcel`
```
[root@localhost ~]$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=112 time=23.9 ms
```
  - le `ping` doit être vers une IP, PAS un nom de domaine
- donnez leur aussi l'adresse d'un serveur DNS qu'ils peuvent utiliser
  - vérifiez que vous avez une résolution de noms qui fonctionne avec `dig`
  - puis avec un `ping` vers un nom de domaine

🌞**Analyse de trames**

- effectuez un `ping 8.8.8.8` depuis `john`
- capturez le ping depuis `john` avec `tcpdump`
- analysez un ping aller et le retour qui correspond et mettez dans un tableau :

| ordre | type trame | IP source          | MAC source                 | IP destination     | MAC destination            |
|-------|------------|--------------------|----------------------------|--------------------|----------------------------|
| 1     | ping       | `john` `10.3.1.11` | `john` `08:00:27:63:f7:af` | `8.8.8.8`          | `08:00:27:27:f6:5a`        |
| 2     | pong       | `8.8.8.8`          | `08:00:27:27:f6:5a`        | `john` `10.3.1.11` | `john` `08:00:27:63:f7:af` |

🦈 **Capture réseau `tp2_routage_internet.pcapng`**

**[TCP3_ROUTAGE](wireshark/tp3_routage_internet.pcapng)**

## III. DHCP

On reprend la config précédente, et on ajoutera à la fin de cette partie une 4ème machine pour effectuer des tests.

| Machine  | `10.3.1.0/24`              | `10.3.2.0/24` |
|----------|----------------------------|---------------|
| `router` | `10.3.1.254`               | `10.3.2.254`  |
| `john`   | `10.3.1.11`                | no            |
| `bob`    | oui mais pas d'IP statique | no            |
| `marcel` | no                         | `10.3.2.12`   |

```schema
   john               router              marcel
  ┌─────┐             ┌─────┐             ┌─────┐
  │     │    ┌───┐    │     │    ┌───┐    │     │
  │     ├────┤ho1├────┤     ├────┤ho2├────┤     │
  └─────┘    └─┬─┘    └─────┘    └───┘    └─────┘
   john        │
  ┌─────┐      │
  │     │      │
  │     ├──────┘
  └─────┘
```

### 1. Mise en place du serveur DHCP

🌞**Sur la machine `john`, vous installerez et configurerez un serveur DHCP** (go Google "rocky linux dhcp server").

- installation du serveur sur `john`
- créer une machine `bob`
- faites lui récupérer une IP en DHCP à l'aide de votre serveur

> Il est possible d'utiliser la commande `dhclient` pour forcer à la main, depuis la ligne de commande, la demande d'une IP en DHCP, ou renouveler complètement l'échange DHCP (voir `dhclient -h` puis call me et/ou Google si besoin d'aide).

🌞**Améliorer la configuration du DHCP**

- ajoutez de la configuration à votre DHCP pour qu'il donne aux clients, en plus de leur IP :
  - une route par défaut
  - un serveur DNS à utiliser
- récupérez de nouveau une IP en DHCP sur `bob` pour tester :
  - `marcel` doit avoir une IP
    - vérifier avec une commande qu'il a récupéré son IP
    - vérifier qu'il peut `ping` sa passerelle
  - il doit avoir une route par défaut
    - vérifier la présence de la route avec une commande
    - vérifier que la route fonctionne avec un `ping` vers une IP
  - il doit connaître l'adresse d'un serveur DNS pour avoir de la résolution de noms
    - vérifier avec la commande `dig` que ça fonctionne
    - vérifier un `ping` vers un nom de domaine

### 2. Analyse de trames

🌞**Analyse de trames**

- lancer une capture à l'aide de `tcpdump` afin de capturer un échange DHCP
- demander une nouvelle IP afin de générer un échange DHCP
- exportez le fichier `.pcapng`

🦈 **Capture réseau `tp2_dhcp.pcapng`**