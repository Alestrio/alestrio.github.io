---
author: "Alexis LEBEL"
title: "Homelab - How it started, how it's going"
date: 2025-08-22
tags: ["homelab", "proxmox", "fortigate", "sdn", "bgp", "ipsec"]
thumbnail: /images/2025/08/homelab_background.jpg
---

# Avant-propos

Cet article a été écrit sans IA. C'est une règle que je me fais sur ce blog. Je n'ai aucune pression de publication à cet endroit, pas de rémunération, juste une volonté de partager ce que je découvre, fais, conçois.

Ainsi, ce blog est un des derniers bastions où le texte que vous lisez vient d'un humain, avec ses qualités, son style, et ses défauts. Le style n'est pas lissé et puisque l'article est écrit sur plusieurs jours, on peut en trouver des variations.

À travers ces pages, je souhaite vous emmener dans mon univers de **homelabbing** : bricolages réseau, auto-hébergement, virtualisation, domotique, et toutes ces petites expérimentations qui transforment mon quotidien. L’idée est simple : montrer comment, avec quelques équipements et beaucoup de curiosité, on peut apprendre énormément, gagner en autonomie, économiser quelques euros, et surtout… s’amuser.

J'espère que vous prendrez autant de plaisir à me lire que moi à écrire ce billet,

Alexis

***

# Acronymes et Jargon

* FGT - Fortigate : Le pare-feu/routeur de l'installation
* FGN - Federated Git Network : Un nom un peu pompeux pour parler de l'infrasctructure entre les réseaux locaux d'amis
* SDN - Software Defined Network : Réseau où l’intelligence est centralisée dans un logiciel, ce qui le rend plus flexible et facile à gérer.
* LVM - Logical Volume Manager : système de gestion de l’espace disque, reposant sur le stockage bas niveau (disque, RAID), qui permet de manipuler des volumes de façon flexible comme une « boîte à tiroirs » que l’on peut agrandir, réduire ou déplacer.
* RAID - Redundant Array of Independent Disks : Technique qui combine plusieurs disques physiques pour améliorer la **sécurité des données** (redondance), la **performance**, ou les deux, en fonction du niveau choisi.
* iSCSI - Internet Small Computer System Interface : protocole qui permet de transporter des commandes de disque dur via le réseau IP, donnant l’illusion qu’un stockage distant est branché localement à la machine.
* BGP - Border Gateway Protocol : protocole qui permet aux différents réseaux d’Internet de s’échanger leurs « cartes routières » pour décider par où faire passer les données.
* IPSec - Internet Protocol Security :  protocole qui crée un tunnel sécurisé entre deux points réseau, chiffrant et authentifiant tous les paquets IP qui le traversent.
* IKEv2 - Internet Key Exchange version 2 : protocole qui négocie et établit les tunnels IPSec, en authentifiant les parties et en échangeant les clés de chiffrement de manière sécurisée.
* LACP - Link Aggregation Control Protocol : protocole qui combine plusieurs liaisons réseau physiques en un seul lien logique pour augmenter la bande passante et assurer la redondance.
* Redistribute connected - commande de routage : injecte dans un protocole de routage toutes les routes correspondant aux interfaces directement connectées du routeur, pour les rendre visibles aux autres routeurs.
* Redistribute static - commande de routage : injecte dans un protocole de routage toutes les routes statiques configurées sur le routeur, pour les rendre visibles aux autres routeurs.
* Voix et quorum - Proxmox : mécanismes de gestion du cluster où chaque nœud a une “voix” pour voter, et le quorum définit le nombre minimum de votes nécessaires pour que le cluster fonctionne correctement et évite le split-brain, c’est-à-dire que des nœuds isolés ne continuent pas à fonctionner indépendamment en provoquant des incohérences de données ou des conflits.
* Hyperviseur : logiciel qui permet de créer et gérer plusieurs machines virtuelles ou conteneurs sur un seul serveur physique, en isolant chaque VM et en partageant les ressources matérielles.
* SSO - Single Sign-On : mécanisme d’authentification qui permet à un utilisateur de se connecter une seule fois pour accéder à plusieurs applications ou services sans ressaisir ses identifiants.
* Cluster : ensemble de plusieurs serveurs ou nœuds qui travaillent ensemble comme une seule unité pour améliorer la disponibilité, la performance ou la tolérance aux pannes.

***

Pourquoi payer des abonnements cloud quand on peut héberger ses propres services ? Cette question, je me la suis posée il y a quelques années, et elle a complètement changé ma façon d'appréhender l'informatique personnelle.

Cet article se propose de retracer mon chemin dans le monde du homelabbing, dans un ordre chronologique, mais aussi logique afin d'être fluide à la lecture.

Tout a commencé quand je me suis retrouvé à mon compte, avec ma maison. Pendant mes années à l'UTC, et un peu après, j'ai loué une maison passoir thermique, chauffée à l'électricité. Le coût prohibitif de l'électricité (plus les crises qui l'ont d'autant fait monter) m'ont fait réfléchir à la façon d'optimiser le chauffage pour économiser.

Ainsi, me voici avec mon Orange Pi Win Plus, un dongle ZigBee, un capteur de température, et des contrôleurs fil-pilote pour les chauffage. La concept était simple, en utilisant Home Assistant, je pilotais une température de consigne qui dépendais de ma présence dans la maison, et un contrôleur simple PID se chargeais d'atteindre cette température de consigne. J'ai ainsi amélioré mon confort thermique tout en économisant environ 300€ par ans (par rapport aux anciens locataires). Sur une facture de 2000€, c'est honorable. Et très vite, j'ai été pris par le virus.

!["Une vue synthétique de ma consommation électrique cette année"](/images/2025/08/iBUNiMqIJywwC9_mUf0Nh7lPDlRxGH3YIxeFwx66oso=.jpeg)

Self note : je devrais en faire un article plus détaillé !

# Mon premier homelab

## HomeAssistant + ZigBee

Très vite, je me suis mis à beaucoup bidouiller avec HomeAssistant, à automatiser la plupart de ma vie en utilisant des capteurs et des actionneurs. Par exemple, voici quelques exemples d'automatisations qui m'ont littéralement changé la vie :

* Le seul interrupteur de la lumière du Salon se situait près de la porte d'entrée, à l'opposé de l'escalier qui me sert à monter dans ma chambre. Ainsi, avec un interrupteur ZigBee et un bouton ZigBee, j'ai pu faire une système de va-et-vient qui m'évite de marcher dans le noir pour éteindre la lumière. Cela me permet aussi d'éteindre la lumière automatiquement quand je pars de chez moi et qu'elle est restée allumée.
* Lorsque j'éteins mon ordinateur sur mon bureau, des voyants et l'imprimante restent allumés, ce qui est assez désagréable la nuit. J'ai donc ajouté une prise connectée qui éteint la multiprise du bureau quand l'ordinateur s'éteint et qui l'allume quand l'ordinateur s'allume (c'est un portable, donc avec une batterie). Cela permet aussi d'économiser un 10W continu, ce qui n'est pas négligeable !
* Une alarme ! C'est bête, mais rassurant quand on part loin de la maison. L'alarme utilise une combinaison d'une caméra, de capteurs de présence ZigBee, de capteurs d'ouverture de porte ZigBee, et d'alarmes sonores (Alexa toutes en même temps, alarme ZigBee 110dB, alarme extérieur ZigBee). Lorsqu'elle se déclenche, cela m'envoie aussi une notification sur mon téléphone avec une photo, et une option pour la désactiver en cas de fausse alerte (ce qui arrive relativement peu pour une alarme DIY !)

Je vais être franc, ayant invité des amis chez moi avec toutes ces automatisations m'a valu quelques moqueries, mais franchement, je trouve que ça n'a pas de prix ! La combinaison de toutes ces petites automatisations (j'en ai une 50aine) me permet de gagner un temps fou, et d'économiser sur mon nombre de décisions quotidiennes (qui, comme pour tout le monde, est limité, mais ce billet n'est pas un billet de sciences comportementales !).

## Un DNS bloqueur de pubs

J'ai aussi profité de cette petite plateforme pour mettre en place un PiHole. Ce petit container docker permet de faire du filtrage de pubs et de trackers au niveau DNS (c'est qui est un début, mais pas suffisant pour tout bloquer !) ça m'a aussi permis de bloquer la résolution des petits devices IoT Chinois (Tuya) qui veulent toujours contacter un serveur chinois, même quand ils ne sont pas utilisés.

Pour les résolutions, j'ai mis les DNS de Cloudflare en version Family, ce qui permet une résolution rapide tout en bloquant une autre partie des trackers, contenus malveillants, etc.

## Une tentative de remplacer Netflix + une sanction

Je ne suis pas un gros consommateur de plateformes de streaming. En fait, je me lasse très facilement des séries et films d'aujourd'hui. Mais je sais que mes parents aiment bien, et j'aime bien regarder quelques animés. je me suis donc dit que je pourrais remplacer ces plateformes en "industrialisant" le téléchargement de films et séries à la demande, et en ayant une plateforme auto-hébergée pour accéder à ces contenus. 

Je ne vais pas trop rentrer dans les détails de ce que j'ai mis en place pour ne pas avoir de soucis avec la justice, cependant, il m'a suffit d'un client Torrent, de programmes en -arr (vous trouverez !) et d'une instance Jellyfin pour arriver à ce résultat, très convaincant ! Les contenus étaient stockés sur mon NAS, nouvellement acquis d'occasion, et le stockage était monté en iSCSI sur l'Orange Pi.

Quelques mois plus tard, j'ai reçu un petit courrier recommandé de l'Arcom, et j'ai décidé de tout stopper. Je ne suis pas forcément d'accord avec ces interdictions de partage de contenus (surtout vu la douille que représentent les plateformes de streaming !), mais force est de constater que continuer m'aurait attiré des soucis 

# Scaling up !

J'ai tourné avec cette configuration pendant 2 ans, et ça m'était largement suffisant. Il y a 3 ans de cela cependant, un de mes amis proches m'a récupéré 2 Lenovo Tiny m710Q (la version avec un i3). Ils ont pris la poussière dans son placard jusqu'à ce que je me décide à passer dans la ville où j'ai fait la permière partie de mes études, et à les récupérer. Forcément, ces deux machines sont bien plus puissantes que mon petit Orange Pi !

Le copain qui m'a récupéré ces machines m'a aussi donné un petit switch manageable, afin de connecter tout ça !

## Proxmox

L'arrivée de ces deux Lenovo m710Q a marqué un tournant. Fini l'époque où je devais choisir entre faire tourner Home Assistant OU PiHole - j'allais enfin pouvoir virtualiser ! Le choix de Proxmox s'est imposé naturellement : interface web simple, communauté active, et surtout, la possibilité de faire cohabiter des VMs et des containers LXC.

Pourquoi virtualiser alors que tout fonctionnait en "bare metal" ? D'abord pour séparer les services : chaque application dans sa propre VM, c'est l'assurance qu'un plantage n'affecte pas tout le reste. Et puis pouvoir éteindre uniquement certains services sans tout couper, c'est quand même pratique ! L'abstraction matérielle limite aussi les risques de sécurité, et accessoirement, ça me permet de garder la main sur mes compétences réseau acquises à l'IUT de Châlons-en-Champagne. Chaque fois qu’un service plante, je ne perds rien d’autre. La virtualisation m’offre cette tranquillité d’esprit.

Côté stockage, chaque VM dispose maintenant de son propre volume LVM, et j'ai mis en place du LVM over iSCSI sur le NAS pour bien séparer les données.

Côté NAS, c'est une machine de récupération, un WD EX2 Ultra configuré en RAID1 avec 2 disques 2To et une Target iSCSI pour s'y connecter.

La migration n'a pas été sans accrocs. Transfert des configs à coup de SCP, ma configuration Traefik devenue illisible que j'ai remplacée par Caddy, et surtout le piège réseau : mon VPN Wireguard de la Freebox ne permettait plus d'accéder aux services virtualisés. Il a fallu mettre en place un tunnel IPSec avec un Fortigate... mais ça, j'en reparle plus loin !

**Spécification du matériel utilisé (pour référence) :**

* PVE1 : Lenovo Thincentre M710q Tiny
  * 16 GB RAM
  * i3 7100t, 3,4GHz 2c4t
  * NIC 1Gbps
  * SSD M2 240GB
  * HDD Sata 1To
* PVE2 : Lenovo Thincentre M710q Tiny - éteint
  * SSD M2 240GB
  * NIC 1Gbps
  * i3 7100t, 3,4GHz 2c4t
  * 8 GB RAM
* NAS : Western Digital EX2 Ultra
  * 2x Seagate IronWolf 2To, RAID 1
* FGT : Fortigate 60E
* Switch : D-Link DGS-3000-10TC
* Freebox Pop

## Un nouveau réseau

### Le réseau local 

!["Synoptique du réseau local"](/images/2025/08/TfNePFjVOqjvy7lizPCYVQOP782ZjdO1YvDR-z8yg2A=.png)

Avec ce matériel, mon réseau local ressemble à un réseau classique de TPE, à ceci près qu'on y trouve quelques compromis techniques.

Le premier étant le WiFi. Je suis plutôt satisfait de la puce WiFi 6 de la Freebox Pop (avec son répèteur), du coup c'est ce que j'utilise pour tous mes appareils en WiFi. Ceci étant, le DHCP est géré par le FGT, ce qui permet de le définir en passerelle par défaut pour pouvoir accéder aux autres réseaux (Homelab, et Prod Network).

Le FGT est connecté au switch en utilisant deux liens 1GbE aggrégés en LACP, ce qui donne un total de 2GbE entre le switch et le FGT.

### Le réseau fédéré : Le Federated Git Network (FGN)

> La GitFamily est simplement un nom inventé de notre groupe d'amis, cela remonte à il y a quelques années, et c'est resté :)

Et c'est là qu'avec 4 amis et anciens de l'IUT, nous avons eu une idée. C'est en fait ce que je qualifierai de bêtise, mais une bonne bêtise : et si on fédérait nos réseaux respectifs comme le font les opérateurs ?

OK, on va reprendre depuis le début.

Internet, c'est une interconnexion de réseaux. Jusque là, rien de nouveau, inter - network. Pour qu'une machine A chez un opérateur OpA puisse contacter un ordinateur B sur le réseau de l'opérateur OpB, il faut que le paquet passe par un point d'interconnexion, c'est à dire un point où les deux réseaux communiquent.

Au niveau de ce point d'interconnexion, il y a un protocole qui permet d'échanger des informations entre les deux réseaux : BGP soit Border Gateway Protocol. Concrètement, le routeur "EDGE" d'OpA va annoncer les routes du réseau d'OpA via BGP au routeur EDGE d'OpB.

Les routes, c'est simplement une annonce, qui dit "Si tu veux contacter l'addresse de A, il faut passer par moi", ou encore "Si tu veux contacter l'adresse de C, il faut passer par moi" même si C n'est pas dans le réseau d'OpA : Le paquet transitera par le réseau d'OpA, pour aller vers OpC, où là aussi, il y a une interconnexion, et un échange de routes BGP.

C'est très simplifié, mais ça permet d'avoir l'idée en tête. 

Et alors, cette bêtise ? Eh bien c'est exactement la même chose. Chacun de nous possède un routeur (en l'occurence, j'ai un Fortigate 60E d'occas). On va d'abord monter chacun un lien, une interconnexion entre nos routeurs. Ensuite, sur cette interconnexion, on va annoncer nos routes en utilisant BGP.

Pour faire l'interconnexion entre nos routeurs, on ne va pas tirer une fibre entre chaque personne, logistiquement c'est compliqué, et il faut avoir les moyens ! Par contre, nous avons tous une connexion Internet, donc on peut faire transiter des données entre nos réseaux en utilisant les réseaux de nos opérateurs Internet ! Nous avons donc monté des liens IPSec. Il faut voir ça comme une "lien virtuel" qui fait passer les paquets d'un réseau à l'autre, au dessus du réseau de l'opérateur, mais de façon chiffrée. Du côté de nos routeurs "EDGE", c'est tout comme si on avait connecté une fibre entre nos routeurs. 

Vous suivez toujours ? Bien, parce que le plus dur est passé.

On s'est ensuite répartis des réseaux privés (192.168.0.0/16, 172.16.0.0/16, 10.0.0.0/8) entre nous quatre. De cette façon, chacun peut annoncer aux autres les préfixes qu'il "possède". En l'occurence, mon routeur annonce 172.24.0.0/14 et 10.128.0.0/10.

!["Diagramme du réseau - Les informations sensibles ont été effacées en partie puis floutés"](/images/2025/08/3uSNmgKG3DnnQBC2uRxGrH93VrVtzvNXB62Z3YSOnx0=.png)

La conséquence de tout ça ? Eh bien maintenant, si je veux contacter une IP interne d'un des membres du réseau, eh bien c'est exactement comme si ce réseau interne était connecté à mon routeur : 

!["Traceroute de mon ordi, connecté au VPN d'un membre, jusqu'à un de mes serveurs. Oui je sais, ce n'est pas du 10.128, c'était au début :)"](/images/2025/08/obM6VOgCL91Go25si390io-3FU-RiwfNuvMZpA4d0dY=.webp)

### Accès distant

L'accès distant à mon Homelab se faisait avec Wireguard au tout début. Sauf que c'était le wireguard fourni par la Freebox. Sauf que la Freebox sait joindre son réseau (192.168.1.0/24), mais pas les réseaux à l'intérieur (en passant par le Fortigate). J'ai donc passé le FGT en DMZ (zone démilitarisée) ce qui a pour effet de rediriger tout le trafic entrant vers le FGT. Ensuite, j'ai mis en place un VPN IPSec IKEv2 client, qui me sert à me connecter au réseau fédéré, et à accéder à mon homelab même quand je ne suis pas chez moi.

Grâce à ça, je peux accéder à mon homelab de n’importe où, comme si j’étais physiquement chez moi, tout en restant *relativement* protégé des intrusions.

Franchement, je suis un peu embêté par ça.. Aujourd'hui, la Freebox ne me sert plus. C''est un simple routeur en plus qui me permet de me connecter au réseau de l'opérateur. Le problème, c'est que ça consomme ! Il faudrait avoir la possibilité de passer en direct avec son propre routeur, mais ce n'est pas encore dans la mentalité Farançaise à priori. Par contre, en Italie où je suis allé en déplacement professionnel, c'est possible ! Si seulement cette liberté pouvait arriver en France !

## I need services !

Forcément, derrière tout ce matériel, j'ai mis une prise connectée (on ne se refait pas), et avec la consommation moyenne (heures pleines et heures creuses prises en compte), le coût d'opération de tout cela est d'environ 100€ par an, ce qui entamme ma rentabilité ! Il faut donc que pour ce prix, mon homelab me fournisse des services pour lesquel je paierais autrement !

En plus de HomeAssistant et PiHole, voici les services que j'ai déployé sur PVE1 (non, il n'a pas encore de petit nom !) :

* **Immich&#x20;**: pour le stockage et la sauvegarde de mes photos, ce qui me permet de ne pas payer un abonnement Google One et de ne pas balancer mes photos privées aux Etats-Unis
* **VaultWarden** : pour le stockage de mes mots de passe. c'est un service un peu particulier car il n'est accessible qu'en interne, ou depuis l'adresse IP de mon travail, ce qui minimise les risques que ce service sensible se retrouve attaqué (surtout s'il fait l'objet d'une CVE !). J'ai passé le pas car en ce moment, BitWarden envoie pas mal de message de connexions non-autorisées, et j'en ai reçu un, ce qui m'a fait un peu peur. Ainsi, en auto-hébergé, les risques sont quand-même bien diminués ! De plus, VW stocke les données sous forme chiffrée côté serveur, avec déchiffrement côté client, ce qui fait que si un de mes disques (que je n'aurais pas détruit au préalable) se retrouve dans la nature, personne ne pourra accéder à ces mots de passe (en tout cas, dans cette période pré-quantique !)
* **Affine :&#x20;**&#x55;n clone de Notion, et d'Obsidienne. Affine a l'avantage d'être mis à jour en temps-réel par rapport à Obsidienne. On peut aussi partager des workspaces avec des amis, des collègues, ce qui le rend plutôt intéressant !
* **Paperless NGX** : Un outil pour stocker tous mes documents importants de façon numérique. Comme VW, le service n'est accessible qu'en interne et depuis l'IP du boulot. Les sauvegardes sont chiffrées sur le Proxmox Backup Server d'un ami.
* **Uptime Kuma :&#x20;**&#x55;n petit service mutualisé sur tout le FGN, qui permet de monitorer le bon fonctionnement de nos services
* **Gokapi&#x20;**: Un petit service de transfert de fichiers, une alternative à Firefox Send ou WeTransfer auto-hébergée
* **Caddy&#x20;**: Le reverse-proxy qui se trouve devant chacun de ces services afin de gérer le SSL-TLS, et le filtrage par IP. Les access-log sont envoyés par rsyslog au conteneur qui héberge CrowdSec.
* **CrowdSec&#x20;**: Un service de réputation qui permet de bloquer l'accès aux IPs malveillantes en utilisant une base de connaissance mise à jour en temps-réel par la toutes les sondes des instances CrowdSec déployées. Crowdsec détecte aussi des comportements en utilisant les logs de Caddy pour bloquer proactivement les IPs qui feraient n'importe quoi.

!["Une vue des comportements bloqués par CrowdSec"](/images/2025/08/-CRr_9ksnkUApAXiMWrkenFZMCI3pzjhbu76HbZkm-g=.png)

PVE2 est encore éteint, le but est d'économiser de la consommation électrique. Bien que ces deux machines soient en cluster, j'ai un peu modifié les voix de PVE1 pour que le quorum soit bon si PVE2 est éteint ou allumé. Cela permet de tourner en mode dégradé sans trop de soucis. Aussi, aucun de mes services sont en High Availability, je n'en ai pas forcément le besoin, cela reste un Homelab ! Ainsi, PVE2 est dédié à l'expérimentation de nouveaux services, techniques, etc.

### Réseau dans l'hyperviseur

Afin d'organiser mes différentes VM et conteneurs dans Proxmox, j'ai créé un réseau de "prod". Pour que ce réseau soit partagé entre les deux machines, j'ai utilisé la fonction SDN (Software Defined Network) de Proxmox.

J'ai donc créé une zone qui s'appelle *shared1 &#x20;*&#x71;ui contient un vnet *swprod&#x20;*&#x71;ui lui-même contient le sous-réseau des machines de prod (10.128.0.0/24)

![Une vue VNets et Subnets dans la configuration SDN du cluster](/images/2025/08/lLKt23CqTIBZPd4F-WdK25Cv55z4gMmSqxc7-bK4RHo=.png)

Chacun des containers et VM a une IP privée dans ce subnet, et leur accès internet est NATé via la passerelle (10.128.0.1). Ceci étant, ce réseau reste routé, ce qui signifie qu'on peut joindre ces machines via leurs IP directement. Pour ce faire, j'ai créé une route statique dans le Fortigate.

En utilisant un **redistribute static ce réseau&#x20;**&#x65;st annoncé aux voisins BGP, qui peuvent joindre directement ces machines.

### Authentification et sécurité

Afin de protéger ces services, j'utilise une combinaison de plusieurs outils.

Tout d'abord, pour les services critiques, j'utilise un filtrage d'IP source au niveau de Caddy, ce qui me permet de m'y connecter quand je suis chez moi, en VPN, ou alors depuis mon travail.

Ensuite, pour les services "publics", j'utilise CrowdSec. Concrètement, lors d'une connexion, Caddy interroge CrowdSec en lui demandant si l'IP source de la requête a une mauvaise réputation. Si c'est le cas, Caddy renvoie un 403, sinon, l'accès est autorisé. Aussi, CrowdSec analyse les logs de Caddy pour déterminer si une attaque est en cours (bruteforce, scans, etc). Si c'est le cas, c'est pareil, Caddy renvoie un 403.

J'ai aussi prévu de rajouter un peu de supervision avec Wazuh (c'est vaste, et je pense que ça mérite son propre article).

Pareillement, je suis en train de rajouter de l'authentification SSO (Single Sign-On) avec Authentik. 

# Conclusion

Au final, tout ça est parti d’un petit chauffage à domotiser, et je me retrouve aujourd’hui avec un homelab qui bouffe de l’électricité, qui me fait passer des soirées entières à débugger des configs réseau improbables… mais qui me régale. C’est officiel : j’ai attrapé le virus du homelab, et je doute fort qu’il existe un vaccin.

Aujourd'hui, c'est plus d'une dizaine de conteneurs et de VMs qui me servent à faire tourner tous ces services, pour environ 40W de consommation (Box Internet, FGT, NAS, switch et PVE1), soit une centaine d'euros par an.



Alexis
