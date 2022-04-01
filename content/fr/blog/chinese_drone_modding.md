---
author: "Alexis LEBEL"
title: "Modding d’un drone chinois (Eachine E58)"
date: 2020-08-02
tags: ["chinois", "drone", "modding"]
thumbnail: /images/2020/02/eachine.jpg
---

J’ai récemment fait l’acquisition d’un petit drone chinois à 60€. J’en suis plutôt content, la qualité vidéo est correcte (1080p), il va plutôt haut, il est rapide, et il y a moyen de s’amuser à faire des figures. Néanmoins, il y a quelques points négatifs. En effet, tout d’abord, même une fois bien trimé, ce drone a toujours tendance à se déplacer tout seul, et son autonomie est vraiment faible.
Jusque là, je m’y était fait, et cela ne me posais pas vraiment de problèmes : j’arrivais toujours à me débrouiller pour faire quelques plans sympa.

# L’élément déclencheur

Et un jour (le drama !), par curiosité, je lance un petit scan wifi sur le réseau créé par le drone. Il s’avère qu’il fonctionne sur une puce Espressif ! Il se trouve que je bidouille beaucoup avec des ESP32 et 8266, du coup je suis un peu familier avec ces technos.

Alors pour continuer dans mon insatiable curiosité, je démonte le drone pour m’apercevoir qu’il s’agissait d’un microcontrôleur, mais que je ne connaissais pas. D’ailleurs, plutôt difficile à reprogrammer, surtout parce que le firmware de ce drone n’est pas fourni par le fabricant.

# L’analyse des trames

Du coup, pour trouver comment fonctionne ce drone, j’ai lancé mon analyseur de packets, et je me suis connecté au réseau du drone. Il s’avère donc que l’application envoie et reçois trois types de packets.

- Sur le port 5000 : Commandes de contrôle du drone
- Sur le port 7060 : Requête et envoi de la vidéo
- Sur le port 8060 : Vidéo aussi

J’ai ensuite analysé les packets pour découvrir que l’application envoie une commande : “lewei_cmd”. Une petite recherche sur Google m’a permis de trouver cet article de blog : https://blog.horner.tj/hacking-chinese-drones-for-fun-and-no-profit/ qui m’a beaucoup servi. Petit TL;DR ci dessous :

```
1st byte – Header: 66 \
2nd byte – Left/right movement (0-254, with 128 being neutral) \
3rd byte – Forward/backward movement (0-254, with 128 being neutral) \
4th byte – Throttle (elevation) (0-254, with 128 being neutral) \
5th byte – Turning movement (0-254, with 128 being neutral) \
6th byte – Reserved for commands (0 = no command) \
7th byte – Checksum (XOR of bytes 2, 3, 4, and 5) \
8th byte – Footer: 99
```

blog.horner.tj

En plus, cette personne à créé une bibliothèque en node.js pour contrôler ce type de drone qui marche à merveille !

# Il est temps de modder

Plein d’idées de bidouilles en tout genre, je suis parti faire quelques achats sur Aliexpress. L’objectif ? Un module accéléromètre pour corriger les problèmes de mouvement incontrôlés du drone. J’avais déjà une idée de la carte de commande, une petite carte que javais déjà en ma possession : une WEMOS D1 Mini. Ensuite, il fallait penser à l’alimentation. J’ai donc acheté un shield pour batterie Li-Po, et une batterie Li-Po (sans blague) de 300mAh. Après un énorme calcul savant (une division), j’en ai conclu qu’une batterie de ce genre pouvait alimenter la carte de contrôle plus de 2000h à puissance max (sans accéléromètre). enfin, j’ai acheté un petit accéléromètre tout simple mais assez précis. (spoiler : pas assez :p)

L’idée était assez simple, installer un petit module qui détecte le mouvement du drone et envoyer ces informations au serveur node.js qui corrigera le mouvement. Forcément, la commande devra aussi se faire via le serveur, de façon à ce que celui-ci ne corrige pas le mouvement que que je demande au drone.

Après quelques manips, je me suis rendu compte que la D1 mini n’avait qu’un seul port analogique.. C’est balot quand il en faut trois pour utiliser l’accéléromètre. D’ailleurs l’accéléromètre, il fonctionne très bien, mais pour les grosse accélérations ou pour les accélérations statiques. Du coup, pour les toutes petites accélérations du drone, c’est un peu light en terme de sensibilité. Et là deux solutions :

Ouvrir le drone et récupérer le signal de l’accéléromètre inclus dans la carte électronique, autant dire que si le module est soudé en SMD, c’est mort.
Racheter un accéléromètre plus sensible
**Conclusion** : ben là, je suis bloqué, mais je travaille toujours dessus ! Du coup, cet article est split en deux parties (la deuxième sera peut être mise dans loooooontemps) afin que je puisse publier cette partie ! 😀

# Conclusion
Pour conclure, j’ai déjà bien avancé sur ce projet, la partie commande est déjà assez bien comprise et la pratique n’est qu’une question de temps. D’ailleurs, pendant que je faisait mes manips, je me suis dit que passer par un serveur externe pour faire tout ça était assez lourd, du coup, tout le traitement devrait se faire sur le microcontrôleur, ce qui, d’emblée, éliminerait un élément de la chaine de contrôle ! Bref, projet à suivre ! Bonne bidouille à tous ! 😀