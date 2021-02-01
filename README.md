# Les bases du réseau

## Étape 1

### Réalisation d'un schéma du réseau :

J'ai réalisé, premièrement, un schéma représentant mon réseau une fois la configuration de chaque élément réalisé. Il est accessible dans ce même répertoire sous le nom de **schema.jpg**.

### Création de la VM template :

J'ai créé une machine virtuelle type, qui me servira quand j'aurais besoin de créer de nouveaux serveurs/clients pour les intégrer à mon réseau.

Cette dernière à comme caractéristiques :

- **Hostname :** Template
- **CPU :** 1 Cœur
- **RAM :** 2 Go
- **OS :** Debian Buster
- **Carte réseau :** Bridge

J'ai ensuite installé Debian sans GUI, en n'incluant que les collections "SSH" et "standard system utilities".

A la fin de l'installation, j'ai finalement changé le réglage de la carte réseau en "Host Only".

## Étape 2

### Préparation des VMs de travail :

J'ai ensuite cloné ma VM template en 4 autres machines virtuelles : *Gateway* (qui sera configuré avec une deuxième carte réseau, réglée ici en Bridge), *Manager*, *Web* et *Client*.

Je me suis connecté à chaque machine en SSH afin de modifier le hostname de chaque système, le faisant correspondre avec leurs utilité. Pour cela, j'ai utilisé la commande :

```
hostnamectl set-hostname <hostname-désiré>.res1.local
```

En remplaçant *<hostname-désiré>* par la fonction de la machine.

## Étape 3

### Préparation des adaptateurs réseau :

Sur la VM *gateway*, j'ai configuré la première carte réseau en *host-only*, et la deuxième en *bridge*. Je vais donc éditer le fichier de configuration des interfaces dans **/etc/network/interfaces**.

Ce dernier ressemble à ça lors de la première connexion :

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug enp0s3
iface enp0s3 inet dhcp
```

Je vais donc modifier la configuration de l'interface **enp0s3** en lui donnant une addresse IP *statique*, et initialiser la deuxième interface réseau (**enp0s8**).

Le fichier ressemblera donc au code ci-dessous après les modifications :

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface : host-only
allow-hotplug enp0s3
iface enp0s3 inet static
    address 10.242.0.1
    netmask 255.255.0.0

# The secondary network interface : bridge
allow-hotplug enp0s8
iface enp0s8 inet dhcp
```

Ensuite, je configure l'interface réseau de chaque VM afin de leurs donner, à elles-aussi, des adresses IP statiques.

Par exemples pour *manager*, j'ai choisi l'adresse **10.242.0.2** :

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface : host-only
allow-hotplug enp0s3
iface enp0s3 inet static
    address 10.242.0.2
    netmask 255.255.0.0
    gateway 10.242.0.1
```

On peut remarquer que j'ai ajouté un **gateway** à cette configuration, avec l'adresse ip de ma VM *gateway*.

A ce stade, nous pouvons ping chaque machine virtuelle depuis n'importe laquelle, mais n'acceder à internet que depuis *gateway*.

## Étape 4

### Configuration du *port-forwarding* :

Pour que les VMs puissent avoir accès à internet via *gateway*, il faut rajouter une option à l'**iptables** de la VM *gateway*.

Nous devons d'abord configurer le port-forwarding, en modifiant le fichier **/etc/sysctl.conf** et en retirant le commentaire devant la ligne :

```
net.ipv4.ip_forward = 1
```

On ne le fait que sur la *gateway*. On redémarre notre VM, et on vérifie que les changements ont bien été pris en compte avec :

```
sysctl -p
```

### Configuration des *iptables* :

Nous allons donc mettre en place une règle pour permettre aux autres VM de se connecter à internet via la *gateway*.

Il existe 2 méthodes pour le faire : avec *SNAT*, et avec *MASQUERADE*. Ici, nous utiliserons **MASQUERADE**, car notre carte bridge est configurée avec une adresse IP dynamique (DHCP).

On teste donc avec cette commande :

```
iptables -t nat -A POSTROUTING ! -d 10.242.0.0/16 -o enp0s8 -j MASQUERADE
```

Ici, `enp0s8` est ma carte réseau bridge, et `10.242.0.0/16` la plage réseau où je souhaite rediriger le connexion internet.

Sur la VM *web*, je teste donc la commande :

```
ping google.com
```

Et j'obtiens :

```
PING google.com (216.58.213.78) 56(84) bytes of data.
64 bytes from lhr25s01-in-f14.1e100.net (216.58.213.78): icmp_seq=1 ttl=117 time=6.27 ms
64 bytes from lhr25s01-in-f14.1e100.net (216.58.213.78): icmp_seq=2 ttl=117 time=10.5 ms
64 bytes from lhr25s01-in-f14.1e100.net (216.58.213.78): icmp_seq=3 ttl=117 time=8.46 ms
64 bytes from lhr25s01-in-f14.1e100.net (216.58.213.78): icmp_seq=4 ttl=117 time=6.77 ms
64 bytes from lhr25s01-in-f14.1e100.net (216.58.213.78): icmp_seq=5 ttl=117 time=5.97 ms
64 bytes from lhr25s01-in-f14.1e100.net (216.58.213.78): icmp_seq=6 ttl=117 time=10.2 ms
^C
--- google.com ping statistics ---
6 packets transmitted, 6 received, 0% packet loss, time 12ms
rtt min/avg/max/mdev = 5.970/8.026/10.502/1.821 ms
```

PS : pour que google.com soit bien reconnue, j'ai dû modifier le fichier **/etc/resolv.conf** en entrant :

```
nameserver 8.8.8.8
```

Ensuite, la commande marche parfaitement.

### Faire en sorte que la règle soit persistente :

Le problème avec la règle vue ci-dessus, c'est qu'elle doit être rentrée à chaque redémarrage de la machine. Nous devons donc faire en sorte qu'elle soit toujours en vigueur.

Pour cela, j'installe *iptables-persistent*. Ce dernier va me permettre de sauvegarder mes configuration dans deux fichiers dinstincts, et de les charger au boot de la machine.

> Une autre méthode aurait été d'inclure cette règle dans un script à lancer à chaque démarrage.

Ensuite, une fois la règle établie, on lance la commande :

```
iptables-save > /etc/iptables/rules.v4
```

Afin de sauvegarder la (ou les) règle désirée. Elle sera donc automatiquement appliquée au démarrage de la machine, permettant donc de la rendre persistente.

## Étape 5

### Installer un serveur DHCP :

Sur la VM *manager*, nous allons installer un *serveur DHCP* afin de pouvoir gérer les clients qui se connectent au réseau. Les dix premières adresses sont réservées aux divers équipements (serveurs, etc), tandis que les 27 suivantes seront réservées aux machines fixes.

Notre DHCP assignera alors des adresses IP allant de **10.242.0.38** à **10.242.255.254**.

Pour installer notre serveur, on rentre la commande suivante :

```
apt install isc-dhcp-server
```

Et on remarque tout de suite qu'il échoue au lancement. Nous allons donc devoir modifier sa configuration.

### Configuration du serveur DHCP :

On se rend dans le fichier **/etc/dhcp/dhcpd.conf** afin de modifier les paramètres de notre serveur.

Le fichier, quand nous l'ouvrons, ressemble à l'exemple ci-dessous :

```
# dhcpd.conf
#
# Sample configuration file for ISC dhcpd
#

# option definitions common to all supported networks...
option domain-name "example.org";
option domain-name-servers ns1.example.org, ns2.example.org;

default-lease-time 600;
max-lease-time 7200;

# The ddns-updates-style parameter controls whether or not the server will
# attempt to do a DNS update when a lease is confirmed. We default to the
# behavior of the version 2 packages ('none', since DHCP v2 didn't
# have support for DDNS.)
ddns-update-style none;

# If this DHCP server is the official DHCP server for the local
# network, the authoritative directive should be uncommented.
#authoritative;

# Use this to send dhcp log messages to a different log file (you also
# have to hack syslog.conf to complete the redirection).
#log-facility local7;

# No service will be given on this subnet, but declaring it helps the
# DHCP server to understand the network topology.

#subnet 10.152.187.0 netmask 255.255.255.0 {
#}
```

Nous allons effectuer quelques modifications :

+ Nous ajoutons un commentaire à la ligne `option domain-name "example.org";`
+ On remplace `ns1.example.org, ns2.example.org;` par `8.8.8.8, 8.8.4.4` car nous n'avons pas encore notre propre DNS.
+ On retire le commentaire à la ligne `#authoritative;` car on souhaite que ce soit le seul server SHCP sur notre réseau.

Et on insère ensuite notre sous-réseau :

```
subnet 10.242.0.0 netmask 255.255.0.0 {
	option routers 10.242.0.1;
	range 10.242.0.38 10.242.255.254;
}
```

Ce qui nous donne ce fichier de configuration :

```
# dhcpd.conf
#
# Sample configuration file for ISC dhcpd
#

# option definitions common to all supported networks...
# option domain-name "example.org";
option domain-name-servers 8.8.8.8;

default-lease-time 600;
max-lease-time 7200;

# The ddns-updates-style parameter controls whether or not the server will
# attempt to do a DNS update when a lease is confirmed. We default to the
# behavior of the version 2 packages ('none', since DHCP v2 didn't
# have support for DDNS.)
ddns-update-style none;

# If this DHCP server is the official DHCP server for the local
# network, the authoritative directive should be uncommented.
authoritative;

# Use this to send dhcp log messages to a different log file (you also
# have to hack syslog.conf to complete the redirection).
#log-facility local7;

# No service will be given on this subnet, but declaring it helps the
# DHCP server to understand the network topology.

subnet 10.242.0.0 netmask 255.255.0.0 {         # On règle notre IP et notre masque de sous-réseau
	option routers 10.242.0.1;                  # On donne l'adresse IP de notre gateway
	range 10.242.0.38 10.242.255.254;           # On définit enfin notre interval. Vu que notre réseau est 10.242.0.0/16, il y aura approximativement 65534 hôtes présent au max.
}
```


### Test sur un client :

On configure l'interface de la VM *client* en DHCP, en prenant soin de bien avoir désactivé le DHCP dans VirtualBox. On lance ensuite la commande `ip a` :

```
nico@client:~$ ip a

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:ef:17:32 brd ff:ff:ff:ff:ff:ff
    inet 10.242.0.38/16 brd 10.242.255.255 scope global dynamic enp0s3
       valid_lft 558sec preferred_lft 558sec
    inet6 fe80::a00:27ff:feef:1732/64 scope link
       valid_lft forever preferred_lft forever

```

L'adresse IP de **enp0s3** est bien comprise dans la range désirée. De plus, l'accès internet est fonctionnel avec la commande `ping google.com`.


## Étape 6

Nous allons mettre en place le SSH de sorte à ce que, depuis *gateway*, on puisse accéder à n'importe quelle machine qui possède la clé SSH.

### Génération de la clé SSH :

On rentre la commande suivante dans la VM *gateway*

```
ssh-keygen -t ed25519 -C "username"
```

On remplace *username* par notre nom d'utilisateur sur la machine.

Une fois crées, les clés se trouvent dans **~/.ssh/id_ed25519** et **~/.ssh/id_ed25519.pub**. Nous n'allons utiliser que celle inscrite dans **~/.ssh/id_ed25519.pub**.

### Configuration dans les autre VM :

On se connecte maintenant sur une autre machine, à laquelle on désire avoir accès sans mot de passe.

Si le dossier **~/.ssh** n'existe pas, nous pouvons le créer avec `mkdir .ssh`.

On se rend ensuite dans ce répertoire et on crée un fichier **authorized_keys**.

On l'ouvre avec un éditeur de texte (ex. `emacs`) et on rentre la clé de notre VM *gateway*.

Exemple, pour ma VM *manager* :

```
nico@manager:~$ cat .ssh/authorized_keys

ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICTSIgmTzZNfWXcyLrVNmQWg6nz1+EoBMizK3zHi25Ep nico
```

On se déconnecte ensuite de la machine pour tester.

### Test du SSH :

Avant nos modifications, un mot de passe était nécessaire :

```
nico@gateway:~/.ssh$ ssh 10.242.0.2         # L'adresse de ma VM manager

nico@10.242.0.2's password:
Linux manager.res1.local 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Jan 24 07:00:51 2021 from 10.242.0.1

nico@manager:~$

```

Une fois nos modifications effectuées, on peut directement se connecter sans soucis :

```
nico@gateway:~/.ssh$ ssh 10.242.0.2

Linux manager.res1.local 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Jan 24 07:46:31 2021 from 10.242.0.1

nico@manager:~$

```

## Étape 7 : Local DNS: dnsmasq

### Installation :

Pour l'installation d'un **DNS local**, je vais utiliser **dnsmasq** ([lien](http://www.thekelleys.org.uk/dnsmasq/doc.html)) afin de configurer la résolution de différents hostname.

Je commence par cloner le GIT qui se trouve sur le site, et je lance `make install` afin d'installer **dnsmasq**.

### Configuration :

J'ouvre le fichier **/etc/dnsmasq.conf** et ajoute/modifie les lignes suivantes :

```
domain-needed       # Evite de renvoyer une résolution de nom incorrecte.
bogus-priv

expand-hosts        # Permet d'ajouter automatiquement le nom de domaine derrière le hostname.
domain=res1.local   # Nom du domaine

server=8.8.8.8      # On donne le DNS de base, afin de pouvoir y accéder une fois la configuration du DNS local terminée.

interface=enp0s3    # On donne l'interface où écouter les différentes requests
```

Ensuite, on édite le fichier **/etc/hosts** :

```
127.0.0.1   localhost
10.242.0.1  gateway
10.242.0.2  manager
10.242.0.3  web
```

Et enfin le fichier **/etc/resolv.conf** :

```
nameserver 127.0.0.1
```

Nous avons donc mis en place un **DNS local** permettant la résolution des *hostnames*. Il nous suffit de rentrer par exemple `ssh manager.res1.local` ou plus simplement `ssh manager` afin d'accéder à la machine désirée.

Pour pouvoir utiliser ce DNS sur toutes les machines du réseau, il faut modifier le fichier **/etc/resolv.conf** et rentrer l'IP de la VM *manager* :

```
nameserver 10.242.0.2   # Adresse IP de manager
```

### Entrées A et AAAA :

Une entrée **AAAA** traduit un nom de domaine vers une adresse **IPv6**, et une entrée **A** pointe un nom de domaine vers une adresse **IPv4**. Dans notre cas, nous utilisons une entrée A car nous sommes en IPv4.

### Entrée CNAME :

L'entrée **CNAME** (*Canonical Name*) désigne le nom de domaine par défaut.

## Étape 8 : Nginx

### Installation de Nginx

Nous allons installer Nginx sur la VM *web*. On lance donc la commande `sudo apt install nginx`.

On crée aussi un fichier **index.html** contenant le texte `Hello from Nginx !` qu'on place dans un dossier qu'on crée à l'adresse */var/www/web.res1.local/html/*.

### Configuration de Nginx

On met en place un *virtual host* web.res1.local qui va rediriger les requêtes vers notre fichier **index.html**.

Le terme *virtual host* est un emprunt d'Apache, en réalité, Nginx met en place des **Server Blocks** ; un exemple existe à l'adresse */etc/nginx/sites-available/default*.

On rentre donc la commande suivante pour créer un *block* :

```
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/web.res1.local

```

On l'édite ensuite afin de rentrer la configuration suivante :

```
server {
        listen 80;
        listen [::]:80;

        root /var/www/web.res1.local/html;
        index index.html index.htm index.nginx-debian.html;

        server_name web.res1.local;

        location / {
                try_files $uri $uri/ =404;
        }
}
```

Et on l'ajoute aux serveurs actuellement en ligne :

```
sudo ln -s /etc/nginx/sites-available/web.res1.local /etc/nginx/sites-enabled/
```

On édite ensuite le fichier */etc/nginx/ngnix.conf* pour retirer le commentaire de la ligne `server_names_hash_bucket_size 64;` afin d'éviter des problèmes lord de la résolution du domaine.

Enfin, on redémarre notre service *Nginx* afin de tester notre configuration avec `sudo systemctl restart nginx`.

### Test de Nginx

On teste d'abord si *Nginx* marche correctement :

```
nico@web: ~ $ curl -I localhost

HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Sun, 24 Jan 2021 21:57:32 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Sun, 24 Jan 2021 20:45:31 GMT
Connection: keep-alive
ETag: "600ddc6b-264"
Accept-Ranges: bytes
```

Et on teste ensuite si le bon fichier est renvoyé :

```
nico@web: ~ $ curl web.res1.local

Hello from Nginx !

```

### Explications

La directive `server_name` sert à définir le nom de ce serveur.

Si avec `curl`, par exemple, on rentre un nom de domaine qui ne correspond à aucun de ceux activés, alors on sera redirigé vers le serveur par défaut.

### Configuration des logs

Pour configurer les logs de notre *virtual host*, il faut ajouter une ligne dans le fichier **/etc/nginx/sites-availables/web.res1.html** :

```
access_log /var/log/nginx-res1.log;
```

Nos logs seront donc enregistrés à l'adresse : **/var/log/nginx-res1.log**.

### Désactivation des mises à jour

Pour désactiver la mise à jour automatique de *Nginx* par `apt`, il faut rentrer la commande suivante :

```
sudo apt-mark hold nginx
```

> Une autre méthode consisterait à modifier le fichier présent à l'adresse suivante : **/etc/apt/preferences.d/official-package-repositories.pref**
