[← Réécrire et corriger l'historique](06-reecrire-et-corriger-lhistorique.md) · [↑ Sommaire](../README.md#table-des-matières) · [Internes et gros dépôts →](08-internes-et-gros-depots.md)

# 7. Outils du quotidien et récupération

## Stash : mettre de côté du travail en cours

> **Que veut dire « stash » ?** Un *stash* (« réserve, planque ») est un tiroir où Git range temporairement vos modifications en cours pour vous rendre un répertoire de travail propre, par exemple le temps de basculer en urgence sur une autre branche. Vous rangez vos affaires dans le tiroir, vous traitez l'urgence, puis vous ressortez vos affaires intactes. Le mot *pile* désigne ici une organisation « dernier rangé, premier ressorti », comme une pile d'assiettes.

```bash
git stash push -m "WIP refactor du parser"   # stocker les modifs courantes
git stash list                                # lister la pile
git stash show -p stash@{0}                   # voir le diff complet
git stash pop                                 # rejouer le plus récent et le retirer de la pile
git stash apply stash@{2}                     # rejouer un stash spécifique sans le retirer
git stash drop stash@{0}                      # supprimer un stash
git stash clear                               # vider toute la pile
```

Pour stocker aussi les fichiers non suivis :

```bash
git stash push -u -m "WIP avec nouveaux fichiers"
```

Garde-fou : un stash est local et n'est pas poussé sur le serveur. Ne l'utilisez pas comme stockage de longue durée : préférez un commit `wip` sur une branche feature, qui pourra être squashé plus tard.

[Retour en haut de page](#table-des-matières)

## Worktree : plusieurs checkouts en parallèle

> **Que veut dire « worktree » et « checkout » ?** Un *checkout* (« extraction ») est l'action de matérialiser une branche ou un commit donné dans vos fichiers sur le disque. Un *worktree* (« arbre de travail ») est un dossier supplémentaire rattaché au même dépôt, positionné sur une autre branche, sans recopier toute l'histoire. Vous obtenez ainsi deux bureaux de travail côte à côte qui partagent le même classeur d'archives, par exemple un pour votre fonctionnalité et un pour un correctif urgent.

`git worktree` est l'une des fonctionnalités les plus sous-utilisées de Git. Elle remplace avantageusement la danse classique `stash → switch → travail → switch → stash pop` lorsqu'une interruption survient.

### Cas typique

Vous travaillez sur `feat/1234-paiement` quand un hotfix urgent arrive. Trois solutions possibles :

1. **Stash + switch** : `git stash`, basculer, fixer, revenir, `git stash pop`. Risque : oublier le stash, conflits au pop, mélanger les contextes.
2. **Cloner à nouveau le dépôt** : marche, mais long sur un dépôt de plusieurs Go ; double l'espace disque ; deux configurations à maintenir.
3. **`git worktree`** : ajouter un répertoire de travail supplémentaire, qui partage les objets Git mais a son propre checkout. Instantané, économe en disque, propre.

### Commandes essentielles

```bash
# Lister les worktrees existants
git worktree list

# Créer un nouveau worktree pour un hotfix urgent
git worktree add ../projet-hotfix hotfix/1.4.2 origin/main

# Travailler dans le nouveau répertoire
cd ../projet-hotfix
# ... fixer, commiter, pusher ...

# De retour, supprimer le worktree
cd ../projet
git worktree remove ../projet-hotfix

# Nettoyer les worktrees orphelins (si on a effacé le dossier à la main)
git worktree prune
```

### Bénéfices concrets

- **Pas de stash** : votre travail en cours sur `feat/1234-paiement` reste intact dans son dossier.
- **Pas de re-build** : les caches d'IDE, de Maven, de `node_modules` du worktree principal ne sont pas perturbés.
- **Pas de double clone** : les objets Git (commits, blobs, packfiles) sont partagés via `.git/worktrees/`.
- **Compatible avec les IDE modernes** : VS Code, IntelliJ et JetBrains tools ouvrent un worktree comme un projet à part.

### Cas d'usage courants

| Situation | Worktree adapté |
|-----------|-----------------|
| Hotfix urgent pendant un gros refactor | `git worktree add ../proj-hotfix hotfix/x` |
| Comparaison côte-à-côte de deux branches | `git worktree add ../proj-feat-b feat/b` |
| Build CI long pendant qu'on continue à coder | Worktree dédié pour la branche en build |
| Bisect automatisé qui prend du temps | Worktree dédié, le worktree principal reste utilisable |
| Démo client pendant qu'on travaille sur la suite | `git worktree add ../proj-demo v2.4.0` |

### Limites

- Une même branche ne peut être checkée que dans un seul worktree à la fois (sauf `--force`).
- Les hooks Git sont partagés via `.git/hooks/` ; leur exécution dépend du contexte du worktree.
- Les outils qui présupposent un seul checkout (certains scripts artisanaux) peuvent dérouter.

[Retour en haut de page](#table-des-matières)

## Bisect : retrouver le commit fautif

> **Que veut dire « bisect », « recherche dichotomique » et « régression » ?** Une *régression* est un bug qui réapparaît, ou plus largement une fonctionnalité qui marchait et qui s'est cassée. *Bisect* (« couper en deux ») applique une *recherche dichotomique* : à chaque étape, on coupe en deux la liste des commits suspects et on teste le commit du milieu, ce qui élimine la moitié des candidats d'un coup. C'est la méthode du jeu « plus ou moins » : pour deviner un nombre entre 1 et 1000, on propose 500, on apprend si c'est au-dessus ou en dessous, et on recommence. Au lieu de tester 1024 commits un par un, on trouve le coupable en une dizaine d'essais.

Sur une plage de 1024 commits, `git bisect` trouve le coupable en 10 étapes au lieu de 1024.

### Mode interactif

```bash
git bisect start
git bisect bad                   # le commit courant est défectueux
git bisect good v1.3.0           # cette version-là était saine
# Git checkout un commit au milieu, à vous de tester :
# - si le bug est présent : git bisect bad
# - sinon                  : git bisect good
git bisect reset                 # quand c'est fini
```

### Mode automatisé : `git bisect run`

Si vous avez une commande qui retourne 0 quand le code est sain et autre chose sinon :

```bash
git bisect start HEAD v1.3.0
git bisect run npm test -- --testPathPattern panier
```

Git exécute le test à chaque étape et conclut tout seul. C'est l'argument massue en faveur des commits atomiques : un commit qui mélange refactor et fonctionnalité fait perdre la moitié de l'efficacité de bisect.

### Exemple complet de bug-hunt automatisé

Scénario : la fonction `calculerTotal()` retourne le mauvais montant depuis la version 1.5.0, alors qu'elle était correcte en 1.4.0. Plusieurs centaines de commits séparent les deux versions.

#### Étape 1 : écrire un test reproducteur minimal

On crée un script `scripts/bisect-test.sh` :

```bash
#!/usr/bin/env bash
# scripts/bisect-test.sh
# Code de sortie :
#   0   = commit sain
#   1   = commit fautif
#   125 = à ignorer (compilation cassée, dépendances absentes, etc.)

set -u

# (1) Réinstaller les dépendances : leur version peut avoir changé entre les commits
npm ci --prefer-offline --no-audit --silent || exit 125

# (2) S'assurer que le projet compile, sinon ce commit ne nous renseigne pas
npm run build --silent || exit 125

# (3) Lancer LE test ciblé qui isole le bug
if npm test -- --testPathPattern 'panier.calcul.total' --silent; then
    exit 0   # Test passe : commit sain
else
    exit 1   # Test échoue : commit fautif
fi
```

Important : le code 125 indique à `bisect` que le commit n'est pas testable (build cassé indépendant) et qu'il doit l'ignorer plutôt que de le marquer mauvais. C'est ce qui sauve la session si vous tombez sur un commit intermédiaire qui ne compile pas.

#### Étape 2 : lancer la chasse

```bash
chmod +x scripts/bisect-test.sh

# Marquer la dernière version connue saine et la première version connue mauvaise
git bisect start
git bisect bad   v1.5.0
git bisect good  v1.4.0

# Laisser Git automatiser la dichotomie
git bisect run scripts/bisect-test.sh
```

Sur 512 commits suspects, Git effectue 9 itérations (log2(512) = 9). Chaque itération checkout un commit, exécute le script, lit le code de retour, choisit la moitié à explorer.

#### Étape 3 : lire le verdict

```text
9c4f7e1 is the first bad commit
commit 9c4f7e1
Author: Alice <alice@example.com>
Date:   Tue Mar 12 14:32:18 2024 +0100

    refactor(panier): extraction du calcul TVA

 src/panier/total.ts | 14 +++++++-------
 1 file changed, 7 insertions(+), 7 deletions(-)
```

Git nomme le commit fautif. On termine la session :

```bash
git bisect reset
```

#### Étape 4 : astuces avancées

```bash
# Sauter un commit qu'on sait défectueux pour une raison non liée
git bisect skip

# Visualiser le sous-ensemble restant à explorer
git bisect log
git bisect visualize

# Reproduire la session ailleurs (script, autre poste)
git bisect log > bisect-session.log
git bisect replay bisect-session.log

# Limiter la dichotomie à un sous-répertoire
git bisect start -- src/panier/

# Combiner avec un worktree pour ne pas geler le checkout principal
git worktree add ../projet-bisect HEAD
cd ../projet-bisect
git bisect start
# ...
```

#### Pourquoi ça marche si bien

L'efficacité de `bisect run` repose sur trois conditions :

1. Un test reproductible qui ne dépend pas d'un état extérieur fragile (date, réseau, secret).
2. Un build qui se relance proprement à chaque commit (utiliser `npm ci`, pas `npm install`).
3. Des commits atomiques. Un seul commit fautif est trouvé par dichotomie ; si ce commit fait dix choses à la fois, vous savez juste *où* mais pas *quoi*.

[Retour en haut de page](#table-des-matières)

## Reflog : la machine à remonter le temps

> **Que veut dire « reflog » ?** Le *reflog* (contraction de *reference log*, « journal des références ») est un carnet de bord local où Git note chaque déplacement de `HEAD` et de vos branches : chaque commit, reset, rebase, changement de branche. Conservé environ 90 jours, il permet de retrouver un commit même si plus aucune branche ne pointe dessus. C'est la boîte noire de l'avion : même après un crash, elle garde la trace de tout ce qui s'est passé.

> **Que veut dire « dangling commit » et « garbage collection » ?** Un *dangling commit* (« commit pendant, orphelin ») est un commit que plus aucune branche ni aucun tag ne désigne ; il flotte sans attache. La *garbage collection* (« ramassage des ordures », commande `git gc`) est le ménage automatique qui finit par supprimer ces objets devenus inaccessibles pour libérer de l'espace, un peu comme la corbeille de votre système qui se vide au bout d'un moment.

Le reflog est la bouée de sauvetage en cas de manipulation hasardeuse (`reset --hard`, `rebase` cassé, branche supprimée par erreur).

```bash
git reflog                       # historique de HEAD
git reflog show ma-branche       # historique d'une branche précise

# Récupérer un commit perdu après un reset --hard
git reset --hard HEAD@{1}

# Recréer une branche supprimée
git switch -c ma-branche-restored <SHA récupéré dans le reflog>
```

Tant qu'un commit apparaît dans le reflog, ses données sont accessibles. Au-delà de 90 jours (configurable via `gc.reflogExpire`), le garbage collector peut les supprimer définitivement.

### Scénarios de récupération concrets

#### Récupérer après un `git reset --hard` malencontreux

```bash
# Avant le drame
git log --oneline
# 3e1f7c2 (HEAD -> main) feat(panier): coupons cumulables
# a91d3b4 feat(panier): squelette
# 7c8e0a1 chore: bump deps

# Le drame
git reset --hard HEAD~3   # adieu trois commits

# Le secours
git reflog
# 7c8e0a1 (HEAD -> main) HEAD@{0}: reset: moving to HEAD~3
# 3e1f7c2 HEAD@{1}: commit: feat(panier): coupons cumulables
# a91d3b4 HEAD@{2}: commit: feat(panier): squelette

# Restaurer
git reset --hard HEAD@{1}
```

Tout est revenu. Le reflog est local : si vous travaillez sur un autre poste qui n'a pas le reflog, vous êtes coincé. D'où l'intérêt de pousser régulièrement sur une branche de sauvegarde.

#### Récupérer une branche supprimée

```bash
# La branche feat/1234 existait, je l'ai effacée
git branch -D feat/1234

# Retrouver son dernier SHA via le reflog
git reflog | grep feat/1234
# 9c4f7e1 HEAD@{14}: checkout: moving from feat/1234 to main

# Recréer la branche
git switch -c feat/1234 9c4f7e1
```

#### Récupérer un commit perdu après un rebase raté

```bash
# Avant : trois commits sur ma feature
# Après un rebase -i où j'ai accidentellement remplacé "pick" par "drop"

git reflog
# 9c4f7e1 HEAD@{0}: rebase finished: returning to refs/heads/feat/x
# ...
# a91d3b4 HEAD@{6}: commit: feat: ajout du test e2e   <- perdu après le rebase

# Cherry-pick du commit perdu
git cherry-pick a91d3b4
```

#### Trouver un commit orphelin (dangling)

```bash
# Lister tous les commits non référencés mais encore présents
git fsck --lost-found

# dangling commit a91d3b4...
# dangling blob  ...

# Inspecter le contenu
git show a91d3b4
git switch -c sauvetage a91d3b4
```

### Étendre la durée de rétention

Pour une mission longue ou une enquête forensique, on peut prolonger la durée de vie du reflog :

```bash
git config --global gc.reflogExpire 365.days
git config --global gc.reflogExpireUnreachable 90.days
```

Le coût en stockage est négligeable sauf sur des dépôts immenses.

[Retour en haut de page](#table-des-matières)

---

[← Réécrire et corriger l'historique](06-reecrire-et-corriger-lhistorique.md) · [↑ Sommaire](../README.md#table-des-matières) · [Internes et gros dépôts →](08-internes-et-gros-depots.md)
