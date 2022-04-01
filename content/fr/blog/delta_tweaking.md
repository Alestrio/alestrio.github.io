---
author: "Alexis LEBEL"
title: "Comment (bien) régler une delta"
date: 2020-09-01
tags: ["delta", "3d", "3d-printing"]
thumbnail: /images/2020/05/photo-delta-1.jpg
---

Vous le savez sûrement si vous me suivez sur Twitter, mais je possède une version béta de l’Anycubic Kossel, une imprimante 3D de type Delta, c’est à dire que les trois moteurs fonctionnent en même temps. Ce type d’imprimante est fait pour imprimer des objets très rapidement, cependant, il y a un inconvénient de taille à ce type de machine : la calibration. La calibration d’une delta est un véritable calvaire et je pense avoir trouvé une technique pour simplifier ce moment horrible qui précède la première, et pas mal d’autres d’impressions, ce qui pourrait simplifier la vie de beaucoup de monde !

# Les prérequis
Tout d’abord, votre machine doit fonctionner avec Marlin 2.0+, qui permet de jouer sur le delta-radius et les offset de capteurs de fin de course dont je vais parler un peu plus tard. Faites aussi gaffe à avoir un plateau bien plan, quitte à mettre un disque en verre pour avoir quelque chose de bien plan, puisque le plateau en aluminium se dilate quand on chauffe le plateau, ce qui fausse et la planéité, et le tilt calculé du plateau. D’ailleurs, cela m’amène au troisième point : calibrez à chaud ! la chaleur et sa dilatation sont à prendre en compte pour chaque calibration puisque le tilt change pour chaque température.

# Les paramètres

Les paramètres sur lesquels nous allons jouer sont :

Les offsets de capteurs de fin de course : ce sont les décalages en mm (toujours négatifs) des chariots moteurs par rapport aux capteurs de fin de course, ces paramètres permettent de régler le tilt du plateau.
Le delta-radius : détermine la planéité de la trajectoire de la tête d’impression par rapport au plateau lorsque vous commandez une trajectoire plane et rectiligne (oui, je sais, il faut faire un peu de gymnastique mentale pour bien comprendre cette phrase 😀 )
La hauteur hors-tout : sert à ajuster la hauteur générale de la buse par rapport au plateau, c’est utile quand on change la hauteur de couche dans le slicer ou qu’on change la buse, il faut alors réajuster cette valeur.
Ces paramètres sont très sensibles, c’est pour cela que l’on va les ajuster petit à petit, afin de réduire notre marge d’erreur au maximum.

# La méthode

![algo](/images/2020/05/algo.png)

Algorithme de calibrage
La méthode de calibrage est simple, quoique longue. Le secret, c’est “Patience et longueur de temps”. Afin d’avoir un réglage parfait, il faut calibrer chaque tower l’une après l’autre jusqu’à avoir une marge d’erreur négligeable, puis le centre en ajustant le delta radius une fois à la louche, dans le sens inverse de la déformation. Il faut ensuite recommencer les towers, puis le centre, jusqu’à avoir une très petite marge d’erreur. Le plus important, c’est de faire cela à chaud, puisque ces réglages changent en fonction de la température (Dilatation du corps de chauffe, du plateau…).

Pour calibrer une tower, il faut aller dans “Delta Calibration” puis “Calibrate X/Y/Z”. Ainsi, vous pourrez voir le décalage à appliquer sur les capteurs de fin de course. Pour moins en suer, vous pouvez activer (ou du moins, dé-commenter) l’option qui permet d’aller à des coordonnées en Z négatives dans le firmware.

Pour calibrer le centre, il faut aller dans “Delta Calibration” puis “Calibrate Center”. D’ici, vous pourrez voir le décalage entre les towers et le centre, en ajustant le delta radius. Le delta radius, il faut un peu le voir comme le paramètre a d’une fonction du second degré. Ainsi, quand a augmente, la pente de la parabole tire vers le haut (en gros).

Vous pouvez aussi utiliser le paramètre “Delta Height” qui va augmenter ou amoindrir la distance entre la buse et le plateau. Attention, ici, petite subtilité : il faut penser que peut importe ce paramètre, vous avez une certaine distance entre vos endstops et le plateau. Ainsi, si vous augmentez cette valeur, la buse va se rapprocher du plateau, et si vous baissez cette valeur, la buse va s’éloigner du plateau. Ainsi, vous pouvez mettre vos décalages au plus proche de chaque point de contrôle (Exemple X=1.2, Y=1.4, Z=1.3, Centre = 1.8 ===> X/Y/Z/Centre = 1,4 puis on joue sur la delta height pour les 0.4 mm restants)

# Conclusion

Ainsi s’achève ce petit tutoriel, dont la méthode m’a demandé pas mal de temps à maîtriser. J’espère qu’il vous aidera à comprendre un peu mieux le fonctionnement et la calibration d’une delta !

De mon côté, l’un des capteurs de fin de course de ma machine à lâché, encore un peu de bidouille en vue !

Amusez-vous bien !