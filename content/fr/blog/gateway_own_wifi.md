---
author: "Alexis LEBEL"
title: "Mettre en place une passerelle pour contourner les restrictions de nombre d’appareils sur un réseau public"
date: 2020-12-01
tags: ["IoT", "wifi", "gateway", "privacy"]
#thumbnail: /images/2020/12/automate_1.jpg
---

Salut à tous ! Si vous avez déjà utilisé un réseau public, vous savez certainement de quoi je vais parler. Les réseaux publics sont souvent derrière ce que l’on appelle un “Portail Captif”, il s’agit souvent d’une page de connexion sur laquelle il faut se connecter afin d’accéder à Internet. Ces réseaux sont aussi souvent limités en nombre d’appareils que vous pouvez connecter avec votre compte. Eh bien dans cet article, nous allons mettre en place une passerelle, afin d’accéder au réseau public avec plusieurs appareils, à travers une seule interface.

Le matériel
Tout d’abord, vous aurez besoin du serveur qui gérera la passerelle. Je vous conseille de prendre un ordinateur monocarte, par exemple un Raspberry Pi. Personnellement, j’utiliserai un Orange Pi one, qui a l’avantage d’être bon marché, et d’avoir un port Ethernet et un port USB : on a pas besoin de plus.

Vous aurez aussi besoin d’une carte réseau WiFi USB, d’un petit câble Ethernet et d’un routeur. Ici, j’utiliserai le RTAC51U de chez Asus, car c’est un routeur que j’avais à disposition lorsque je faisait ce test.

Et voilà, rien de plus ! Passons à la pratique !

Se connecter à la passerelle
Afin de configurer la passerelle, connectez-là d’abord à un port local de votre routeur, afin d’y accéder en SSH. Sur les Raspberry Pi, il faut au préalable l’activer en branchant un écran et un clavier. Sur tout autre matériel tournant sous Armbian, le SSH est activé par défaut. Les identifiants par défaut sont root/1234 . Je vous conseille vivement de les changer à la première utilisation.

Configuration de la passerelle
Dans cette partie, je prendrai comme base, le tutoriel qui se trouve ici, mais adapté à notre cas.

Tout d’abord, nous allons nous connecter au WiFI. Il vous faudra donc faire la commande suivante :

nmtui

Vous vous retrouverez avec cette “fenêtre”. Il faut ensuite aller dans “Activer une connexion” puis vous connecter au réseau public de votre choix.

Ensuite, nous allons avoir besoin du nom de nos deux interfaces. Vous pouvez ainsi faire la commande suivante, et relever les interfaces qui ont un nom qui commence par “eth”, “ens”, “wlan”, ou encore “wlx”, cela dépend des systèmes.

ifconfig
Dans la suite de ce guide, nous prendrons comme base les interfaces eth0 et wlan0.

Nous allons ensuite devoir configurer un serveur dhcp, que nous allons installer :

sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install isc-dhcp-server iptables-persistent -y
Ensuite nous désactivons de DHCP pour l’interface Filaire :

sudo nano /etc/dhcpcd.conf
En ajoutant la ligne :

denyinterfaces eth0
Et en spécifiant une nouvelle adresse pour cette interface :

sudo nano /etc/network/interfaces
auto eth0
allow-hotplug eth0
iface eth0 inet static
    address 192.168.2.1
    netmask 255.255.255.0
    network 192.168.2.0
    broadcast 192.168.2.255
Maintenant il faut configurer le serveur DHCP, ce serveur n’aura pour utilité que de donner une adresse IP locale à notre routeur :

sudo cp /etc/dhcp/dhcpd.conf /etc/dhcp/dhcpd.conf.bak
sudo nano /etc/dhcp/dhcpd.conf
Nous allons modifier le fichier afin d’avoir les lignes suivantes :

# Sample /etc/dhcpd.conf
# (add your comments here) 
default-lease-time 600;
max-lease-time 7200;
option subnet-mask 255.255.255.0;
option broadcast-address 192.168.1.255;
option routers 192.168.1.254;
option domain-name-servers 192.168.1.1, 192.168.1.2;
option domain-name "undomaine.peuimporte";
option ntp-servers 192.168.1.254;

subnet 192.168.2.0 netmask 255.255.255.0 {
   range 192.168.2.2 192.168.2.50;
   option subnet-mask 255.255.255.0;
   option routers 192.168.2.1;
   option broadcast-address 192.168.2.255;
   default-lease-time 600;
   max-lease-time 7200;
} 
Cela permet de créer un réseau dont l’adresse de passerelle est 192.168.2.1, et dont les clients auront les adresses de 192.168.2.2 à 192.168.2.50 (ce qui est largement suffisant vu la tâche du serveur DHCP).

Enfin, il faut configurer le routage Filaire -> WiFi :

Nous allons d’abord activer cette fonction en allant dans le fichier /etc/sysctl.conf

sudo nano /etc/sysctl.conf
Il faut ensuite dé-commenter ceci en enlevant le # :

net.ipv4.ip_forward=1
Puis activer les modifications :

sudo sh -c "echo 1 > /proc/sys/net/ipv4/ip_forward"
Nous allons ensuite créer un script :

sudo nano iptablereset.sh
Dans lequel on va mettre

#!/bin/sh
echo "Resetting the IP Tables"
ipt="/sbin/iptables"
## Failsafe - die if /sbin/iptables not found [ ! -x "$ipt" ] && { echo "$0: \"${ipt}\" command not found."; exit 1; }
$ipt -P INPUT ACCEPT
$ipt -P FORWARD ACCEPT
$ipt -P OUTPUT ACCEPT
$ipt -F
$ipt -X
$ipt -t nat -F
$ipt -t nat -X
$ipt -t mangle -F
$ipt -t mangle -X
$ipt -t raw -F
$ipt -t raw -X
On enregistre…

sudo chmod +x iptablereset.sh
…on le rend exécutable…

sudo ./iptablereset.sh
… et on l’exécute !

Maintenant on va mettre nos nouvelle règles dans iptables, avec ces commandes :

sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -i wlan0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o wlan0 -j ACCEPT
On sauve :

sudo sh -c "iptables-save > /etc/iptables.ipv4.nat"
Puis on applique ces règles au démarrage. Pour ce faire, on modifie le rc.local :

sudo nano /etc/rc.local
Puis on rajoute cette ligne avant le exit 0

iptables-restore < /etc/iptables.ipv4.nat
Et voilà, votre passerelle est configurée pour redistribuer le réseau WiFI via le réseau filaire !

Configuration du routeur
Nous allons maintenant configurer le routeur. A partir de maintenant, vous pouvez brancher le câble réseau de la passerelle au port WAN ou Internet du routeur. Il devrais récupérer son adresse et être fonctionnel. Cependant, nous allons faire quelques modifications dans sa configuration. par conséquent, il faut se connecter à son interface d’administration. Par défaut, son adresse est 192.168.1.1

Tout d’abord, afin que les différents réseaux ne rentrent pas en conflit, nous allons changer les paramètres IP. Pour ma part cela ressemble à ça :



J’utilise ici un réseau 10.6.0.0 avec un masque en 255.255.255.0 qui me permet de connecter 253 machines.

Et pour le reste, c’est à votre guise (mot de passe wifi, points d’accès, dual WAN…)

Conclusion
Voilà, vous savez maintenant comment faire de la translation de connexion d’une interface à une autre, mais vous avez aussi votre propre réseau privé, dans un réseau public. Maintenant, pour des soucis de sécurité, je vous conseille vivement d’utiliser un VPN, puisque toutes les données vont quand même passer sur le réseau public. Bien que pratiquement tout internet est maintenant en HTTPS, certaines techniques notamment le Man In the Middle, permettent quand même d’intercepter du traffic non chiffré. Le routeur que j’utilise permet de mettre un VPN qui est utilisé de manière transparente pour tous les appareils connectés, ce qui n’est pas forcément une mauvaise idée si vous voulez mon avis !

Bonne bidouille !