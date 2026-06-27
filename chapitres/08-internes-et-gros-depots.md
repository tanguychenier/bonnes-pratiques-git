[← Outils du quotidien et récupération](07-outils-du-quotidien-et-recuperation.md) · [↑ Sommaire](../README.md#table-des-matières) · [Configuration du dépôt et sécurité des fichiers →](09-configuration-du-depot-et-securite-des-fichiers.md)

# 8. Internes et gros dépôts

## Internes Git : objets, refs, packfiles

Connaître la mécanique interne de Git n'est presque jamais utile au quotidien, mais devient précieux dès qu'il faut diagnostiquer un dépôt qui « rame » (devient lent), un push qui n'aboutit pas ou une maintenance d'urgence.

### La base d'objets

Tous les objets Git vivent sous `.git/objects/`. Chaque objet est compressé avec zlib, identifié par le SHA-1 (ou SHA-256) de son contenu, et rangé dans un fichier dont le chemin reprend les deux premiers caractères de l'empreinte :

> **Que veut dire « zlib » ?** zlib est une bibliothèque de compression de données très répandue. Compresser, c'est réduire la taille d'un fichier en éliminant les redondances, comme on tasse des vêtements dans une valise sous vide. Git compresse chaque objet pour économiser de l'espace disque.

```text
.git/objects/3e/1f7c2a91d3b4...   <- contenu zlib
```

Quatre types d'objets :

| Type | Contenu | Inspection |
|------|---------|-----------|
| **blob** | Le contenu brut d'un fichier (sans nom, sans permissions). | `git cat-file -p <SHA>` |
| **tree** | La liste « nom + permissions + SHA blob/tree » d'un répertoire. | `git cat-file -p <SHA>` |
| **commit** | Pointeur vers un tree, parents, auteur, committer, message. | `git cat-file -p <SHA>` |
| **tag annoté** | Pointeur vers un commit (ou un autre objet), auteur, message, signature. | `git cat-file -p <SHA>` |

```bash
# Type d'un objet
git cat-file -t 3e1f7c2

# Contenu lisible
git cat-file -p HEAD
# tree 4f9a2b8c...
# parent a91d3b4...
# author Alice <alice@example.com> 1735689600 +0100
# committer Alice <alice@example.com> 1735689600 +0100
#
# feat(panier): coupons cumulables

# Le tree pointé par le commit
git cat-file -p HEAD^{tree}
# 100644 blob 7c8e0a1... package.json
# 040000 tree 9c4f7e1... src
```

### Les références (refs)

Sous `.git/refs/`, des fichiers texte contiennent simplement un SHA :

```text
.git/refs/heads/main           <- 3e1f7c2a91d3b4...
.git/refs/heads/feat/1234      <- a91d3b47c8e0a1...
.git/refs/tags/v1.4.0          <- 9c4f7e1f4b9d22...
.git/refs/remotes/origin/main  <- 3e1f7c2a91d3b4...
```

`HEAD` est un cas spécial : c'est un fichier `.git/HEAD` qui contient soit un SHA (état détaché) soit `ref: refs/heads/main` (pointe vers une branche).

Pour économiser de l'espace, Git regroupe parfois ces fichiers dans `.git/packed-refs`.

### Les packfiles

Stocker chaque objet dans son propre fichier devient inefficace quand un dépôt en compte des centaines de milliers. Git regroupe alors les objets dans des *packfiles* (`.git/objects/pack/pack-*.pack`), accompagnés d'un index (`.idx`) pour les retrouver vite.

> **Que veut dire « packfile » et « delta » ?** Un *packfile* (« fichier compacté ») est un gros fichier qui rassemble des milliers d'objets, plus économe que des milliers de petits fichiers. À l'intérieur, les versions successives d'un même fichier ne sont pas recopiées en entier : Git ne garde qu'un *delta*, c'est-à-dire la différence d'une version à la suivante. Plutôt que de réimprimer tout un livre à chaque correction, on ne note que les pages modifiées.

```bash
# Forcer un packing
git gc

# Plus agressif : recompacter tout
git gc --aggressive --prune=now

# Inspecter les packfiles
git verify-pack -v .git/objects/pack/pack-*.idx | sort -k 3 -n | tail -20
# Liste les 20 plus gros objets, utile pour traquer un binaire qui pèse 500 Mo
```

### Diagnostiquer un dépôt lent

| Symptôme | Cause probable | Diagnostic |
|----------|----------------|-----------|
| `git status` lent | Beaucoup de fichiers non suivis ou hooks lents. | `git status -uno`, `core.untrackedCache=true`, `core.fsmonitor=true`. |
| `git log` lent | Reflog géant ou packfiles non optimisés. | `git gc`, `git reflog expire`. |
| `git push` lent | Gros binaire dans l'historique. | `git verify-pack` pour identifier ; `git filter-repo` pour purger. |
| Clone très long | Histoire profonde, beaucoup d'objets. | `git clone --depth=1` (clone superficiel) ou `--filter=blob:none` (partial clone). |
| `.git` énorme | Binaires non en LFS, historique non purgé. | LFS, `git filter-repo`. |

### Maintenance régulière

Git 2.30+ propose `git maintenance`, qui automatise les tâches de gc, prefetch, packing :

```bash
# Activer la maintenance automatique en arrière-plan
git maintenance start

# Lancer ponctuellement
git maintenance run --task=gc --task=commit-graph
```

Sur un dépôt actif, exécuter `git gc` une fois par semaine suffit en général à maintenir des performances correctes.

[Retour en haut de page](#table-des-matières)

## Gros dépôts : LFS, sparse checkout, partial clone

Les équipes modernes manipulent des monorepos de 10, 50 voire plusieurs centaines de Go (jeu vidéo, ML, design assets). Git seul ne tient pas la charge ; trois mécanismes complémentaires existent.

### Git LFS : déporter les gros binaires

> **Que veut dire « Git LFS » ?** LFS veut dire *Large File Storage*, « stockage de gros fichiers ». C'est une extension de Git qui, au lieu de mettre un gros fichier binaire (image, vidéo, modèle d'apprentissage automatique, exécutable) dans le dépôt, n'y range qu'un petit *pointeur texte* indiquant où trouver le vrai fichier sur un serveur dédié. C'est le principe d'un vestiaire : vous gardez le ticket (le pointeur) dans votre poche, et le gros manteau (le binaire) reste à part. *Binaire* désigne ici un fichier non textuel, illisible à l'œil, comme une photo ou une vidéo.

Sans LFS, chaque modification d'un gros binaire est stockée en delta dans la base d'objets : un dépôt avec 100 versions d'une vidéo de 500 Mo finit à 50 Go. Avec LFS, le binaire actuel pèse seulement son poids réel, et l'historique pointe vers les versions distantes.

```bash
# Installer (une fois par poste)
git lfs install

# Déclarer les types de fichiers à stocker en LFS
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "*.bin"
git lfs track "*.mp4"

# Cela écrit dans .gitattributes :
# *.psd filter=lfs diff=lfs merge=lfs -text

# Commiter normalement
git add .gitattributes design.psd
git commit -m "chore: ajout du PSD via LFS"
git push
```

Le dépôt Git ne contient qu'un pointeur texte (~130 octets) ; le binaire vit sur le serveur LFS de GitHub / GitLab / un serveur self-hosted.

#### Pièges LFS

- **Migrer un dépôt existant** : `git lfs migrate import --include="*.psd" --everything` réécrit l'historique. Force-push obligatoire.
- **Coût** : GitHub facture le stockage et la bande passante LFS au-delà du quota gratuit (1 Go offert). Surveiller la consommation.
- **Clone partiel par défaut** : `GIT_LFS_SKIP_SMUDGE=1 git clone ...` pour récupérer les pointeurs sans télécharger les binaires immédiatement (utile en CI qui n'en a pas besoin).
- **Pas adapté à tout** : pour quelques fichiers de quelques Mo, LFS est superflu. Réservez-le aux binaires de plusieurs dizaines de Mo qui changent souvent.

### Partial clone : ne télécharger que ce qu'on lit

> **Que veut dire « clone » et « partial clone » ?** *Cloner* un dépôt, c'est en télécharger une copie complète sur votre machine. Un *partial clone* (« clone partiel ») télécharge seulement une partie au départ (souvent l'histoire et les dossiers, mais pas le contenu de tous les fichiers) et récupère le reste au fur et à mesure que vous en avez besoin. C'est comme commander un meuble livré d'abord en kit léger, les pièces lourdes n'arrivant qu'au moment où vous les montez.

```bash
# Clone sans aucun blob (les arbres et commits sont là, le contenu des fichiers vient au checkout)
git clone --filter=blob:none git@github.com:org/monorepo.git

# Clone sans les blobs > 1 Mo
git clone --filter=blob:limit=1m git@github.com:org/monorepo.git

# Clone sans aucun arbre (encore plus radical)
git clone --filter=tree:0 git@github.com:org/monorepo.git
```

Au premier `git checkout` sur une branche, Git récupère à la volée les blobs nécessaires. Sur un monorepo de 50 Go, le gain de temps de clone passe de 30 minutes à 30 secondes.

### Sparse checkout : ne déployer qu'une partie de l'arbre

> **Que veut dire « sparse checkout » ?** *Sparse checkout* veut dire « extraction partielle » (*sparse* = « clairsemé »). Ce mécanisme ne fait apparaître sur votre disque qu'une partie choisie des dossiers du dépôt, tout en gardant l'historique complet. Sur un monorepo géant, vous ne déballez que les rayons qui vous concernent, même si l'entrepôt entier reste accessible.

Combiné avec un partial clone, on obtient l'expérience d'un dépôt léger sur disque malgré une histoire massive en amont.

```bash
# Activer
git sparse-checkout init --cone

# Déclarer les répertoires qu'on veut voir
git sparse-checkout set apps/web libs/ui libs/shared

# Vérifier ce qui est matérialisé
git sparse-checkout list

# Désactiver (récupérer tout)
git sparse-checkout disable
```

Le mode `--cone` est plus rapide et plus prévisible que le mode complet (basé sur des motifs `.gitignore`), au prix d'une expressivité réduite (on ne peut sélectionner que des sous-répertoires entiers).

### Combo gagnant pour un monorepo

```bash
# Cloner intelligemment un monorepo de 50 Go
git clone --filter=blob:none --no-checkout git@github.com:org/monorepo.git
cd monorepo
git sparse-checkout init --cone
git sparse-checkout set apps/mon-app libs/utilises-par-mon-app
git checkout main
```

Résultat : sur disque, seuls les répertoires nécessaires sont déployés. L'historique est complet (on peut faire `git log`, `git blame`), les blobs absents seront récupérés à la demande.

### Stratégies complémentaires

| Stratégie | Utile pour |
|-----------|-----------|
| `git clone --depth=1` (*shallow clone*, « clone superficiel » : ne récupère que le dernier commit, sans l'historique passé) | CI, déploiements, agents qui n'ont pas besoin de l'historique. |
| `git fetch --filter=blob:none` | Réduire la bande passante des fetchs incrémentaux. |
| Submodules | À éviter sauf besoin précis : expérience utilisateur déroutante. |
| Subtree | Vendoring d'une dépendance externe : alternative aux submodules, sans état détaché. |
| Scalar / VFS for Git | Solutions Microsoft pour des dépôts énormes (Windows source). |

[Retour en haut de page](#table-des-matières)

---

[← Outils du quotidien et récupération](07-outils-du-quotidien-et-recuperation.md) · [↑ Sommaire](../README.md#table-des-matières) · [Configuration du dépôt et sécurité des fichiers →](09-configuration-du-depot-et-securite-des-fichiers.md)
