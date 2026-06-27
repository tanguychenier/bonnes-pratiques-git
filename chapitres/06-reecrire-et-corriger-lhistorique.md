[← Versions et signatures](05-versions-et-signatures.md) · [↑ Sommaire](../README.md#table-des-matières) · [Outils du quotidien et récupération →](07-outils-du-quotidien-et-recuperation.md)

# 6. Réécrire et corriger l'historique

## Squash

> **Que veut dire « squash » ?** *Squash* (« écraser, aplatir ») consiste à fondre plusieurs commits en un seul. Pendant le développement, on accumule souvent des commits de tâtonnement (« WIP » pour *Work In Progress*, « travail en cours », « fix typo », « ça marche enfin ») ; les écraser produit un commit unique et lisible qui raconte proprement la fonctionnalité. C'est passer de dix brouillons griffonnés à une seule page recopiée au propre.

### Squash interactif local

```bash
git rebase -i origin/main
```

Dans l'éditeur, on remplace `pick` par `squash` (ou `s`) sur les commits à fondre dans le précédent. La variante `fixup` (ou `f`) fait la même chose mais jette le message au lieu de le concaténer :

```text
pick   3e1f7c2 feat(panier): squelette du panier
squash a91d3b4 wip
fixup  7c8e0a1 corrige test cassé
squash f4b9d22 ajout du test e2e
```

Git propose ensuite de réécrire le message final.

### Squash à la fusion

La plupart des plateformes (GitHub, GitLab, Bitbucket) offrent une option *Squash and merge* qui fait la même chose au moment d'intégrer la PR. Elle est généralement préférable parce qu'elle préserve les commits individuels sur la branche feature et n'écrit qu'un commit propre dans `main`. La case « Default to PR title and description » côté GitHub évite l'écueil du message squashé qui répète mécaniquement les sujets de chaque commit.

### Quand ne pas squasher

- Branches partagées (réécrire l'historique commun perturbe les autres contributeurs).
- Travail composé d'étapes vraiment distinctes qui méritent chacune un `git revert` ou `git bisect` indépendant.
- PR de gros refactor où la séquence des commits raconte la progression d'une transformation (extract → rename → inline).

[Retour en haut de page](#table-des-matières)

## Réécrire l'historique en sécurité

Réécrire l'historique (rebase, squash, amend, filter-repo) crée de nouveaux commits à la place des anciens. Bien fait, c'est un atout ; mal fait, c'est une catastrophe partagée.

### Quand le force-push est acceptable

| Contexte | Force-push autorisé ? |
|----------|-----------------------|
| Branche personnelle sur laquelle vous êtes seul à travailler | Oui, avec `--force-with-lease`. |
| Branche de PR, après rebase ou squash, avant merge | Oui, avec `--force-with-lease`. Prévenir si la PR est en cours de revue. |
| Branche partagée par plusieurs développeurs | Non. Préférer un nouveau commit (`revert`) ou une nouvelle branche. |
| `main`, `master`, `develop`, `release/*` | **Jamais.** Ces branches doivent être protégées au niveau de la plateforme. |

### `--force-with-lease` vs `--force`

```bash
# Dangereux : écrase tout, même les commits qu'un collègue vient de pousser.
git push --force

# Recommandé : refuse le push si la branche distante a bougé depuis votre dernier fetch.
git push --force-with-lease
```

Avec `push.useForceIfIncludes = true` (cf. [Configuration initiale](#configuration-initiale)), Git ajoute un garde-fou supplémentaire qui vérifie aussi que vos refs locales reflètent bien l'état distant.

### `git commit --amend`

Modifie le dernier commit (message ou contenu). Tant qu'il n'est pas poussé, c'est anodin.

```bash
git commit --amend                  # rouvre l'éditeur sur le dernier message
git commit --amend --no-edit        # ajoute les fichiers staged au dernier commit, message inchangé
```

Une fois poussé, un amend nécessite un force-push, donc les mêmes précautions qu'un rebase.

### `git filter-repo` : réécriture massive

Pour purger un fichier de tout l'historique, renommer un auteur sur cent commits, ou extraire un sous-répertoire en dépôt indépendant. Voir aussi la section [Fichiers sensibles](#fichiers-sensibles) pour le cas particulier des secrets.

[Retour en haut de page](#table-des-matières)

## Cherry-pick, revert, reset : choisir le bon outil

> **Que veut dire « cherry-pick » ?** *Cherry-pick* (« cueillir la cerise ») consiste à prendre un seul commit, où qu'il soit, et à le rejouer sur la branche courante. Git recrée un commit au contenu identique mais avec une nouvelle empreinte SHA. Image : aller piocher une seule recette précise dans le carnet d'un collègue pour la recopier dans le vôtre, sans emporter tout le carnet.

> **Que veut dire « backporter » ?** *Backporter* (« reporter en arrière ») signifie appliquer une correction faite sur une version récente à une version plus ancienne encore utilisée. Par exemple, corriger un bug sur la version 2.0 puis reporter ce même correctif sur la 1.4 qu'un client n'a pas encore quittée.

### `git cherry-pick`

Utile pour :

- Backporter un fix de `main` vers une branche `release/1.4` maintenue.
- Récupérer un commit isolé d'une branche abandonnée.

```bash
git switch release/1.4
git cherry-pick 3e1f7c2
git cherry-pick 3e1f7c2..a91d3b4   # une plage de commits
```

À éviter pour des dizaines de commits : préférer un merge ou un rebase complet.

### `git revert`

Crée un nouveau commit qui annule les changements d'un commit antérieur. Sans réécriture d'historique, donc sûr sur les branches partagées.

```bash
git revert 3e1f7c2                     # un commit
git revert -m 1 <SHA-merge>            # annuler un merge (le 1 désigne le parent à conserver)
```

C'est la méthode standard pour défaire un commit déjà poussé sur `main`.

### `git reset` : trois modes

> **Que veut dire « index » (ou *staging area*) et « répertoire de travail » ?** Git distingue trois zones. Le *répertoire de travail* (en anglais *working tree*) est l'ensemble des fichiers tels que vous les éditez sur le disque. L'*index* (aussi appelé *staging area*, « zone de préparation ») est un sas intermédiaire où vous placez ce qui partira au prochain commit, via `git add`. Le *commit* est l'enregistrement final. Image : le répertoire de travail est votre bureau en désordre, l'index est le carton dans lequel vous rangez ce que vous voulez expédier, et le commit est le colis posté.

| Mode | Effet sur HEAD | Effet sur l'index | Effet sur le répertoire de travail |
|------|----------------|-------------------|-----------------------------------|
| `--soft` | Déplacé | Inchangé (les modifs restent staged) | Inchangé |
| `--mixed` (défaut) | Déplacé | Réinitialisé (les modifs deviennent unstaged) | Inchangé |
| `--hard` | Déplacé | Réinitialisé | **Réinitialisé : modifications perdues** |

```bash
git reset --soft HEAD~1     # défaire le dernier commit, garder les modifs staged
git reset --mixed HEAD~1    # défaire le dernier commit, modifs en working tree
git reset --hard HEAD~1     # défaire le dernier commit ET les modifs (DANGEREUX)
```

`--hard` ne devrait pas être utilisé sur des modifications non sauvegardées sans `git stash` préalable, et jamais sur une branche partagée.

[Retour en haut de page](#table-des-matières)

---

[← Versions et signatures](05-versions-et-signatures.md) · [↑ Sommaire](../README.md#table-des-matières) · [Outils du quotidien et récupération →](07-outils-du-quotidien-et-recuperation.md)
