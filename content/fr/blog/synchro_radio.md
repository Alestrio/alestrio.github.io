---
author: "Alexis LEBEL"
title: "Comment avoir une radio synchronisée sur plusieurs postes"
date: 2021-12-01
tags: ["synchro", "radio", "icecast"]
#thumbnail: /images/2020/12/automate_1.jpg
---

Si vous travaillez avec une web radio, sur plusieurs postes en même temps, ou dans plusieurs pièces différentes, vous vous êtes déjà sûrement confronté au problème de la radio qui a un décalage sur les différents postes. Si ce n’est pas le cas, eh bien moi oui ! Je vais du coup vous expliquer comment j’ai réussi à faire en sorte que la radio soit parfaitement synchronisée entre plusieurs postes/pièces.

Faire son propre serveur web radio (spoiler : c’est pas synchro, mais c’est intéressant)
Tout d’abord, comme dans la totalité de cet article, nous allons adopter une architecture client/serveur. Bien sûr, le client et le serveur peuvent être sur la même machine, mais pour un besoin de clarté, ces deux entités seront séparées. Le serveur que nous allons utiliser est une serveur Debian.

Ma première idée était de redistribuer le flux radio, en utilisant un serveur icecast2, ce qui permet d’éliminer la latence liée à l’utilisation de flux autres.

Ainsi, nous allons utiliser l’architecture suivante :


Tout d’abord, il faut installer le serveur :

sudo apt install icecast2
Puis nous éditons le fichier de configuration :

sudo nano /etc/icecast2/icecast.xml
Imaginons que nous voulons redistribuer la radio RTL en 128kbps, le fichier de configuration aura donc :

    <relay>
        <server>icecast.rtl.fr</server>
        <port>80</port>
        <mount>/rtl-1-44-128</mount>
        <local-mount>/rtl</local-mount>
        <on-demand>0</on-demand>
        <relay-shoutcast-metadata>0</relay-shoutcast-metadata>
    </relay>
Et voilà, il suffit de se connecter avec vlc à l’adresse http://ip.du.serveur.local:8000/rtl dans le cas où l’on a gardé la configuration par défaut du serveur icecast2.

Mais malheureusement, cette solution n’était pas la bonne, car latence il y avait..

Bien configurer VLC
J’ai donc essayé autre chose. En partant du fait que la latence était du tampon, eh bien j’ai enlevé le tampon !

Dans VLC, il faut se rendre dans les paramètres, puis cliquer sur “Tous”, aller dans Entrée/CODEC et on trouve l’option “Cache réseau”, que l’on passe à 0.

Mais évidement, ce n’était pas la solution, car toujours, latence il y avait. Mais après quelques recherches, je suis arrivé à une bien meilleure solution. Mais d’abord, un peu de théorie.

Théorie sur les réseaux IP
Les adresses IP sur Internet sont séparées en plusieurs classes comme ci-dessous :

CHAPITRE I : PREREQUIS POUR NOTRE ETUDES TECHNIQUE – Institut numerique

Les trois premières classes sont les classes d’IP qui sont attribuables sur Internet. Dans ces classes, on trouve des réseaux réservés :

Classe A
10.0.0.0/8 pour des grands réseaux locaux
127.0.0.0/8 pour les bouclages locaux (retour direct à la machine, ex: 127.0.0.1 équivalent à localhost)
Classe B
172.16.0.0/12 pour des réseaux locaux (maxi 172.31.255.255)
Classe C
192.168.0.0/16 pour des réseaux locaux (maxi 192.168.255.255)
Et maintenant la classe qui nous intéresse : la classe D, réservée au multicast.

Le multicast, c’est un type de communication qui va d’une machine, à plusieurs machines en même temps. Il s’oppose au traditionnel unicast (les classes A, B et C), qui correspond à une communication d’une seule machine à une seule autre.

Eh bien ce type d’adresse multicast, c’est exactement ce qu’il nous faut !

Ces adresses vont de 224.0.0.0 à 239.255.255.255, ce qui fait 268 435 456 adresses possibles. Nous allons taper dans le tas et prendre l’adresse 239.1.1.1 (parce que facile à retenir, et j’ai vu qu’aucun protocole ni programme n’utilise cette adresse)

Faire une diffusion en multicast du flux radio avec VLC
Bon maintenant qu’on a posé un peu les bases, passons à la pratique ! Nous allons utiliser l’architecture ci-dessous :


Tout d’abord, on va installer VLC sur notre serveur :

sudo apt install vlc
Puis nous allons démarrer une diffusion du flux de RTL en 128kbps, sur l’adresse multicast 239.1.1.1 :

cvlc -vvv http://streaming.radio.rtl.fr/rtl-1-44-128 --sout '#rtp{dst=239.1.1.1,port=5004,mux=ts}'
Bon, expliquons cette commande un peu barbare :

cvlc est le nom de l’exécutable de VLC en ligne de commande
l’option vvv permet de dire que l’on veut faire une diffusion
l’adresse du flux source
la description du flux de destination :
#rtp qui est le protocole "Real Time Protocole" utilisé pour de la diffusion audio et vidéo
dst : notre adresse de destination
port : le port de destination. Ici 5004, qui est le port par défaut du protocole RTP
mux : dans le cas où de la vidéo est diffusée en même temps (pas notre cas ici)
Maintenant, si l’on veut faire cette diffusion au démarrage du serveur, il faut :

Aller dans le fichier de démarrage :

sudo nano /etc/rc.local
Mettre la commande écrite ci-dessus au dessus du exit 0 dans le fichier, sauver, et quitter.

Et maintenant, il faut tester !

Jouer la diffusion sur un poste client
Pour cela deux solutions : la solution propre, et la solution de test.

La solution test
Pour tester votre flux, il suffit, sur vlc, d’aller dans “Média”, puis “Ouvrir un flux réseau…”, et dans la fenêtre qui s’ouvre, mettre l’adresse qui nous intéresse, précédée de son protocole : rtp://239.1.1.1. Et voilà, si tout est bien fait, le flux est joué, et si vous le jouez sur un autre poste, il est parfaitement synchronisé !

La solution propre
Pour un déploiement en situation réelle, pour une utilisation quotidienne simple, il suffit de créer un fichier texte avec l’extension .m3u, avec pour contenu :

rtp://239.1.1.1
Plus qu’à enregistrer, et votre fichier pour jouer la diffusion est prêt !

Conclusion
Pour conclure, si vous voulez diffuser une web radio, qui soit parfaitement synchronisée sur tous les postes, il vous faudra régler le cache réseau de VLC à 0, et créer une diffusion du flux internet sur votre réseau local, vers une adresse multicast. J’espère avoir été suffisamment clair, si vous avez des questions, n’hésitez pas à poster un commentaire, j’y répondrai au plus vite !

Bonne bidouille !