# TP4 : TCP, UDP et services réseau

# Sommaire

- [TP4 : TCP, UDP et services réseau](#tp4--tcp-udp-et-services-réseau)
- [Sommaire](#sommaire)
- [I. First steps](#i-first-steps)
- [II. Mise en place](#ii-mise-en-place)
  - [1. SSH](#1-ssh)
  - [2. Routage](#2-routage)
- [III. DNS](#iii-dns)
  - [1. Présentation](#1-présentation)
  - [2. Setup](#2-setup)
  - [3. Test](#3-test)


# I. First steps

Faites-vous un petit top 5 des applications que vous utilisez sur votre PC souvent, des applications qui utilisent le réseau : un site que vous visitez souvent, un jeu en ligne, Spotify, j'sais po moi, n'importe.

🌞 **Déterminez, pour ces 5 applications, si c'est du TCP ou de l'UDP**

- avec Wireshark, on va faire les chirurgiens réseau
- déterminez, pour chaque application :
  - IP et port du serveur auquel vous vous connectez
  - le port local que vous ouvrez pour vous connecter

Discord :

- IP dst = 162.159.128.232
- Port dst = :443
- Port src = :55667
[DISCORD](wireshark/discord.pcapng)
```
PS C:\Windows\system32> netstat -n -b

  TCP    10.33.16.200:55667     162.159.128.232:443    ESTABLISHED
 [Discord.exe]
```

Spotify :

- IP dst = 35.188.42.15
- Port dst = :443
- Port src = :56434
[SPOTIFY](wireshark/Spotify.pcapng)
```
PS C:\Windows\system32> netstat -n -b

  TCP    10.33.16.200:56434     35.188.42.15:443    ESTABLISHED
 [Spotify.exe]
```

Steam :

- IP dst = 23.217.237.25
- Port dst = :443
- Port src = :56467
[STEAM](wireshark/steam.pcapng)
```
PS C:\Windows\system32> netstat -n -b

  TCP    10.33.16.200:56467     23.217.237.25:443       ESTABLISHED
 [Steam.exe]
```

Teams :

- IP dst = 8.8.8.8
- Port dst = :443
- Port src = :58064
[TEAMS](wireshark/teams.pcapng)
```
PS C:\Windows\system32> netstat -n -b

  TCP    10.33.16.200:58064     8.8.8.8:443            ESTABLISHED
 [Teams.exe]
```

Darkest Dungeon :

- IP dst = 8.8.8.8
- Port dst = :443
- Port src = :58064
[DARKEST_DUNGEON](wireshark/darkest-dungeon.pcapng)
```
PS C:\Windows\system32> netstat -n -b

  TCP    10.33.16.200:58363     44.238.28.230:443      ESTABLISHED
 [darkest.exe]
```


**Il faudra ajouter des options adaptées aux commandes pour y voir clair. Pour rappel, vous cherchez des connexions TCP ou UDP.**

```
# MacOS
$ netstat

# GNU/Linux
$ ss

# Windows
$ netstat
```

🦈🦈🦈🦈🦈 **Bah ouais, captures Wireshark à l'appui évidemment.** Une capture pour chaque application, qui met bien en évidence le trafic en question.

# II. Mise en place

## 1. SSH

🖥️ **Machine `node1.tp4.b1`**

- n'oubliez pas de dérouler la checklist (voir [les prérequis du TP](#0-prérequis))
- donnez lui l'adresse IP `10.4.1.11/24`

Connectez-vous en SSH à votre VM.

🌞 **Examinez le trafic dans Wireshark**

- **déterminez si SSH utilise TCP ou UDP**
  - TCP
- **repérez le *3-Way Handshake* à l'établissement de la connexion**
  - c'est le `SYN` `SYNACK` `ACK`
- **repérez du trafic SSH**
- **repérez le FIN FINACK à la fin d'une connexion**
- entre le *3-way handshake* et l'échange `FIN`, c'est juste une bouillie de caca chiffré, dans un tunnel TCP


🌞 **Demandez aux OS**

- repérez, avec une commande adaptée (`netstat` ou `ss`), la connexion SSH depuis votre machine
- ET repérez la connexion SSH depuis votre VM

Depuis le pc :
```
PS C:\Windows\system32> netstat -b -n

Connexions actives

  Proto  Adresse locale         Adresse distante       État
  TCP    10.4.1.1:56889         10.4.1.11:22           ESTABLISHED
 [ssh.exe]
```
Depuis la vm :

```
[hugoa@node1 ~]$ ss -t
State        Recv-Q        Send-Q               Local Address:Port               Peer Address:Port        Process
ESTAB        0             0                        10.4.1.11:ssh                    10.4.1.1:56889
```


🦈 **Je veux une capture clean avec le 3-way handshake, un peu de trafic au milieu et une fin de connexion**

[SSH](wireshark/ssh.pcapng)

## 2. Routage

Ouais, un peu de répétition, ça fait jamais de mal. On va créer une machine qui sera notre routeur, et **permettra à toutes les autres machines du réseau d'avoir Internet.**

🖥️ **Machine `router.tp4.b1`**

- n'oubliez pas de dérouler la checklist (voir [les prérequis du TP](#0-prérequis))
- donnez lui l'adresse IP `10.4.1.11/24` sur sa carte host-only
- ajoutez-lui une carte NAT, qui permettra de donner Internet aux autres machines du réseau
- référez-vous au TP précédent

> Rien à remettre dans le compte-rendu pour cette partie.

# III. DNS

## 1. Présentation

Un serveur DNS est un serveur qui est capable de répondre à des requêtes DNS.

Une requête DNS est la requête effectuée par une machine lorsqu'elle souhaite connaître l'adresse IP d'une machine, lorsqu'elle connaît son nom.

Par exemple, si vous ouvrez un navigateur web et saisissez `https://www.google.com` alors une requête DNS est automatiquement effectuée par votre PC pour déterminez à quelle adresse IP correspond le nom `www.google.com`.

> La partie `https://` ne fait pas partie du nom de domaine, ça indique simplement au navigateur la méthode de connexion. Ici, c'est HTTPS.

Dans cette partie, on va monter une VM qui porte un serveur DNS. Ce dernier répondra aux autres VMs du LAN quand elles auront besoin de connaître des noms. Ainsi, ce serveur pourra :

- résoudre des noms locaux
  - vous pourrez `ping node1.tp4.b1` et ça fonctionnera
  - mais aussi `ping www.google.com` et votre serveur DNS sera capable de le résoudre aussi

*Dans la vraie vie, il n'est pas rare qu'une entreprise gère elle-même ses noms de domaine, voire gère elle-même son serveur DNS. C'est donc du savoir ré-utilisable pour tous qu'on voit ici.*

> En réalité, ce n'est pas votre serveur DNS qui pourra résoudre `www.google.com`, mais il sera capable de *forward* (faire passer) votre requête à un autre serveur DNS qui lui, connaît la réponse.

![Haiku DNS](./pics/haiku_dns.png)

## 2. Setup

🖥️ **Machine `dns-server.tp4.b1`**

- n'oubliez pas de dérouler la checklist (voir [les prérequis du TP](#0-prérequis))
- donnez lui l'adresse IP `10.4.1.201/24`

Installation du serveur DNS :

```bash
# assurez-vous que votre machine est à jour
$ sudo dnf update -y

# installation du serveur DNS, son p'tit nom c'est BIND9
$ sudo dnf install -y bind bind-utils
```

La configuration du serveur DNS va se faire dans 3 fichiers essentiellement :

- **un fichier de configuration principal**
  - `/etc/named.conf`
  - on définit les trucs généraux, comme les adresses IP et le port où on veu écouter
  - on définit aussi un chemin vers les autres fichiers, les fichiers de zone
- **un fichier de zone**
  - `/var/named/tp4.b1.db`
  - je vous préviens, la syntaxe fait mal
  - on peut y définir des correspondances `IP ---> nom`
- **un fichier de zone inverse**
  - `/var/named/tp4.b1.rev`
  - on peut y définir des correspondances `nom ---> IP`

➜ **Allooooons-y, fichier de conf principal**

```bash
# éditez le fichier de config principal pour qu'il ressemble à :
$ sudo cat /etc/named.conf
options {
        listen-on port 53 { 127.0.0.1; any; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
[...]
        allow-query     { localhost; any; };
        allow-query-cache { localhost; any; };

        recursion yes;
[...]
# référence vers notre fichier de zone
zone "tp4.b1" IN {
     type master;
     file "tp4.b1.db";
     allow-update { none; };
     allow-query {any; };
};
# référence vers notre fichier de zone inverse
zone "1.4.10.in-addr.arpa" IN {
     type master;
     file "tp4.b1.rev";
     allow-update { none; };
     allow-query { any; };
};
```

➜ **Et pour les fichiers de zone**

```bash
# Fichier de zone pour nom -> IP

$ sudo cat /var/named/tp4.b1.db

$TTL 86400
@ IN SOA dns-server.tp4.b1. admin.tp4.b1. (
    2019061800 ;Serial
    3600 ;Refresh
    1800 ;Retry
    604800 ;Expire
    86400 ;Minimum TTL
)

; Infos sur le serveur DNS lui même (NS = NameServer)
@ IN NS dns-server.tp4.b1.

; Enregistrements DNS pour faire correspondre des noms à des IPs
dns-server IN A 10.4.1.201
node1      IN A 10.4.1.11
```

```bash
# Fichier de zone inverse pour IP -> nom

$ sudo cat /var/named/tp4.b1.rev

$TTL 86400
@ IN SOA dns-server.tp4.b1. admin.tp4.b1. (
    2019061800 ;Serial
    3600 ;Refresh
    1800 ;Retry
    604800 ;Expire
    86400 ;Minimum TTL
)

; Infos sur le serveur DNS lui même (NS = NameServer)
@ IN NS dns-server.tp4.b1.

;Reverse lookup for Name Server
201 IN PTR dns-server.tp4.b1.
11 IN PTR node1.tp4.b1.
```

➜ **Une fois ces 3 fichiers en place, démarrez le service DNS**

```bash
# Démarrez le service tout de suite
$ sudo systemctl start named

# Faire en sorte que le service démarre tout seul quand la VM s'allume
$ sudo systemctl enable named

# Obtenir des infos sur le service
$ sudo systemctl status named

# Obtenir des logs en cas de probème
$ sudo journalctl -xe -u named
```

🌞 **Dans le rendu, je veux**

- un `cat` des fichiers de conf

```
[hugoa@dns-server ~]$ sudo cat /etc/named.conf
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//

options {
        listen-on port 53 { 127.0.0.1; any; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { localhost; any; };
        allow-query-cache { localhost; any; };

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;

        dnssec-validation yes;

        managed-keys-directory "/var/named/dynamic";
        geoip-directory "/usr/share/GeoIP";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        /* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
        include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "tp4.b1" IN {
        type master;
        file "tp4.b1.db";
        allow-update { none; };
        allow-query { any; };
};

zone "1.4.10.in-addr.arpa" IN {
        type master;
        file "tp4.b1.rev";
        allow-update { none; };
        allow-query { any; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```
- un `systemctl status named` qui prouve que le service tourne bien

```
[hugoa@dns-server ~]$ sudo systemctl status named
● named.service - Berkeley Internet Name Domain (DNS)
     Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; vendor preset: disabled)
     Active: active (running) since Tue 2022-10-25 16:40:26 CEST; 8min ago
   Main PID: 11195 (named)
      Tasks: 5 (limit: 5907)
     Memory: 18.4M
        CPU: 30ms
     CGroup: /system.slice/named.service
             └─11195 /usr/sbin/named -u named -c /etc/named.conf

Oct 25 16:40:26 dns-server named[11195]: network unreachable resolving './DNSKEY/IN': 2001:5>
Oct 25 16:40:26 dns-server named[11195]: network unreachable resolving './NS/IN': 2001:500:9>
Oct 25 16:40:26 dns-server named[11195]: zone localhost.localdomain/IN: loaded serial 0
Oct 25 16:40:26 dns-server named[11195]: zone tp4.b1/IN: loaded serial 2019061800
Oct 25 16:40:26 dns-server named[11195]: zone 1.0.0.127.in-addr.arpa/IN: loaded serial 0
Oct 25 16:40:26 dns-server named[11195]: all zones loaded
Oct 25 16:40:26 dns-server systemd[1]: Started Berkeley Internet Name Domain (DNS).
Oct 25 16:40:26 dns-server named[11195]: running
Oct 25 16:40:26 dns-server named[11195]: managed-keys-zone: Initializing automatic trust anc>
Oct 25 16:40:26 dns-server named[11195]: resolver priming query complete
```
- une commande `ss` qui prouve que le service écoute bien sur un port

```
[hugoa@dns-server ~]$ sudo ss -alntp
State   Recv-Q  Send-Q    Local Address:Port     Peer Address:Port  Process
LISTEN  0       10           10.4.1.201:53            0.0.0.0:*      users:(("named",pid=11195,fd=21))
```

🌞 **Ouvrez le bon port dans le firewall**

- grâce à la commande `ss` vous devrez avoir repéré sur quel port tourne le service
  - vous l'avez écrit dans la conf aussi toute façon :) 
  C'est le port 53 \^o^/
- ouvrez ce port dans le firewall de la machine `dns-server.tp4.b1` (voir le mémo réseau Rocky)

```
[hugoa@dns-server ~]$ sudo firewall-cmd --add-port=53/tcp --permanent
success
[hugoa@dns-server ~]$ sudo firewall-cmd --reload
success
```

## 3. Test

🌞 **Sur la machine `node1.tp4.b1`**

- configurez la machine pour qu'elle utilise votre serveur DNS quand elle a besoin de résoudre des noms

```
[hugoa@node1 ~]$ sudo cat /etc/resolv.conf
nameserver 10.4.1.201
```
- assurez vous que vous pouvez :
  - résoudre des noms comme `node1.tp4.b1` et `dns-server.tp4.b1`

```
[hugoa@node1 ~]$ dig node1.tp4.b1

; <<>> DiG 9.16.23-RH <<>> node1.tp4.b1
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43964
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: ee6fdfbe2c6aed1f0100000063580224debadc57eca7e243 (good)
;; QUESTION SECTION:
;node1.tp4.b1.                  IN      A

;; ANSWER SECTION:
node1.tp4.b1.           86400   IN      A       10.4.1.11

;; Query time: 2 msec
;; SERVER: 10.4.1.201#53(10.4.1.201)
;; WHEN: Tue Oct 25 17:35:04 CEST 2022
;; MSG SIZE  rcvd: 85

[hugoa@node1 ~]$ ping node1.tp4.b1
PING node1.tp4.b1 (10.4.1.11) 56(84) bytes of data.
64 bytes from node1.tp4.b1 (10.4.1.11): icmp_seq=1 ttl=64 time=0.021 ms
64 bytes from node1.tp4.b1 (10.4.1.11): icmp_seq=2 ttl=64 time=0.072 ms
```

  - mais aussi des noms comme `www.google.com`

```
[hugoa@node1 ~]$ dig www.google.com

; <<>> DiG 9.16.23-RH <<>> www.google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20920
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 5ac3113f07688efd01000000635803c4c3427b71c4ce9d13 (good)
;; QUESTION SECTION:
;www.google.com.                        IN      A

;; ANSWER SECTION:
www.google.com.         8       IN      A       142.250.179.100

;; Query time: 1 msec
;; SERVER: 10.4.1.201#53(10.4.1.201)
;; WHEN: Tue Oct 25 17:42:00 CEST 2022
;; MSG SIZE  rcvd: 87

[hugoa@node1 ~]$ ping www.google.com
PING www.google.com (142.250.179.100) 56(84) bytes of data.
64 bytes from par21s20-in-f4.1e100.net (142.250.179.100): icmp_seq=1 ttl=113 time=25.2 ms
64 bytes from par21s20-in-f4.1e100.net (142.250.179.100): icmp_seq=2 ttl=113 time=27.0 ms
```

🌞 **Sur votre PC**

- utilisez une commande pour résoudre le nom `node1.tp4.b1` en utilisant `10.4.1.201` comme serveur DNS

```
PS C:\Users\hugoa> nslookup node1.tp4.b1 10.4.1.201
Serveur :   dns-server.tp4.b1
Address:  10.4.1.201

Nom :    node1.tp4.b1
Address:  10.4.1.11

```

> Le fait que votre serveur DNS puisse résoudre un nom comme `www.google.com`, ça s'appelle la récursivité et c'est activé avec la ligne `recursion yes;` dans le fichier de conf.

🦈 **Capture d'une requête DNS vers le nom `node1.tp4.b1` ainsi que la réponse**

[DNS](wireshark/response-dns.pcapng)