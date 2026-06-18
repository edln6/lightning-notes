# Initialisation d'un package R 🇫🇷

Lorsqu'on charge un package, généralement, on utilise la fonction `library()`. Mais pour des packages utilisant des dépendances ou des initialisations particulières, il faut rentrer dans les détails de l'initialisation.

## "Attaching" and "Loading"

La différence entre "Attaching" (attaché ou indexé) et "Loading" (chargé) est décrite ici :https://r-pkgs.org/dependencies-mindset-background.html#sec-dependencies-attach-vs-load.

Ce qu'il faut retenir, c'est que :

- "Loading" prépare le package en mémoire (toutes les classes, les fonctions, les data...)
- "Attaching" indexe le package et ses fonctionnalités à l'environnement R

## Trigger Event

Quand est ce qu'un package est "Attached" ou "Loaded" ?

### "Loading"

Voilà différents cas de figure qui charge le package en mémoire :

- dès que le package est "attached"
- `devtools::load_all()`
- `mon_package::ma_fonction()`
- `loadNamespace("mon_package")`
- `requireNamespace("mon_package")`

### "Attaching"

Voilà différents cas de figure qui indexe le package dans l'environnement :

- `library("mon_package")`
- `require("mon_package")`

## Utilisation

Mais quand faut attacher un package ou juste le charger ?

Du point de vue développeur, un package dépendance ne doit jamais être attaché mais uniquement chargé.

## Que mettre dans le `.onAttach` et le `.onLoad` ?

[Ref R Packages](https://r-pkgs.org/code.html#sec-code-onLoad-onAttach)
Étant donné que l'utilisateur peut utiliser notre package à partir de `library("mon_package")` mais aussi à partir de `mon_package::ma_fonction()`, il faut que les informations minimales à l'utilisation du package soient dans le `.onLoad()`.

### `.onLoad`

Le `.onLoad`  doit contenir :
- La préparation des options (si il y en a)
- le chargement des jars (si dépendance Java)
- le chargement des proto (si il y en a)

### `.onAttach`

Dans le .onAttach, on va plutôt mettre des messages informatifs à l'utilisateur par rapport à son environnement.

### Et Java

Sachant qu'un utilisateur de R pourrait appeler un package sans l'attacher avec `::` (exemple : `rjd3x13::x13(AirPassengers)`), ne faudrait-il pas mettre la condition sur la version de Java (`Java >= 21`) dans le `.onLoad()` et non le `.onAttach()` et de mettre une erreur ?

1) Non pour l'erreur car sinon, un utilisateur ne pourrait pas installer le package sans avoir `Java >= 21` et ce n'est pas ce que l'on veut. On veut que l'utilisateur puisse quand même installer le package et si besoin mettre à jour son Java a posteriori. Aussi cela empêcherai les checks du CRAN.
2) Oui pour le `.onLoad()` car n'importe quell utilisateur doit pouvoir utiliser le package (avec `library()` ou avec `::`.

## Dépendances

La page du livre R packages est bien développée à ce sujet : https://r-pkgs.org/dependencies-mindset-background.html

### Imports vs Depends vs Suggests vs LinkingTo vs Enhances

Il existe 5 moyens de déclarer des dépendances en R. 

- **Depends** : liste les dépendances qui **seront attachées**. A priori, il n'y a **aucune raison** de mettre des packages dans cette section (on privilégiera la section Imports).
- **Imports** : liste les dépendances dont l'utilisateur a besoin pour faire fonctionner le package. Ces packages **sont installés et chargés** ("Loading") en même temps que le package principal.
- **Suggests** : liste les dépendances dont le développeur a besoin pour développer ou utiliser des fonctionnalités avancées du packages (tests, vignettes). Ces packages **ne sont pas** automatiquement installé en même temps que le package principal.
- **LinkingTo** : Pour le code C (inutile ici) 
- **Enhances** : Concept compliqué. Pas utilisé ici.

### Dans le zzz.R

Par défault, les packages dépendances (dans la section **Imports** du fichier DESCRIPTION) sont chargés ("Loaded") lors du chargement du package principal (mais uniquement dans certaines conditions, voir [discussion 161](https://github.com/rjdverse/rjd3toolkit/discussions/161)). Il n'y a donc pas besoin de recharger à la main un package dépendance. 

Si une dépendance est manquante R le signifiera à l'utilisateur avec  le message :

```
Error in loadNamespace(i, c(lib.loc, .libPaths()), versionCheck = vI[[i]]) : 
  there is no package called 'rjd3toolkit'
```

## Schéma rjdverse

Pour le package {rjd3x13}, voilà un extrait du fichier DESCRIPTION :

```
Depends:
    R (>= 4.1.0)
Imports:
    rJava (>= 1.0-6),
    RProtoBuf (>= 0.4.20),
    rjd3toolkit (>= 3.7.1)
```

On liste ici les packages dont on aura besoin (et qui seront installés) et la version de R minimale. 

Dans le fichier zzz.R, on retrouve :

- Pas de `.onAttach()` 

- un `.onLoad()` avec le chargement des Jars, des extracteurs de {rjd3toolkit}, des classes ProtoBuf et de quelques options : 

```r
#' @importFrom RProtoBuf readProtoFiles2
#' @importFrom rJava .jpackage
#' @importFrom rjd3toolkit get_java_version minimal_java_version
#' @importFrom rjd3toolkit reload_dictionaries
.onLoad <- function(libname, pkgname) {
	# Chargement des packages dépendances
	if (!requireNamespace("rjd3toolkit", quietly = TRUE)) {
		stop("Loading {rjd3toolkit} failed", call. = FALSE)
	}
	
    # Chargement des classes Java
    jars_inst <- file.path(libname, pkgname, "inst", "java") |>
        list.files(pattern = "\\.jar$", full.names = TRUE, all.files = TRUE)
    result <- rJava::.jpackage(
        pkgname,
        lib.loc = libname,
        morePaths = jars_inst
    )
    if (!result) {
        stop("Loading java packages failed", call. = FALSE)
    }
        
    # Chargement des classes proto
    proto.dir <- system.file("proto", package = pkgname)
    RProtoBuf::readProtoFiles2(protoPath = proto.dir)

    # Définition des options
    if (is.null(getOption("summary_info"))) {
        options(summary_info = TRUE)
    }
    if (is.null(getOption("thresholds_pval"))) {
        options(
            thresholds_pval = c(
                Severe = 0.001,
                Bad = 0.01,
                Uncertain = 0.05,
                Good = Inf
            )
        )
    }
}
```

Le `.onLoad()` de {rjd3jars} contient un warning par rapport à la version de Java:

```r
#' @importFrom rJava .jpackage
.onLoad <- function(libname, pkgname) {
	# Vérification de la version de Java
    has_java <- check_java_version()
    
    # Chargement des extracteurs
    if (has_java) {
        rjd3toolkit::reload_dictionaries()
    }
    
	# Chargement des classes Java
    jars_inst <- file.path(libname, pkgname, "inst", "java") |>
        list.files(pattern = "\\.jar$", full.names = TRUE, all.files = TRUE)
    result <- rJava::.jpackage(
        pkgname,
        lib.loc = libname,
        morePaths = jars_inst
    )
    if (!result) {
        stop("Loading java packages failed", call. = FALSE)
    }

	# Définition des options
    if (is.null(getOption("summary_info"))) {
        options(summary_info = TRUE)
    }
}
```

# Initialize a R package 🇬🇧

When loading a package, you typically use the `library()` function. However, for packages that use dependencies or require specific initialization steps, you need to delve into the details of the initialization process.
## "Attaching" and "Loading"

The difference between "Attaching" (attached or indexed) and "Loading" (loaded) is described here: https://r-pkgs.org/dependencies-mindset-background.html#sec-dependencies-attach-vs-load. 

The key points to remember are:

- "Loading" prepares the package in memory (all classes, functions, data, etc.)
- "Attaching" indexes the package and its features to the R environment

## Trigger Event

When is a package "Attached" or "Loaded"?

### "Loading"

Here are different scenarios that load the package into memory:

- as soon as the package is "attached"
- `devtools::load_all()`
- `my_package::my_function()`
- `loadNamespace("my_package")`
- `requireNamespace("my_package")`

### "Attaching"

Here are different scenarios that "attach" the package in the environment:

- `library("my_package")`
- `require("my_package")`

## Usage

But when should you attach a package, and when should you just load it?

From a developer’s perspective, a dependency package should never be attached—only loaded.

## What should be included in `.onAttach` and `.onLoad`?

Ref: [R Packages](https://r-pkgs.org/code.html#sec-code-onLoad-onAttach)

Since users can use our package via `library("my_package")` as well as via `my_package::my_function()`, the minimum information required to use the package must be included in `.onLoad()`.

### `.onLoad`

The `.onLoad` function must contain:

- Configuration of options (if any)
- Loading of JAR files (if there are Java dependencies)
- Loading of Protobuf files (if any)

### `.onAttach`

In the `.onAttach` function, we will primarily include informational messages for the user regarding their environment.

### And Java

Given that an R user could call a package without attaching it with `::` (example: `rjd3x13::x13(AirPassengers)`), shouldn’t we put the Java version check (`Java >= 21`) in `.onLoad()` instead of `.onAttach()` and throw an error?

1) No to the error, because otherwise a user wouldn’t be able to install the package without having `Java >= 21`, and that’s not what we want. We want the user to still be able to install the package and, if necessary, update their Java later. Also, this would prevent the CRAN checks.

2) Yes for `.onLoad()` because any user should be able to use the package (with `library()` or with `::`).

## Dependencies

The R packages book page covers this topic well: https://r-pkgs.org/dependencies-mindset-background.html

### Imports vs Depends vs Suggests vs LinkingTo vs Enhances

There are five ways to declare dependencies in R.

- **Depends**: lists the dependencies that **will be attached**. In principle, there is **no reason** to put packages in this section (the Imports section is preferred).
- **Imports**: lists the dependencies the user needs to run the package. These packages **are installed and loaded** (“Loading”) at the same time as the main package.
- **Suggests**: lists the dependencies the developer needs to develop or use advanced features of the package (tests, vignettes). These packages **are not** automatically installed at the same time as the main package.
- **LinkingTo**: For C code (not used here)
- **Enhances**: A complicated concept. Not used here.

### In zzz.R

By default, dependency packages (in the **Imports** section of the DESCRIPTION file) are loaded ("Loading") when the main package is loaded (but only under certain conditions; see [discussion 161](https://github.com/rjdverse/rjd3toolkit/discussions/161)). Therefore, there is no need to manually reload a dependency package.

If a dependency is missing, R will notify the user with the message:

```
Error in loadNamespace(i, c(lib.loc, .libPaths()), versionCheck = vI[[i]]) :
there is no package called ‘rjd3toolkit’
```

## rjdverse Schema

For the {rjd3x13} package, here is an except from the DESCRIPTION file:

```
Depends:
	R (>= 4.1.0)
Imports:
	rJava (>= 1.0-6),
	RProtoBuf (>= 0.4.20),
	rjd3toolkit (>= 3.7.1)
```

Here we list the packages we will need (and which will be installed) and the minimum R version.

In the zzz.R file, we find:

- No `.onAttach()`
- a `.onLoad()` that loads the JARs, the {rjd3toolkit} extractors, the ProtoBuf classes, and a few options:

```r
#' @importFrom RProtoBuf readProtoFiles2
#' @importFrom rJava .jpackage
#' @importFrom rjd3toolkit get_java_version minimal_java_version
#' @importFrom rjd3toolkit reload_dictionaries
.onLoad <- function(libname, pkgname) {
	# Loading dependency packages
	if (!requireNamespace("rjd3toolkit", quietly = TRUE)) {
		stop("Loading {rjd3toolkit} failed", call. = FALSE)
	}
	
    # Loading Java class
    jars_inst <- file.path(libname, pkgname, "inst", "java") |>
        list.files(pattern = "\\.jar$", full.names = TRUE, all.files = TRUE)
    result <- rJava::.jpackage(
        pkgname,
        lib.loc = libname,
        morePaths = jars_inst
    )
    if (!result) {
        stop("Loading java packages failed", call. = FALSE)
    }
    
    # Loading proto class
    proto.dir <- system.file("proto", package = pkgname)
    RProtoBuf::readProtoFiles2(protoPath = proto.dir)

    # Options setup
    if (is.null(getOption("summary_info"))) {
        options(summary_info = TRUE)
    }
    if (is.null(getOption("thresholds_pval"))) {
        options(
            thresholds_pval = c(
                Severe = 0.001,
                Bad = 0.01,
                Uncertain = 0.05,
                Good = Inf
            )
        )
    }
}
```

The `.onLoad()` method in {rjd3jars} and contains a warning regarding the Java version:

```r
#' @importFrom rJava .jpackage
.onLoad <- function(libname, pkgname) {
	# Check Java version
    has_java <- check_java_version()
    
    # Loading extractors
    if (has_java) {
        rjd3toolkit::reload_dictionaries()
    }
    
	# Loading Java class
    jars_inst <- file.path(libname, pkgname, "inst", "java") |>
        list.files(pattern = "\\.jar$", full.names = TRUE, all.files = TRUE)
    result <- rJava::.jpackage(
        pkgname,
        lib.loc = libname,
        morePaths = jars_inst
    )
    if (!result) {
        stop("Loading java packages failed", call. = FALSE)
    }

	# Options setup
    if (is.null(getOption("summary_info"))) {
        options(summary_info = TRUE)
    }
}
```

