---
author: "Alexis LEBEL"
title: "La calculatrice Numworks : Avis, et bidouilles"
date: 2020-03-25
tags: ["numworks", "calculatrice"]
#thumbnail: /images/2020/12/automate_1.jpg
---

Bonjour à tous ! Bon, tout d’abord, un petit mot de prévention : en ces temps de crise, restez chez vous et adoptez les gestes barrière, c’est important. Et si vous vous ennuyez à cause du confinement, et bien, bidouillez ! D’ailleurs, cet article est à propos de mes bidouilles de cette période, et plus précisément des changement apportés à ma fidèle calculatrice graphique : la Numworks N0100.

Mais c’est quoi la Numworks ?
Numworks est le nom d’une calculatrice graphique ultra puissante, et je pèse mes mots. Il s’agit d’une machine reposant sur un microcontrôleur STM32 soit un Cortex M4 dépassant largement les performances des calculatrices graphiques de la même gamme. Ses points forts ? Une interface graphique claire et intuitive, une integration native du langage Python, des couleurs utiles, et surtout une construction totalement open-source, du hardware au firmware. Et oui, les sources du système d’exploitation Epsilon sont disponibles sur GitHub ! Il en va de même pour les plans 3D des parties mécaniques et les plans des PCBs, disponibles gratuitement sur le site de Numworks ! En bref, la calculatrice parfaite du bidouilleur !

En tant que lycéen, ça change beaucoup par rapport à une Casio ou une TI ?
Personnellement, je suis passé d’une Casio Graph 35 + E à cette machine alors oui ça change beaucoup. Là où j’ai mis presque trois mois en seconde à bidouiller pour comprendre toutes les fonctions de la Casio, il m’a fallu 2 semaines pour savoir exploiter à fond la Numworks. Ca montre à quel point l’interface est claire et intuitive ! Les fonctions sont bien plus faciles à trouver et l’affichage et beaucoup plus clair et représentatif, notamment avec les représentations graphiques de fonctions. Avant de la commander, j’ai même pu l’essayer grâce à l’émulateur, que j’utilise quand je n’ai pas ma fidèle machine sous la main.

Et ces améliorations dont tu parles ?
Ah, rentrons dans le vif du sujet ! Voici la partie la plus intéressante. Il va sans dire que vu le côté open-source de cette machine, elle allait être bidouillée grandement. J’ai donc commencé par m’intéresser à la partie logiciel. Bon première surprise, installer l’environnement de développement est simple et surtout ultra bien expliqué sur le site de Numworks ! Ca m’a pris 30 mn la première fois. Vraiment, c’est ultra clair, et permet de se mettre dans le bain très vite. Puis, après quelques recherches, j’ai découvert le firmware alternatif Omega, et… en 1h, c’était compilé et flashé ! 😀

Ce firmware présentes plusieurs améliorations que j’adore :

Le tableau périodique des éléments, indispensable en physique-chimie pour gagner un maximum de temps
Une flopée de constantes physiques et chimiques en tout genre
La gestion des unités et des conversions (qu’on retrouve d’ailleurs dans la version 13.1 d’Epsilon)
Le choix de la couleur de la Led pour le mode examen, inutile, mais fun
Le calcul symbolique, je m’en sers pas trop, mais ça peut être utile. Cette fonction n’est pas du niveau de KhiCas, mais elle fait le taff quand même ! 🙂
Le nom du détenteur de la machine, affiché dans les paramètres, et modifiable seulement avec la compilation et le flash d’un autre firmware
Plein d’autres fonctions dont je ne me rappelle plus mais qui sont très utiles !
Bon, voilà, à ce moment, j’avais encore mieux que ma Numworks d’origine, ce qui est incroyable. Mais ça ne me suffisait pas, je ne m’étais pas assez sali les mains. J’ai donc créé mon propre fork de ce fork d’Epsilon (oui, ça fait beaucoup 🙂 ), où je travaille sur mes propres améliorations. On peut d’ores et déjà compter parmi ces amélioration l’allumage de la LED lors du chargement. Je travaille de plus sur d’autres améliorations, tout en m’appuyant sur les dernières versions d’Omega, pour y intégrer les bugfixes.

J’ai pour projet d’être un peu moins sélectif concernant les ajouts de fonctionnalités dans ce fork, tout en respectant bien sûr les spécifications du mode examen de cette année. Donc si vous avez des fonctionnalités qui ont été refusées en PR pour Epsilon et Omega, passez-les dans ce fork, on ne sait jamais 😉

Conclusion
Pour conclure, on peut dire que je n’ai pas fini de m’amuser avec cette machine. Je l’adore et honnêtement, elle me fait gagner tellement de temps que je ne pourrais plus m’en passer. Je pourrais apparaître un peu trop enjoué avec cette simple calculatrice, mais honnêtement, elle le mérite. Pour 80€, on a une calculatrice claire, simple, efficace, et surtout évolutive avec des mises à jour très fréquentes ! La communauté autour de cette marque est elle aussi géniale, avec de l’entraide entre tout le monde, et les employés de cette entreprise sont très réactifs et répondent rapidement si vous avez un problème ! Si vous hésitiez, eh bien foncez, vous ne serez pas déçus ! D’ici là, portez vous bien, faites attention et restez chez vous !

Bonnes bidouilles,