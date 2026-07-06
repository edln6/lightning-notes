# Release 🇫🇷

Ce doc explique comment faire une release de package R dans le calme et la sérénité.

## Cadre dans une organisation de plusieurs packages

Les releases se font ***en cascade***, de la plus grande dépendance (en haut) jusqu'au package qui en dépendent en bas. 

Donc pour faire la release de rjd3workspace, il faudra faire dans l'ordre :
1) La release de {rjd3jars}
2) La release de {rjd3toolkit}
3) La release de {rjd3x13}, {rjd3tramoseats} et {rjd3providers} (ordre indépendant)
4) La release de {rjd3workspace}

> [!CAUTION]
> Comme les releases se font une par une et qu'il n'est pas possible de les faire simultanément, sur la période qui sépare la release du premier package de celle du dernier package, il peut y avoir des erreurs de fonctionnement (sur GitHub comme sur le CRAN) car les versions installées par défault ne correspondront pas. 
> Mais c'est un mal pour un bien, ça ne dure jamais plus d'une semaine.

## Faire la release d'un seul package

On se trouve dans la situation suivante : les développements du package sont terminés et le package est prêt à être publié !

### Vérifications

D'abord, il faut faire qqs vérifications sur le dernier commit de la branche develop
:

1. Les dépendances R ont bien été releasés ({rjd3toolkit}, {rjd3x13}...).
2. Les JARS sont à jour (utiliser la GHA `update-java-dependencies`) à faire dans fork (PR)
3. Le fichier NEWS.md est à jour (PR), (unreleased : pour l'instant à mettre à jour qd release github, apres le CRAN)
4. Le numéro de version minimum des dépendances (Exemple `rjd3toolkit (>= 3.7.1)`): modifier fichier description (PR)
   Lancer releaser::....
6. Les GHA doivent être au vert :
	- Linting
	- R-CMD-Check 
	- Check des links et du fichier NEWS.md (pas encore dans rjd3jars)
7. [CRAN Checker](https://win-builder.r-project.org/upload.aspx) (Win builder) sur les machines du CRAN (ou `devtools::check_win_release()` et `devtools::check_win_oldrelease()`)

Si il y a un problème à un moment de la chaîne, on corrige et on recommence la chaîne de vérification (on retourne au point 1).
Si aucun problème, 0 notes, 0 warnings, 0 errors alors on peut passer à la release !

### Actions de release

#### CRAN

Il faut d'abord faire la release CRAN et ensuite la release GitHub car si le CRAN a des retouches / commentaires sur le package, il faudra refaire une nouvelle release GitHub, modifier le tag / la précédente ou avoir une release GitHub différente de la release (pas acceptable).

7) Récupérer le dernier commit de la branche develop
8) Mettre à jour le numéro de version : passer de la version en cours de développement à la nouvelle version (Exemple passer de `3.6.1.9000` à `3.7.0` ou `4.0.0`)
9) Modifier le fichier cran-comments.md à partir du NEWS.md et commentaires additionnels pour les équipes du CRAN
10) Construire le package au format source (`.tar.gz`)
11) Soumettre le package sur le [site du CRAN]([https://cran.r-project.org/submit.html](https://cran.r-project.org/submit.html))
12) Le CRAN revient rapidement (qqs jours) vers vous avec :
	- Des remarques / commentaires et modifications à faire. Alors il faut modifier le package et on recommence les checks au point 1.
13) Si les développeurs du CRAN vous disent que le package est accepté, alors il sera disponible sur le CRAN (d'abord au format source puis binaire).

#### GitHub

Après release sur le CRAN: maj de cran-comments.md et DESCRIPTION: merge into develop avec PR

Pour faire la release et préparer le branche develop pour la prochaine version
Sur GitHub, on fait appel à la nouvelle GHA `release.yaml` qui va prendre comme argument la branche sur laquelle on fait la release (ici develop) et le futur numéro de version puis exécuter les actions :
	- Valider le nouveau numéro de version
	- Mettre à jour le fichier DESCRIPTION (avec le nouveau numéro de version en developpement)
	- Mettre à jour le fichier NEWS.md: Ajout d'une nouvelle section pour la release (unreleased=> release + develop + ajout de liens (comparer les versions))
	- Création du nouveau tag de la release github
	- Merge intermédiaire de main et develop
	- Push des 2 branches et du nouveau tag
	- Draft de la nouvelle release dans la partie `Releases` de GitHub

Il suffit d'accepter et de publier cette nouvelle release. Et tout est bon !

## Blocages CRAN

Les éléments bloquants étaient principalement : la version de Java du CRAN qui est < 17 et la taille du package {rjd3toolkit} (> 5 Mo).

Les solutions appliquées :

- Création du package {rjd3jars} qui ne contient que le JAR relatif à ProtoBuf (pour l'instant)
- Ajout d'une nouvelle fonction `get_java_version()` pour vérifier la version de Java de l'utilisateur lorsqu'on fait tourner les exemples et les tests

Il y a d'autres blocages mais plus commun aux autres packages R (documentation, exemples...).


# Release 🇬🇧

This document explains how to release an R package smoothly and efficiently.

## Context within an organisation of multiple packages

Releases are carried out ***in a cascade***, starting with the top-level dependency (at the top) and working down to the packages that depend on it. 

Therefore, to release {rjd3workspace}, you must carry out the following in order:

1) The release of {rjd3jars}
2) Release of {rjd3toolkit}
3) Release of {rjd3x13}, {rjd3tramoseats} and {rjd3providers} (order irrelevant)
4) Release of {rjd3workspace}

> [!CAUTION]
> 
> As releases are carried out one by one and cannot be performed simultaneously, during the period between the release of the first package and that of the last package, there may be **operational errors** (on GitHub as well as on CRAN) because the default installed versions will not match.
> 
> But it’s a blessing in disguise; it never lasts more than a week.

## Releasing a single package

We find ourselves in the following situation: development of the package is complete and the package is ready to be published!

### Checks

First, you need to carry out a few checks on the latest commit in the `develop` branch:

1. The R dependencies have been released ({rjd3toolkit}, {rjd3x13}...).
2. The JARs are up to date (use the GHA `update-java-dependencies`)
3. The NEWS.md file is up to date
4. The minimum version number of the dependencies (e.g. `rjd3toolkit (>= 3.7.1)`)
5. The GHA must be green:
	- Linting
	- R-CMD-Check
	- Check of links and the NEWS.md file
6. [CRAN Checker](https://win-builder.r-project.org/upload.aspx) (Win builder) on CRAN machines (or `devtools::check_win_release()` and `devtools::check_win_oldrelease()`)

If there is a problem at any point in the pipeline, we fix it and restart the verification pipeline (we go back to step 1).

If there are no problems, 0 notes, 0 warnings, 0 errors, then we can proceed to the release!

### Release actions

#### CRAN

You must first perform the CRAN release and then the GitHub release, because if CRAN has any feedback or comments on the package, you will need to create a new GitHub release, modify the tag or the previous one, or have a GitHub release that differs from the CRAN release (which is not acceptable).

7) Fetch the latest commit from the `develop` branch
8) Update the version number: change from the current development version to the new version (e.g. change from `3.6.1.9000` to `3.7.0` or `4.0.0`)
9) Edit the `cran-comments.md` file based on `NEWS.md` and add additional comments for the CRAN teams
10) Build the package in source format (`.tar.gz`)
11) Submit the package to the [CRAN website]([https://cran.r-project.org/submit.html](https://cran.r-project.org/submit.html))
12) The CRAN will get back to you quickly (within a few days) with:
	- Comments and changes to be made. You will then need to modify the package and restart the checks from step 1.
13) If the CRAN developers tell you that the package has been accepted, it will be available on the CRAN (first in source format, then as a binary).

#### GitHub

The CRAN version is normally a commit above the `develop` branch: the version number has changed from `.9000` to the stable number.

On GitHub, we use the new GHA `release.yaml`, which takes as arguments the branch on which the release is being made (here, `develop`) and the future version number, then performs the following actions:

- Validate the new version number
- Update the DESCRIPTION file (with the new version number)
- Update the NEWS.md file (add a new section for the release + add links)
- Create the new tag
- Perform an intermediate merge of the `main` and `develop` branches
- Push the two branches and the new tag
- Create a draft of the new release in the `Releases` section of GitHub

All you need to do is accept and publish this new release. And that’s it!

## CRAN Blockers

The main issues were: the Java version on CRAN, which is < 17, and the size of the {rjd3toolkit} package (> 5 MB).

Solutions implemented:

- Creation of the {rjd3jars} package, which currently contains only the ProtoBuf JAR
- Addition of a new function `get_java_version()` to check the user’s Java version when running examples and tests

There are other issues, but these are more common to other R packages (documentation, examples, etc.).
