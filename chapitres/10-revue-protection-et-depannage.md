[← Configuration du dépôt et sécurité des fichiers](09-configuration-du-depot-et-securite-des-fichiers.md) · [↑ Sommaire](../README.md#table-des-matières)

# 10. Revue, protection et dépannage

## Pull Requests : la revue comme garde-fou

Une Pull Request (PR), appelée Merge Request sur GitLab, n'est pas qu'un bouton « fusionner ». C'est l'unité de revue, la dernière barrière avant `main`.

> **Que veut dire « revue de code » ?** La *revue de code* (en anglais *code review*) est la relecture du code par un ou plusieurs collègues avant qu'il rejoigne la branche principale. Ils vérifient la justesse, la clarté, la sécurité et proposent des améliorations. C'est la relecture d'un article par un comité avant publication : un deuxième regard attrape ce que l'auteur ne voit plus.

### Description d'une PR utile

```markdown
## Pourquoi
Décrire le problème ou la valeur métier (lien vers ticket).

## Quoi
Lister les changements clés en une dizaine de lignes maximum.

## Comment tester
Étapes reproductibles : commande, URL, jeu de données.

## Risques / impacts
- Migration de schéma : oui / non
- Feature flag : nom du flag, état par défaut
- Rétrocompatibilité : oui / non
```

### Bonnes pratiques côté auteur

- Ouvrir la PR tôt, en *draft* (« brouillon » : une PR marquée comme non finie, qu'on ne peut pas encore fusionner), pour valider la direction avant d'avoir tout codé.
- Garder la diff sous ~400 lignes : au-delà, la qualité de revue chute drastiquement.
- Répondre aux commentaires plutôt que les fermer silencieusement.
- Rebaser sur `main` avant le merge final pour une diff propre.

### Bonnes pratiques côté revieweur

- Distinguer **bloquants** (« ne fusionne pas tant que… ») et **suggestions** (« nit: ... »).
- Approuver explicitement quand c'est bon ; ne pas laisser une PR en suspens parce qu'on l'a oubliée.
- Critiquer le code, pas la personne. Préférer « cette boucle est en O(n²) » à « vous avez écrit une boucle en O(n²) ». (La notation *O(n²)*, « grand O de n au carré », décrit la vitesse d'un algorithme : si le nombre d'éléments `n` double, le temps de calcul est multiplié par quatre, ce qui devient vite lent.)

### Protection de branche recommandée sur `main`

- Require a pull request before merging.
- Require approvals : au moins 1 (2 sur projets sensibles).
- Require status checks to pass : CI verte, tests, lint.
- Require signed commits (si l'équipe les utilise).
- Require linear history (option qui force squash ou rebase, interdit les merges classiques).
- Do not allow bypassing the above settings.
- Restrict who can push to matching branches.
- Disallow force pushes.

[Retour en haut de page](#table-des-matières)

## Protection de branche côté GitHub

GitHub propose deux générations d'outils pour verrouiller `main` : les *Branch protection rules* (historiques) et les *Rulesets* (depuis 2023, plus puissants). Cette section détaille les options critiques.

### Required reviewers

Sous **Settings → Branches → Add rule** (ou **Rulesets → New ruleset**) :

| Option | Effet | Recommandation |
|--------|-------|----------------|
| **Require a pull request before merging** | Interdit les push directs sur la branche. | **Activer toujours** sur `main`. |
| **Require approvals (1 minimum)** | N revues approuvées avant merge. | 1 minimum, 2 sur projets sensibles ou réglementés. |
| **Dismiss stale pull request approvals when new commits are pushed** | Si l'auteur pousse de nouveaux commits, les approbations précédentes sont annulées. | **Activer**, sinon une PR approuvée tôt peut être modifiée silencieusement avant merge. |
| **Require review from Code Owners** | Force la revue par les responsables désignés dans `.github/CODEOWNERS`. | Activer dès qu'un fichier `CODEOWNERS` existe. |
| **Restrict who can dismiss pull request reviews** | Limite la levée d'une revue bloquante. | Activer pour éviter qu'un auteur ne lève sa propre revue critique. |
| **Allow specified actors to bypass required pull requests** | Liste blanche d'utilisateurs ou bots qui contournent la règle. | À utiliser avec parcimonie : Dependabot, Renovate, release-bot. |

#### Exemple `CODEOWNERS`

```text
# .github/CODEOWNERS

# Par défaut : l'équipe core revoit tout
*                    @org/core-team

# La sécurité revoit le périmètre auth
/src/auth/           @org/security
/src/crypto/         @org/security

# Le frontend a son équipe
/apps/web/           @org/frontend
/libs/ui/            @org/frontend

# La doc peut être merge sans revue technique
/docs/               @org/tech-writers

# Les workflows CI sont sensibles : double revue
/.github/workflows/  @org/devops @org/security
```

GitHub bloque le merge tant que les owners désignés n'ont pas approuvé.

### Required status checks

| Option | Effet |
|--------|-------|
| **Require status checks to pass before merging** | Bloque le merge tant que la CI n'est pas verte. |
| **Require branches to be up to date before merging** | Force un rebase / merge de `main` dans la PR avant le merge final. Plus strict mais évite les régressions de type « la PR seule passait, mais en combinaison avec un autre merge récent ça casse ». |
| Liste des checks requis | Cocher explicitement : `build`, `test`, `lint`, `security-scan`, etc. |

Attention : un check requis qui n'est jamais lancé (parce que le workflow ne se déclenche pas sur la PR) bloque indéfiniment le merge. Vérifier que les triggers de workflow correspondent aux required checks.

### Signed commits, linear history, conversation resolution

| Option | Effet | Quand activer |
|--------|-------|---------------|
| **Require signed commits** | Refuse les commits non signés (GPG ou SSH). | Projets sensibles (sécurité, supply chain). Toute l'équipe doit savoir signer. |
| **Require linear history** | Interdit les merge commits (force squash ou rebase). | Si l'équipe veut un `main` linéaire absolu. |
| **Require conversation resolution before merging** | Bloque tant qu'il reste des commentaires non résolus. | Bonne pratique : oblige à répondre à chaque remarque. |
| **Require deployments to succeed before merging** | Couple le merge à un déploiement de staging réussi. | Avancé : équipes avec environnements de preview par PR. |

### Force push et suppression

| Option | Effet | Recommandation |
|--------|-------|----------------|
| **Allow force pushes** | Si désactivé : aucun force push possible, même par les admins. | Désactiver sur `main`, `release/*`, branches protégées. |
| **Allow deletions** | Si désactivé : la branche ne peut pas être supprimée. | Désactiver sur `main`. |
| **Lock branch** | Branche en lecture seule : aucun push, même via PR. | Pour les branches archivées (vieilles releases). |

### Le piège des « bypass admins »

Par défaut, dans les *classic branch protection rules*, les administrateurs du dépôt peuvent contourner toutes les règles. C'est une commodité dangereuse : le compte admin devient une porte dérobée permanente.

| Option | Effet |
|--------|-------|
| **Do not allow bypassing the above settings** (classic) | Applique la règle aux admins eux-mêmes. |
| **Restrict who can bypass these rules** (rulesets) | Liste blanche explicite des contournements autorisés. |

**Règle de gouvernance** : un admin légitime n'a pas besoin de contourner la règle ; il peut ouvrir une PR et la faire approuver comme tout le monde. Activer le bypass admins, c'est en réalité un signal « personne ne fait vraiment respecter la règle ».

Audit : GitHub journalise toute tentative de bypass dans l'audit log (visible côté Organization → Settings → Audit log). Sur un projet sensible, mettre en place une alerte sur ces événements.

### Rulesets vs Branch protection rules (classic)

GitHub recommande désormais les **Rulesets** (2023+) :

| Avantage des Rulesets | Détail |
|----------------------|--------|
| Couvrent plusieurs branches via patterns | `main`, `release/**`, `hotfix/**` en un seul ruleset. |
| Mode « Evaluate » | Tester une règle sans l'appliquer ; observer les violations potentielles avant de durcir. |
| Liste de bypass plus fine | Granularité par utilisateur, équipe ou app GitHub. |
| Versionnés en JSON | Exportables, importables, intégrables dans une infra-as-code. |
| Couvrent aussi les tags | Un ruleset peut protéger les tags `v*` contre la suppression. |

Migration : les anciennes branch protection rules continuent de fonctionner en parallèle ; en cas de conflit, la règle la plus stricte gagne. Migrer progressivement, avec une phase d'évaluation en mode « audit only ».

### Exemple de ruleset JSON

```json
{
  "name": "Protection main",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": {
      "include": ["refs/heads/main", "refs/heads/release/**"],
      "exclude": []
    }
  },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 2,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true,
        "required_review_thread_resolution": true
      }
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          { "context": "build" },
          { "context": "test" },
          { "context": "lint" }
        ]
      }
    },
    { "type": "required_signatures" }
  ],
  "bypass_actors": []
}
```

`bypass_actors: []` signifie : **personne** n'échappe à la règle, pas même le owner de l'organisation. C'est l'option la plus sûre.

[Retour en haut de page](#table-des-matières)

## Pièges classiques et comment s'en sortir

### « Je suis en *detached HEAD* »

> **Que veut dire « detached HEAD » ?** *Detached HEAD* veut dire « tête détachée ». Normalement `HEAD` (le repère qui indique où vous travaillez) pointe sur une branche. En *detached HEAD*, il pointe directement sur un commit précis, sans branche. Tout nouveau commit créé dans cet état ne s'accroche à aucune étiquette et risque d'être ramassé puis supprimé : c'est comme écrire sur une feuille volante hors de tout classeur, facile à égarer. La solution est de créer aussitôt une branche pour rattacher ce travail.

```bash
# Vous avez fait git checkout <SHA> et travaillé un peu. Pour sauver :
git switch -c sauvetage-de-mon-travail
# Puis revenir à l'état normal :
git switch main
```

### « J'ai commité sur main par erreur »

```bash
# Sauver les commits dans une branche dédiée
git switch -c feat/mon-travail

# Revenir main à sa position d'origine (avant les commits accidentels)
git switch main
git reset --hard origin/main
```

Si les commits ont déjà été poussés sur `main`, ne **pas** force-push ; faire un `git revert` à la place et ouvrir une PR.

### « J'ai fait un `reset --hard` et perdu mon travail »

```bash
git reflog
# Identifier le HEAD@{n} d'avant le reset
git reset --hard HEAD@{n}
```

### « J'ai force-push sur une branche partagée »

Prévenir immédiatement l'équipe. Chaque collègue qui avait la branche localement devra :

```bash
git fetch origin
git reset --hard origin/<branche>   # s'ils n'avaient pas de modifs locales non poussées
```

Pour les modifs locales, sauvegarder d'abord (`git stash` ou nouvelle branche), puis réappliquer après `reset`.

### « `git pull` a créé un commit de fusion bizarre »

Cela arrive quand `pull.rebase` est à `false` et que la branche distante a divergé. Solution : configurer `pull.rebase = true` (cf. [Configuration initiale](#configuration-initiale)) et, en cas d'urgence, défaire avec `git reset --hard ORIG_HEAD`.

### « J'ai poussé un secret par erreur »

Voir [Fichiers sensibles](#fichiers-sensibles). Action immédiate : faire tourner le secret. La réécriture d'historique vient ensuite.

### « Ma PR a 800 commits depuis que j'ai mergé main dedans plusieurs fois »

Préférer un rebase une fois en fin de PR plutôt que des merges successifs en cours de route. Pour rattraper :

```bash
git fetch origin
git rebase -i origin/main
# squasher les commits parasites, garder les commits métier
git push --force-with-lease
```

[Retour en haut de page](#table-des-matières)

## Antisèche des commandes

### Inspecter

```bash
git status                          # état du working tree et de l'index
git log --oneline --graph --all     # historique compact, toutes branches
git log --stat                      # historique avec liste de fichiers modifiés
git log -p <fichier>                # historique des modifications d'un fichier
git blame <fichier>                 # qui a écrit chaque ligne et quand
git show <SHA>                      # contenu complet d'un commit
git diff                            # working tree vs index
git diff --staged                   # index vs HEAD
git diff main...feat/x              # ce qu'apporte feat/x par rapport à main
```

### Manipuler

```bash
git switch <branche>                # changer de branche (Git 2.23+)
git switch -c <branche>             # créer et basculer
git restore <fichier>               # défaire les modifs non staged
git restore --staged <fichier>      # unstage
git add -p                          # ajouter par hunks (revue ligne à ligne)
git commit -m "feat(x): ..."        # commit
git commit --amend                  # modifier le dernier commit
```

### Synchroniser

```bash
git fetch origin                    # récupérer sans modifier le working tree
git pull --rebase                   # fetch + rebase (recommandé)
git push -u origin <branche>        # premier push, lie la branche locale au distant
git push --force-with-lease         # force-push sécurisé
```

### Sortir d'un mauvais pas

```bash
git reflog                          # journal local des mouvements
git stash                           # mettre de côté
git merge --abort                   # annuler un merge en cours
git rebase --abort                  # annuler un rebase en cours
git cherry-pick --abort             # annuler un cherry-pick en cours
git fsck --lost-found               # retrouver des objets orphelins
```

### Travailler sur plusieurs choses en parallèle

```bash
git worktree add ../proj-hotfix hotfix/x   # nouveau checkout dans un autre dossier
git worktree list                          # lister les worktrees
git worktree remove ../proj-hotfix         # nettoyer
```

### Gros dépôts

```bash
git clone --filter=blob:none <url>         # partial clone (blobs à la demande)
git clone --depth=1 <url>                  # shallow clone (CI, agents)
git sparse-checkout init --cone            # restreindre l'arbre matérialisé
git sparse-checkout set apps/web libs/ui   # ne déployer qu'un sous-ensemble
git lfs track "*.psd"                      # déporter les gros binaires
git gc                                     # repacker le dépôt
git maintenance start                      # maintenance automatique en arrière-plan
```

### Réutiliser des résolutions de conflit

```bash
git config --global rerere.enabled true    # activer rerere
git rerere status                          # voir ce qui est mémorisé
git rerere clear                           # purger les résolutions enregistrées
```

### Internes : inspecter les objets

```bash
git cat-file -t <SHA>                      # type d'un objet
git cat-file -p <SHA>                      # contenu lisible
git rev-parse HEAD                         # SHA complet de HEAD
git rev-parse HEAD^{tree}                  # SHA du tree pointé par HEAD
git verify-pack -v .git/objects/pack/*.idx | sort -k 3 -n | tail -20
                                           # 20 plus gros objets (chasse aux binaires)
```

[Retour en haut de page](#table-des-matières)

---

[← Configuration du dépôt et sécurité des fichiers](09-configuration-du-depot-et-securite-des-fichiers.md) · [↑ Sommaire](../README.md#table-des-matières)
