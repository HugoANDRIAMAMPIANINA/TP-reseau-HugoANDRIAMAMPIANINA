# MITM et DNS Spoofing


Lien du dépot GIT avec le code Python :
https://github.com/HugoANDRIAMAMPIANINA/MITM_Hugo

Pour récupérer le script python :

- faire un `git clone https://github.com/HugoANDRIAMAMPIANINA/MITM_Hugo` **sur la machine `hacker`**

- installer les bibliothèques `scapy` et `netfilterqueue`
  

**ATTENTION !!!**

Il faut activer l'IPv4 Forwarding (toujours sur le hacker)

```
sysctl -w net.ipv4.ip_forward=1
```

et taper : 

```
iptables -I FORWARD -j NFQUEUE --queue-num 0
```
Cette commande va permettre de stoquer toutes les requetes DNS dans une file d'attente afin de les modifier

- Puis lancer le programme (normalement ça devrait marcher)

```
sudo python3 poison.py
```

Petit truc :

y'a possibilité de rajouter des noms de domaine comme `facebook.com` et une IP pour rediriger la requête vers cette IP, j'ai mis un petit site sympa pour ceux qui ping google.com :grin: