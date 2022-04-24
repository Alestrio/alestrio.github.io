---
author: "Alexis LEBEL"
title: "Créer un site web statique avec Github Pages et Hugo"
date: 2022-04-24
tags: ["static", "github", "hugo", "website"]
thumbnail: /images/2022/04/github_actions.jpg
---

Durant ces deux dernières années, j'ai étudié les Réseaux et Télécommunications à l'IUT de Châlons-en-Champagne.
Parmis les compétences que j'ai acquises, il y a, bien entendu des compétences techniques mais aussi des compétences de communication.

Lors de mon projet de troisième année, j'ai été invité à remplacer le traditionnel rapport de projet papier par un site web statique, dont la vocation était aussi de servir de documentation, sur la production du projet. 

Dans ce genre de cas, pour faire passer un message, ce n'est pas le tout de faire un site ultra bien fait, avec des animations de partout mais très lourd. Il est plus efficace de miser sur la sobriété, et sur la simplicité, afin que l'information soit facile à retrouver.

De plus, quand on rédige cette information sur une grande période, et quelle est amenée à évoluer, il est impensable de partir dans du code classique (HTML/CSS/JS), qui serait bien trop long à alimenter.

# Les générateurs de sites statiques

Les générateurs de site statiques sont des outils qui permettent de créer des sites web statiques, c'est à dire sans communication en arrière-plan avec un serveur pour fournir les données. 

Il s'agit donc de simples pages web, dont le contenu ne change pas en temps réel.

L'avantage, c'est que ces outils utilisent comme base des fichiers souvent au format Markdown (comme les README Github).
Ce format permet un formattage de texte très simple en utilisant des symboles. Par exemple `#` permet de créer un titre de niveau 1.

Ensuite, le code source des pages est généré automatiquement, sans aucune intervention humaine.
Cela facilite énormément la création de contenu, sans passer par des plateformes de blogs traditionnelles, qui intègrent souvent des modules lourds pour les navigateurs. (Wordpress par exemple)

![static_site_generator](/images/2022/04/static_site_generator.webp)
*Source : Netlify*

Les sites statiques générés sont ainsi très rapides, dû au fait que ce ne sont que des empilements de balises HTML et de classes CSS, sans aucune autre interaction.

Le générateur de site statique que nous allons utiliser ici est [Hugo](https://gohugo.io/).

# Tutoriel

## Initialisation du site

Tout d'abord, nous allons créer un nouveau dossier dans lequel nous allons créer notre site.

```bash
mkdir static_personnal_site && cd static_personnal_site
```

Dans ce dossier, nous allons mettre le binaire Hugo téléchargé ici : https://gohugo.io/getting-started/installing/

Puis nous allons initialiser le site avec Hugo : 

```bash
hugo new site
```

Voici l'architecture ainsi créée :

```
.
├── archetypes => contient les modèles de contenu
├── config.toml => contient les configurations de Hugo
├── content => contient les contenus
├── data => contient les données
├── layouts => contient les modèles de page
├── static => contient les fichiers statiques (images, scripts)
└── themes => contient les thèmes
```

Nous allons ensuite initialiser le repo Git :
    
```bash
git init
```

Git ici, va nous servir à gérer les différentes versions du site. La plateforme GitHub va nous permettre de publier ces différentes versions. Nous ne rentrerons pas ici dans les détails de ces outils.

## Choix et installation du thème

Ensuite, il faut choisir un thème pour le site.

Le thème en lui même importe peu, c'est à choisir selon vos goûts. Ce qui est important par contre, c'est de choisir un thème que vous pourrez republier pour votre site. Il faut donc bien faire attention à la license d'utilisation du thème. Parfois, il faut l'acheter pour pouvoir l'utiliser publiquement.

Beaucoup de thèmes sont disponibles ici : https://themes.gohugo.io/

Une fois le thème choisi, nous allons l'installer, pour ce faire, il faut copier l'URL du repo du thème puis faire la commande :

    
```bash
git submodule add https://url_du_repo_du_theme.git themes/mytheme
```

Généralement, le thème contient un site d'exemple, qui peut être utilisé comme base pour votre site. Si c'est le cas, il suffit de copier le contenu des dossiers de l'exemple dans le dossier du site. 

Le plus important dans ce cas, c'est de récupérer le fichier `config.toml` du thème, et de le mettre à la racine du site. Il contient des informations qui sont utilisées par les différentes pages du thème.

Pour vérifier que tout fonctionne bien et que le site "compile", il suffit de faire la commande :

```bash
hugo server
```
A partir de là, le site est accessible en local sur le port 1313. Si tout fonctionne bien, on peut passer à la suite.

## Intégration continue

Afin de compiler automatiquement le site internet lors de la modification d'un fichier, nous allons nous servir de Github Actions.

Nous allons donc créer un dossier `.github/workflows/` dans le dossier du site, et y mettre le fichier `hugo.yml`.

```yml
name: Hugo build

on:
  push:
    branches:
      - master  # Set a branch to deploy
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/master'
        with:
          github_token: ${{ secrets.TOKEN }}
          publish_dir: ./public
```

On va ensuite créer un repo Github, avec un nom de la forme `pseudo.github.io`.

Il faut ensuite aller dans les paramètres de votre profil Github, onglet "paramètres développeur", et y générer un token personnel (PAT) avec un accès aux repos.

![github_pages_pat](/images/2022/04/github_pages_pat.jpg)

Une fois ce token créé, il faut l'ajouter dans les paramètres du repo, dans un secret, dont le nom sera `TOKEN`.

![github_pages_token](/images/2022/04/github_pages_token.jpg)

## Mise en ligne avec Github Pages

A partir de là, nous allons ajouter ce repo aux **origin** du repo local.

```bash
git remote add origin https://github.com/user/user.github.io.git
```

Puis nous allons envoyer les modifications à Github :

```bash
git push -u origin master
```

A partir de là, le site va être compilé automatiquement, mais ne sera pas encore disponible.
Cet automatisme va créer une branche `gh-pages` sur Github, et y copier le contenu du dossier `public`. Nous allons donc devoir dire à Github qu'il faut publier le contenu de cette branche.

Pour ce faire, il faut aller dans les paramètres du repo, dans l'onglet "pages", et changer ce paramètre :

![github_pages_branch](/images/2022/04/github_pages_branch.jpg)

A partir de là, le site est disponible à l'adressse `https://user.github.io`.

Les rapports de déploiement sont disponibles dans l'onglet "actions" du repo :

![github_actions](/images/2022/04/github_pages_actions.jpg)

## Bonus : Utiliser son propre domaine

Pour utiliser un domaine personnalisé, il va falloir éditer la zone DNS de ce dernier.
Dans cette zone DNS, on va créer 4 champs A :

```
mondomaine.fr A 185.199.108.153
mondomaine.fr A 185.199.109.153
mondomaine.fr A 185.199.110.153
mondomaine.fr A 185.199.111.153
```

et 4 champs AAAA :

```
mondomaine.fr AAAA 2606:50c0:8000::153
mondomaine.fr AAAA 2606:50c0:8001::153
mondomaine.fr AAAA 2606:50c0:8002::153
mondomaine.fr AAAA 2606:50c0:8003::153
```

![github_pages_ovh](/images/2022/04/github_pages_ovh.jpg)

Une fois cela fait, nous allons paramétrer le site pour qu'il pointe vers ce domaine. Pour cela, il faut se rendre dans les paramètres du repo, dans l'onglet "pages", et changer ce paramètre :

![github_pages_domain](/images/2022/04/github_pages_domain.jpg)

Attention : l'application de la configuration de la zone DNS peut prendre jusque 24 heures. Il faudra donc peut être attendre un peu afin de configurer ce paramètre.

## Publier du contenu

Pour créer un article ou une page sur votre nouveau site, cela se passe dans le dossier `content`. Pour créer une nouvelle page, il suffit de créer un fichier `.md` dans ce dossier.

Au tout début de ce fichier, il faut mettre une en-tête de la forme :

```md
---
author: "Jean Bon"
title: "Titre de la page"
date: 2022-04-23
tags: ["des", "mots", "clés", "importants"]
thumbnail: /images/une/miniature.format
---
```

Une fois cela fait, vous pouvez écrire le contenu de la page.

Pour enregistrer la version de votre site, il suffit de faire :

```bash
git commit -m "Nouvelle page"
```

... puis de les envoyer à Github :

```bash
git push
```

# Conclusion

Vous savez maintenant comment mettre en place un site web statique avec Hugo et Github Pages.

A noter qu'avec cette méthode, il est impossible de dynamiser le site avec des données d'API par exemple. Cela ne permet que de faire des sites internet très simples, mais très rapides à alimenter.

