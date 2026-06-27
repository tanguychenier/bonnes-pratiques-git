[← Internes et gros dépôts](08-internes-et-gros-depots.md) · [↑ Sommaire](../README.md#table-des-matières) · [Revue, protection et dépannage →](10-revue-protection-et-depannage.md)

# 9. Configuration du dépôt et sécurité des fichiers

## Fichiers sensibles

> **Que veut dire « secret », « clé d'API » et « faire tourner un secret » ?** Un *secret* est une information confidentielle qui donne un accès : mot de passe, certificat, jeton. Une *clé d'API* (*Application Programming Interface*, « interface de programmation ») est un identifiant secret qui autorise votre programme à utiliser un service externe (paiement, envoi d'e-mails). *Faire tourner* un secret (en anglais *rotate*), c'est le révoquer et en générer un nouveau, exactement comme on change une serrure dont la clé a été copiée.

Mots de passe, clés d'API, certificats, fichiers `.env` (où l'on range les variables d'environnement, souvent sensibles) : tout secret enregistré dans un commit reste dans l'historique pour toujours, **même après suppression du fichier**. Une fois publié, considérez le secret comme compromis et faites-le tourner immédiatement, car n'importe qui ayant cloné le dépôt en garde une copie.

### Prévenir

| Pratique | Outils |
|----------|--------|
| `.gitignore` complet dès l'init du dépôt | [gitignore.io](https://www.toptal.com/developers/gitignore) |
| Gabarits sans valeurs (`.env.example`) | Aucun outil requis |
| Hook `pre-commit` qui scanne les secrets | [gitleaks](https://github.com/gitleaks/gitleaks), [trufflehog](https://github.com/trufflesecurity/trufflehog), [detect-secrets](https://github.com/Yelp/detect-secrets) |
| Stockage dans un coffre | [Vault](https://www.vaultproject.io/), AWS Secrets Manager, 1Password, Doppler |
| Secret scanning côté plateforme | [GitHub secret scanning](https://docs.github.com/fr/code-security/secret-scanning) (gratuit pour les dépôts publics, partenaires intégrés pour AWS, Stripe, GitHub Tokens, etc.). |

### Réagir à une fuite

```bash
# 1. Faire tourner immédiatement le secret compromis (révocation, nouvelle clé).
# 2. Réécrire l'historique pour retirer le fichier.
git filter-repo --path config/secrets.yml --invert-paths

# Variante : retirer un motif (ex. toute ligne contenant un token AWS).
git filter-repo --replace-text <(echo 'AKIA[0-9A-Z]{16}==>REDACTED')

# 3. Forcer le push (et prévenir l'équipe).
git push --force-with-lease --all
git push --force-with-lease --tags
```

`git filter-repo` ([documentation](https://github.com/newren/git-filter-repo)) remplace `git filter-branch`, désormais déprécié. Une alternative simple à utiliser est [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/), particulièrement efficace sur les gros dépôts :

```bash
# Supprimer toutes les occurrences d'un fichier dans l'historique
bfg --delete-files secrets.yml

# Remplacer des chaînes (mots de passe, tokens) par ***REMOVED***
bfg --replace-text passwords.txt
```

### Après réécriture

- Sur GitHub, demander la purge du cache des forks (sinon le secret reste accessible via les forks).
- Faire tourner aussi tout secret dérivé (sessions, certificats émis avec la clé compromise).
- Ouvrir un *post-mortem* (« analyse après coup », sans chercher de coupable) : pourquoi le contrôle automatique n'a-t-il pas attrapé la fuite, et comment l'éviter la prochaine fois ?

[Retour en haut de page](#table-des-matières)

## Le fichier .gitignore

> **Que veut dire « `.gitignore` » et « suivre un fichier » ?** *Suivre* (en anglais *track*) un fichier, c'est demander à Git d'en surveiller les changements. Le fichier `.gitignore` (« Git, ignore ») contient une liste de motifs de fichiers à ne PAS suivre, pour que Git les laisse tranquilles. C'est la liste « ne pas ranger » posée sur un bureau : certains papiers de travail temporaires ne doivent jamais finir aux archives. *Un motif* est une formule courte qui désigne plusieurs fichiers à la fois, par exemple `*.log` pour « tous les fichiers qui finissent par `.log` ».

`.gitignore` sert à écarter ce qui n'a pas à être versionné : *artefacts* de build (fichiers fabriqués automatiquement à partir du code), dépendances installées, réglages locaux de l'IDE, gros binaires.

> **Que veut dire « IDE » ?** IDE veut dire *Integrated Development Environment*, « environnement de développement intégré ». C'est le logiciel dans lequel on écrit le code (VS Code, IntelliJ), qui réunit éditeur de texte, coloration, autocomplétion et lancement des tests au même endroit, comme un atelier où tous les outils sont à portée de main.

### Exemple PHP / Node minimal

```gitignore
# Dépendances
/vendor/
/node_modules/

# Variables d'environnement
.env
.env.local

# Builds
/dist/
/build/
*.log

# IDE / OS
.idea/
.vscode/
.DS_Store
Thumbs.db
```

### Syntaxe rapide

| Motif | Effet |
|-------|-------|
| `build/` | Tout dossier nommé `build`, à n'importe quelle profondeur. |
| `/build/` | Le dossier `build` à la **racine** du dépôt seulement. |
| `*.log` | Tout fichier `.log`, à n'importe quelle profondeur. |
| `!important.log` | Exception : ne **pas** ignorer `important.log`. |
| `**/tmp` | Tout sous-dossier `tmp`, à n'importe quel niveau. |
| `doc/**/*.pdf` | Tous les `.pdf` sous `doc/`, à toute profondeur. |

### Pièges courants

| Symptôme | Cause | Solution |
|----------|-------|----------|
| Le fichier reste suivi malgré le `.gitignore` | Il a été commité avant l'ajout de la règle. | `git rm --cached <fichier>` puis recommit. |
| Une règle ne s'applique pas | Mauvais slash : `/build` (racine) vs `build/` (partout). | Tester avec `git check-ignore -v <fichier>`. |
| Trop de bruit dans le diff | Pas de gabarit central. | Démarrer depuis un modèle [gitignore.io](https://www.toptal.com/developers/gitignore). |
| Un fichier privé est ignoré mais reste sensible | `.gitignore` n'enlève **rien** de l'historique. | Voir [Fichiers sensibles](#fichiers-sensibles). |

### `.gitignore` global

Pour les fichiers spécifiques à votre poste (configuration d'IDE personnelle, fichiers macOS), un `.gitignore` global évite de polluer chaque dépôt :

```bash
git config --global core.excludesFile ~/.gitignore_global
```

[Retour en haut de page](#table-des-matières)

## Le fichier .gitattributes et la normalisation CRLF

> **Que veut dire « `.gitattributes` » ?** Le fichier `.gitattributes` attache des règles de traitement à certains fichiers : comment gérer leurs fins de ligne, comment les comparer, lesquels considérer comme binaires, lesquels confier à Git LFS. Là où `.gitignore` dit « ignore ces fichiers », `.gitattributes` dit « traite ces fichiers de telle manière ». C'est l'étiquette de lavage cousue sur un vêtement : elle ne le cache pas, elle indique comment s'en occuper.

Là où `.gitignore` cache des fichiers, `.gitattributes` change la façon dont Git les traite.

### Cas d'usage typiques

```gitattributes
# Fins de ligne : LF dans le dépôt, conversion à la volée selon la plateforme
* text=auto eol=lf

# Forcer LF pour les scripts shell et la config (sinon CRLF sous Windows casse tout)
*.sh   text eol=lf
*.yml  text eol=lf
*.yaml text eol=lf

# Forcer CRLF pour les fichiers Windows-only
*.bat  text eol=crlf
*.cmd  text eol=crlf

# Marquer comme binaire (pas de diff, pas de fusion automatique)
*.png  binary
*.pdf  binary
*.xlsx binary

# Diff personnalisé pour des formats courants
*.json   diff=json
*.md     diff=markdown

# Exclure des fichiers de l'archive `git archive` (utile pour les releases)
.github/        export-ignore
tests/          export-ignore
.gitattributes  export-ignore
.gitignore      export-ignore

# Git LFS pour les gros binaires (après git lfs install)
*.psd  filter=lfs diff=lfs merge=lfs -text
*.zip  filter=lfs diff=lfs merge=lfs -text
```

### Pourquoi c'est important pour une équipe mixte Windows / macOS / Linux

Sans `.gitattributes`, deux développeurs sur deux OS différents finissent par produire des diffs vides remplis uniquement de changements de fins de ligne. La règle `* text=auto eol=lf` règle ce problème une fois pour toutes en imposant LF dans le dépôt et en laissant Git convertir au checkout selon l'OS.

### Le problème CRLF en détail

> **Que veut dire « CRLF » et « LF » ?** À la fin de chaque ligne d'un fichier texte se cache un caractère invisible qui dit « passe à la ligne ». Windows en met deux, *CR* (*Carriage Return*, « retour chariot ») suivi de *LF* (*Line Feed*, « saut de ligne »), noté `\r\n`. macOS et Linux n'en mettent qu'un, *LF* seul, noté `\n`. Comme la machine à écrire d'autrefois qui faisait deux gestes (ramener le chariot, puis dérouler le papier) là où un traitement de texte moderne n'en fait qu'un. Quand deux systèmes mélangent ces conventions, Git voit des modifications partout alors que le texte visible n'a pas changé.

Historique : Windows utilise `CRLF` (`\r\n`) en fin de ligne, héritage des télétypes IBM. macOS / Linux utilisent `LF` (`\n`). Ouvrir un fichier `LF` dans le Bloc-notes Windows historique affiche tout sur une ligne ; ouvrir un fichier `CRLF` dans certains outils Linux fait apparaître `^M` à la fin de chaque ligne.

Sans normalisation, voici ce qui se passe :

1. Alice (Linux) crée `app.js` en `LF` et le commite.
2. Bob (Windows) clone, son éditeur convertit le fichier en `CRLF` à l'enregistrement.
3. Bob commite : *toutes* les lignes apparaissent comme modifiées dans le diff, alors qu'il n'a touché qu'une seule ligne.
4. Alice tire les changements de Bob, son éditeur reconvertit en `LF`, et chaque va-et-vient produit un diff catastrophique.
5. Le `git blame` est ruiné : chaque ligne est attribuée au dernier qui a sauvegardé.

### Trois leviers complémentaires

| Levier | Effet | Recommandation |
|--------|-------|----------------|
| `core.autocrlf` (config Git locale) | `true` sur Windows : convertit en CRLF au checkout, en LF au commit. `input` sur Linux/macOS : pas de conversion au checkout, LF au commit. `false` : pas de conversion. | Acceptable comme repli, mais dépend de la config de chaque poste. |
| `.gitattributes` avec `* text=auto eol=lf` | Politique versionnée dans le dépôt, s'applique à tous les contributeurs. | **À privilégier** : la règle voyage avec le dépôt. |
| `.editorconfig` à la racine | Indique aux IDE la convention (indentation, fin de ligne, charset). | Complémentaire : règle aussi le problème dès l'édition. |

### Recette qui marche

```gitattributes
# .gitattributes : politique standard
* text=auto eol=lf

# Forcer LF même sur Windows (scripts shell qui ne tolèrent pas CRLF)
*.sh   text eol=lf
*.bash text eol=lf

# Forcer CRLF pour les fichiers Windows-only
*.bat text eol=crlf
*.cmd text eol=crlf
*.ps1 text eol=crlf

# Marquer comme binaire (pas de conversion, pas de diff textuel)
*.png  binary
*.jpg  binary
*.pdf  binary
*.xlsx binary
*.docx binary
```

```ini
# .editorconfig : recommandation IDE
root = true

[*]
end_of_line = lf
charset = utf-8
insert_final_newline = true
trim_trailing_whitespace = true

[*.{bat,cmd,ps1}]
end_of_line = crlf
```

### Migrer un dépôt qui a déjà du CRLF dans son historique

Si l'historique est déjà pollué, une réécriture en masse remet les pendules à l'heure :

```bash
# 1. Ajouter ou mettre à jour .gitattributes (cf. ci-dessus).
git add .gitattributes
git commit -m "chore: normalisation des fins de ligne"

# 2. Renormaliser tout l'index.
git add --renormalize .
git commit -m "chore: renormalisation des fins de ligne sur tout le dépôt"
```

`--renormalize` re-stocke chaque fichier en respectant la nouvelle politique `.gitattributes`, sans modifier leur contenu côté texte (juste les sauts de ligne).

[Retour en haut de page](#table-des-matières)

## Hooks Git

> **Que veut dire « hook » ?** Un *hook* (« crochet, point d'accroche ») est un petit script que Git lance automatiquement à un moment précis : juste avant un commit, juste avant un push, après une fusion. Il peut compléter l'action (par exemple reformater le code) ou la bloquer (refuser un commit non conforme). C'est un déclencheur automatique, comme le détecteur de fumée qui se met en marche tout seul quand l'événement attendu survient. *Un script* est une suite d'instructions exécutées par l'ordinateur sans intervention.

Les hooks sont **locaux** : `git clone` ne les copie pas. Pour les partager dans une équipe, on passe par un gestionnaire dédié.

### Hooks usuels

| Hook | Déclenchement | Usage typique |
|------|---------------|---------------|
| `pre-commit` | Avant la création du commit | Linter, formateur, tests rapides, scan de secrets. |
| `prepare-commit-msg` | Avant l'ouverture de l'éditeur | Pré-remplir le message (ex. ajouter le numéro de ticket extrait du nom de branche). |
| `commit-msg` | Après saisie du message | Vérifier la convention (Conventional Commits via `commitlint`). |
| `pre-push` | Avant `git push` | Suite de tests complète, refus de pousser sur `main`. |
| `post-merge` | Après un `git merge` | Réinstaller les dépendances si `composer.lock` a changé. |
| `post-checkout` | Après `git switch` ou `git checkout` | Recharger les outils de version (nvm, asdf). |

### Outils de partage en équipe

- [pre-commit](https://pre-commit.com/) (Python, multi-langage, écosystème très riche)
- [Husky](https://typicode.github.io/husky/) (JavaScript / Node)
- [Lefthook](https://github.com/evilmartians/lefthook) (binaire unique, multi-langage, configuration YAML)

### Exemple `pre-commit` : lint et scan de secrets

> **Que veut dire « lint » ?** Un *linter* est un outil qui analyse le code sans l'exécuter pour repérer les erreurs de style, les fautes courantes et les constructions douteuses. *Linter* le code, c'est le passer au peigne fin automatiquement, comme un correcteur orthographique pour la programmation. Le nom vient de l'anglais *lint*, les peluches qu'on retire d'un vêtement.

```bash
#!/usr/bin/env bash
# .git/hooks/pre-commit : rendre exécutable : chmod +x .git/hooks/pre-commit
set -euo pipefail

echo "==> Lint"
if ! npm run lint --silent; then
    echo "Lint en échec - commit annulé." >&2
    exit 1
fi

echo "==> Scan de secrets"
if command -v gitleaks >/dev/null 2>&1; then
    gitleaks protect --staged --redact --no-banner
else
    echo "gitleaks non installé, scan ignoré." >&2
fi
```

### Exemple `commit-msg` : vérifier Conventional Commits

```bash
#!/usr/bin/env bash
# .git/hooks/commit-msg
COMMIT_MSG_FILE="$1"
PATTERN='^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([a-z0-9_-]+\))?!?: .{1,72}$'

if ! grep -qE "$PATTERN" "$COMMIT_MSG_FILE"; then
    echo "Message non conforme Conventional Commits :" >&2
    cat "$COMMIT_MSG_FILE" >&2
    echo
    echo "Format attendu :" >&2
    echo "  <type>(<scope>): <description>" >&2
    echo "  Types : feat, fix, docs, style, refactor, perf, test, build, ci, chore, revert" >&2
    exit 1
fi
```

### Exemple `pre-push` : bloquer un push direct sur main

```bash
#!/usr/bin/env bash
# .git/hooks/pre-push
protected_branch='main'
current_branch=$(git symbolic-ref HEAD | sed -e 's,.*/\(.*\),\1,')

if [ "$current_branch" = "$protected_branch" ]; then
    echo "Push direct sur '$protected_branch' interdit. Passez par une Pull Request." >&2
    exit 1
fi
```

### Exemple `pre-commit` (gestionnaire `pre-commit`) : `.pre-commit-config.yaml`

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-merge-conflict
      - id: check-added-large-files
        args: ['--maxkb=500']
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.4
    hooks:
      - id: gitleaks
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.2.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
```

Installation : `pip install pre-commit && pre-commit install --hook-type pre-commit --hook-type commit-msg`.

### Limites des hooks : la CI reste le vrai garde-fou

Les hooks Git locaux ont une limite fondamentale : **ils peuvent être contournés**. Tout développeur peut, volontairement ou non, ignorer un hook qui le gêne :

```bash
# Saute pre-commit ET commit-msg
git commit --no-verify -m "fix: hotfix urgent"

# Saute pre-push
git push --no-verify

# Désactivation complète des hooks
git config core.hooksPath /dev/null
```

Conséquences pratiques :

- Un secret peut être commité malgré `gitleaks` en `pre-commit`, si l'auteur tape `--no-verify` (parfois par habitude, parfois par urgence ressentie).
- Un message non conforme passe si `commit-msg` est sauté.
- Les linters et tests rapides ne s'exécutent plus.

Le hook est donc un **assistant**, pas un **gardien**. Le vrai gardien est la **CI / le serveur** :

| Garde-fou | Forçabilité par le développeur | Couverture |
|-----------|-------------------------------|-----------|
| `pre-commit` local | Contournable (`--no-verify`). | Avant le commit, sur le poste de l'auteur. |
| Hooks côté serveur (Git natif `pre-receive`) | Non contournable, mais nécessite un serveur Git auto-hébergé pour les configurer. | Au push, refus définitif si le hook échoue. |
| **CI (GitHub Actions, GitLab CI, Jenkins)** | **Non contournable**. | Sur chaque PR / commit poussé, exécutée par la plateforme. |
| Branch protection (« required status checks ») | Bloque le merge si la CI n'est pas verte. | Au moment du merge dans `main`. |
| Secret scanning côté plateforme | Non contournable. | À chaque push, sur le serveur. |

### Stratégie recommandée

| Contrôle | Côté hook local | Côté CI |
|----------|-----------------|---------|
| Lint rapide | Oui (feedback immédiat) | Oui (vérité) |
| Format auto | Oui (corrige avant commit) | Vérification (refuser si pas formaté) |
| Tests unitaires rapides | Optionnel | **Oui, obligatoire** |
| Tests d'intégration | Non (trop lents) | **Oui** |
| Scan de secrets | Oui (interception précoce) | **Oui, obligatoire** (filet de sécurité) |
| Conventional Commits | Oui | Oui (sur le titre de PR) |
| Build de production | Non | **Oui** |
| Audit de licences, dépendances vulnérables | Non | **Oui** |

Règle d'or : tout ce qui doit être *garanti* avant un merge doit tourner en CI. Les hooks locaux sont un **gain d'ergonomie** (feedback en 2 secondes au lieu de 5 minutes), pas une garantie.

### Hooks côté serveur

Sur un serveur Git auto-hébergé (Gitea, GitLab self-managed, Bitbucket Server), on peut configurer des hooks `pre-receive` ou `update` qui s'exécutent à chaque push et qu'aucun client ne peut contourner. C'est la seule manière de garantir, par exemple, qu'aucun secret ne franchit jamais la frontière vers le dépôt.

GitHub.com ne permet pas de hooks `pre-receive` personnalisés (sauf sur GitHub Enterprise Server) ; il faut s'appuyer sur GitHub Actions et les *required status checks* à la place.

[Retour en haut de page](#table-des-matières)

---

[← Internes et gros dépôts](08-internes-et-gros-depots.md) · [↑ Sommaire](../README.md#table-des-matières) · [Revue, protection et dépannage →](10-revue-protection-et-depannage.md)
