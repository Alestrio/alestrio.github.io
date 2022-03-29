---
author: "Alexis LEBEL"
title: "Connecter un portail électrique"
date: 2021-01-01
description: "Comment rendre un portail électrique connecté"
tags: ["IoT"]
thumbnail: /images/2020/12/automate_1.jpg
---

Bonjour à tous ! Pour ce premier article dans la série “Connecter sa maison” nous allons nous intéresser à la domotisation d’un portail électrique à télécommande. Nous allons pour cela utiliser un système fait maison, afin d’envoyer un signal à l’automate du portail, afin de commander son ouverture ou sa fermeture depuis un système vocal comme Alexa ou manuellement.

# Étude de l’automate :

Pour commencer, voici une photo de l’automate du portail. On peut voir l’alimentation 220V – 12V tout en haut, ainsi que le fil d’antenne, et en dessous, l’automate en lui même.

![automate_1](/images/2020/12/automate_1.jpg)

Nous allons nous intéresser tout particulièrement aux borniers du dessous de l’automate : Le bornier d’alimentation et le bornier de signal.

Le bornier d’alimentation est celui du milieu. On peut y voir une sortie 12V et GND. Ces sorties seront l’alimentation de notre système.

Le bornier signal est celui tout à droite. Plusieurs fils y sont déjà connectés : c’est l’interrupteur à clé. Nous allons nous brancher en parallèle de cet interrupteur à clé.

Pour envoyer un signal à cette broche, il faut ponter le GND (-) à la broche B2 pendant 1-2s. ceci déclenche une action au portail : la fermeture ou l’ouverture complète de celui-ci. La broche que nous n’utilisons pas (B1) est pontée en permanence au GND.

# Système de commande : partie électronique

Afin de réaliser ce pontage, nous allons créer un petit système tout simple, avec un relai électromécanique.

Fonctionnement d’un relai électromécanique :

![fonctionnement_relai](/images/2020/12/relay_principle.png)
https://circuitdigest.com/article/relay-working-types-operation-applications

Nous allons contrôler ce système via Wi-Fi, grâce à un microcontrôleur que j’affectionne particulièrement : un ESP8266 !

Ainsi, nous allons avoir besoin des composants suivants :

- 1 Node MCU Amica
![amica_nodemcu](/images/2020/12/amica_nodemcu.jpg)

- 1 Convertisseur de 4 niveaux logiques 2595
![convertisseur_2595](/images/2020/12/convertisseur_2595.jpg)

- 1 Relai 5VDC – 220V
![relay_5v](/images/2020/12/relay_5v.png)

Pour le câblage, nous allons le faire comme à la figure suivante. Personnellement, j’ai utilisé les fils d’une vieille alim d’ordinateur afin de réaliser les connexions.

![cablage_relai](/images/2020/12/cablage.png)
Schéma technique de notre système

On peut sans problème fournir du +12V à l’Amica, puisqu’il possède un régulateur de tension intégré. Cela nous simplifie donc les branchements !

Le système est fini, électriquement, mais nous n’allons pas l’installer tout de suite.

# Système de commande : partie logicielle
## Téléverser le logiciel

Afin de contrôler ce système, nous allons utiliser le logiciel Tasmota. L’installation de ce logiciel sur notre microcontrôleur est très simple :

Tout d’abord, il faut télécharger l’utilitaire de téléversement à cette adresse : https://github.com/tasmota/tasmotizer/releases/

Ensuite, exécutez l’utilitaire et vous devriez vous retrouver avec cette interface :

![tasmota_1](/images/2020/12/tasmota_1.png)

L’interface de Tasmotizer
Sélectionnez ensuite les options comme ci-dessous :

![tasmota_2](/images/2020/12/tasmota_2.png)

La configuration de Tasmotizer
Puis dans “Select Port”, sélectionnez le port de votre appareil (généralement COM X)

Cliquez sur “Tasmotize”, et le logiciel est envoyé sur la carte, mais la configuration n’est pas finie !

## Configurer Tasmota

Configurer le Wi-Fi
Tout d’abord, connectez-vous au réseau Wi-Fi créé par le microcontrôleur : généralement “TASMOTA-XXXXXXX”, puis entrez les informations de votre réseau Wi-Fi (SSID, Clé réseau).

### Configurer le Relai
Pour déclencher une action sur l’automate, nous devons envoyer un signal bref, entre 1 et 2 secondes. Nous allons donc configurer cela, avec dans la tête, le fait que demander cette action à Alexa n’aura pour effet que d’activer le pin de signal, pas de le désactiver.

Nous allons donc devoir mettre une “règle” qui applique un minuteur pour désactiver le pin 1.5 secondes après son activation.

Tout d’abord, il faut accéder à la console de l’appareil. Pour ce faire, il faut ouvrir un navigateur, et entrer l’ip de l’appareil :

![tasmota_3](/images/2020/12/tasmota_3.png)

Si tout est câblé correctement, vous devriez pouvoir appuyez sur Toggle 1 ce qui devrait déclencher le relai.

Vous pouvez ensuite accéder à la console :

![tasmota_4](/images/2020/12/tasmota_4.png)

Dans laquelle il faut entrer la commande :

`RULE1 ON POWER1#State=1 DO BACKLOG DELAY 15; POWER1 OFF ENDON
Appuyez sur Entrée, et ça y est, votre sortie signal est configurée.`

Maintenant il faut que le système puisse être reconnu par Alexa, nous allons donc émuler le fonctionnement d’une ampoule connectée. Il faut donc aller dans les paramètres en cliquant sur “Configuration”.

![tasmota_5](/images/2020/12/tasmota_5.png)

Cliquez sur “Configure other” et activez le paramètre suivant :

![tasmota_6](/images/2020/12/tasmota_6.png)

Cliquez sur “Save” et la configuration est finie côté appareil !

# Installer le système
## Connexion au portail

Pour installer le système, nous allons utiliser le schéma électrique ci-dessus.

Pour l’alimentation, nous utilisons le bornier 12v – GND auquel nous branchons les quatre fils (+ Amica/Relai sur le +12V | – Amica/Relai sur le GND)

On peut ensuite brancher la borne commune (COM) sur la borne B2 de l’automate, et la borne normalement ouverte (NO) sur le GND associé au contact B2.

Et voilà ! C’est pas plus compliqué que ça pour les branchements !

## Configurer Alexa
Bon là, pas de surprise, on va faire un ajout d’appareil classique sur l’application Alexa. Rendez vous donc dans votre application :

Explication en images
Voici la liste d’instruction pour ajouter notre appareil :

>Aller dans l’onglet **“Plus”** \
Aller dans **“Ajouter un appareil”** \
Choisir **“Autre”** \
Cliquer sur **“Découvrir des appareils”** \
Ajouter l’appareil nommé **“Tasmota”** par défaut \
Le renommer dans l’application Alexa \

Maintenant, vous pouvez dire à Alexa “Alexa, ouvre le portail”. Cela permet d’ouvrir mais aussi de fermer le portail. Nous allons créer la routine “Alexa, ferme le portail” afin que cela soit plus intuitif :

Voici la liste d’instructions pour ajouter la routine :

>Aller dans **“Plus”** \
Cliquer sur **“Routines”** \
Appuyer sur le Plus, en haut à droite pour ajouter une routine \
Mettre **“Voix”** comme déclencheur \
Écrire la phrase à dire \
Cliquer sur **“Ajouter une action”** \
Sélectionnez l’appareil \
Le mettre en **“marche”** \
Sauvegarder et quitter \

# Conclusion

Voilà, le portail à l’origine non connecté, est maintenant contrôlable par Internet et par Alexa ! Si vous avez des questions, ou des suggestions, vous pouvez laisser un commentaire. J’espère que vous avez apprécié la lecture de cet article ! Je vous laisse avec la liste du matériel que j’ai utilisé !

Liens du matériel
Voici la liste du matériel utilisé pour cet appareil :

- https://www.gotronic.fr/art-module-relais-5-v-gt1080-26130.htm
- https://www.gotronic.fr/art-module-nodemcu-esp8266-27744.htm
- https://www.gotronic.fr/art-convertisseur-de-4-niveaux-logiques-2595-22272.htm

Bonne bidouille, et surtout, meilleurs vœux ! Que cette année soit bien meilleure pour tout le monde que 2020 !

Alexis