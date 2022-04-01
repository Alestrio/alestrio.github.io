---
author: "Alexis LEBEL"
title: "Remplacement de l’effecteur de ma Kossel"
date: 2020-06-29
tags: ["kossel", "effector", "replacement", "3d"]
#thumbnail: /images/2020/12/automate_1.jpg
---

Bonjour à tous !
Ça fait maintenant 4 ans que j’ai ma Kossel, qui est une imprimante 3D de type Delta. Et évidemment, durant ces quatre ans, j’ai fait des erreurs, j’ai bidouillé, et parfois, cassé/endommagé des composants. Par conséquent, je me suis mis en tête de remplacer tout ça, en commençant par l’effecteur.

L’effecteur est le composant qui permet de transformer de mouvement linéaire vertical des trois “towers” en mouvement linéaire horizontal parallèle au plateau. Ainsi, la combinaison des mouvement des trois “towers”, permet de définir la position de l’ensemble effecteur/système d’extrusion en X, Y et en Z en cas de mouvement simultané des trois “towers”. (Une tower est un des axes d’une delta)

Le choix de l’effecteur
Tout d’abord, il a fallu acheter cette pièce, en respectant quelques critères que je me suis fixé.

Le poids : il fallait que ce soit léger, afin qu’il y ait moins d’inertie dans les mouvements
Le type de système d’extrusion : il fallait le support d’une E3D V6, plus efficace, moins sujette aux fuites
Un switch mécanique : afin de remplacer mon système de capteur inductif pour l’autocalibration
Une fixation magnétique : afin de réduire les jeux au niveau des liaisons et de pouvoir détacher l’ensemble facilement
Je me suis donc tourné vers ce genre de produit :


Qui avait plusieurs avantages :

La pièce est en injection plastique, ce qui la rend légère
Elle possède un switch mécanique
Elle supporte l’E3D V6
Malheureusement, celle-ci ne possède aucune fixation magnétique, mais ce n’est pas ça qui va nous arrêter !

Je passe vite-fait sur ce que j’ai commandé en plus de ça :


Cartouche de chauffe

Kit de liaison magnétique

Tubes en téflon + raccords (pneufits)
Assemblage de l’effecteur
Malgré le fait qu’il n’existe aucun manuel d’assemblage, c’était assez facile. Dans les grandes lignes : on met la pièce et on visse ! 😀 Malgré tout, il a fallu quand même s’adapter pour mettre le système de fixation magnétique. J’ai donc créé une pièce à imprimer en 3D afin de pallier à ce problème (Pièce blanche sur la photo).

Une fois monté, je me suis dit que c’était un peu faible, et j’avais des doutes sur le système. En effet, je redoutais que la fixation magnétique ne tienne pas durant une impression. Malgré tout, les essais ont prouvé que ça fonctionne, alors je suis plutôt content.

Je n’ai pas encore utilisé l’ILS (Interrupteur à Lame Souple) pour l’autocalibration, mais peut-être que j’essaierai quand j’aurais un peu de courage 😀 .

Avec tout ça, l’effecteur ressemble donc à ça :


Tests
Bon, tout d’abord, après avoir branché le tout, j’ai commencé par tester la chauffe de la tête. Malheureusement, j’utilisais un clone chinois très mauvais d’E3D V6, et le heatbreak (pièce qui sert d’isolation thermique entre le bloc de chauffe et le refroidisseur, et aussi de guide pour le filament) et la buse étaient une seule et même pièce, qui plus est sans espace d’isolation thermique entre les deux parties de la tête. La conséquence ? La tête chauffait, mais une trop grande partie de la chaleur était dissipée par le refroidisseur. J’ai donc acheté des heatbreak plus conventionnels ainsi qu’une buse, et le problème était réglé !

Un autre test, après calibration (qui est d’ailleurs le sujet du prochain article… SPOILER) m’a permis de voir que je n’avais pas vissé assez fort le heatbreak contre la buse. je me suis donc retrouvé avec des fuites des deux côtés du bloc de chauffe. Une fois resserré, plus de problème !

J’ai donc pu imprimer deux trois petites choses, en variant mes paramètres, afin de me rapprocher des paramètres optimaux que j’avais avant. ( Slicer : Simplify3D)


Alors, je vous l’accorde, c’est pas très beau, mais j’y travaille ! Ce n’est qu’une question de temps avant que je trouve les paramètres optimaux. Ces objets sont faits en TitanX de ReForm (FormFutura) qui est un ABS modifié avec moins de problème de warping ou autres, et surtout, qui est une matière recyclée !

Conclusion
Voilà, nous y sommes, le moment fatidique de la conclusion, qui implique que cette aventure est finie…. ou pas. En effet, avec une imprimante 3D, il est toujours possible d’aller plus loin dans les améliorations. Ainsi, je ne pense pas que c’est le dernier article sur ma delta, d’ailleurs, j’en suis sûr, car comme dit précédemment, le prochain article sera à propos de la calibration de cette machine ! N’hésitez pas à vous abonner à la newsletter afin d’être avertis de la sortie des derniers articles, surtout si le sujet vous intéresse !

A la prochaine, et bonne bidouille !