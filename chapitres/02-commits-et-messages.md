[← Fondations et concepts](01-fondations-et-concepts.md) · [↑ Sommaire](../README.md#table-des-matières) · [Branches et stratégies de workflow →](03-branches-et-strategies-de-workflow.md)

# 2. Commits et messages

## Messages de commit

> **Que veut dire « commit » ?** Un *commit* (verbe anglais « valider, enregistrer ») est un enregistrement daté d'un état du projet : la photo dont parle l'introduction. Chaque commit garde un message, le nom de son auteur et une empreinte SHA. Faire un commit, c'est dire à Git « garde cet état, j'y reviendrai peut-être ».

Un message de commit s'adresse au futur lecteur (vous, dans six mois) qui cherche à comprendre **pourquoi** une modification a été faite. La spécification [Conventional Commits](https://www.conventionalcommits.org/fr/) impose un format que les programmes savent lire, utile pour générer automatiquement des changelogs et déclencher des publications de version.

> **Que veut dire « changelog » ?** Un *changelog* (« journal des changements ») est la liste, version par version, de ce qui a été ajouté, corrigé ou retiré dans un logiciel. C'est l'équivalent du carnet d'entretien d'une voiture : on y lit ce qui a changé et quand. Quand les messages de commit suivent une norme, un outil peut fabriquer ce journal tout seul.

### Format

```text
<type>(<portée optionnelle>): <description impérative à la 1re ligne, 50 caractères max>

<corps optionnel : pourquoi, alternatives écartées, lien vers l'issue>

<pied optionnel : BREAKING CHANGE, Refs #123, Reviewed-by, etc.>
```

### Les onze types standards

| Type | Quand l'utiliser |
|------|------------------|
| `feat` | Ajout d'une fonctionnalité visible par l'utilisateur. |
| `fix` | Correction de bug. |
| `docs` | Documentation uniquement (README, commentaires de code, ADR). |
| `style` | Mise en forme, espaces, points-virgules, **aucun** changement de comportement. |
| `refactor` | Réorganisation interne sans ajout de fonctionnalité ni correction. |
| `perf` | Amélioration de performance mesurable. |
| `test` | Ajout ou refonte de tests. |
| `build` | Changements affectant la compilation, les dépendances, le packaging. |
| `ci` | Configuration d'intégration continue (`.github/workflows/`, `.gitlab-ci.yml`). |
| `chore` | Tâche de maintenance qui ne rentre dans aucune autre case (mise à jour `.gitignore`, renommage de scripts internes, etc.). |
| `revert` | Annulation d'un commit antérieur (le corps cite le SHA reverté). |

> **Que veut dire « CI » et « intégration continue » ?** CI veut dire *Continuous Integration*, « intégration continue ». C'est un robot, hébergé chez votre plateforme (GitHub, GitLab), qui récupère chaque nouveau code et lance automatiquement les vérifications : compilation, tests, contrôle de style. Au lieu d'attendre la veille de la livraison pour découvrir que tout casse, on vérifie en continu, à chaque modification. C'est comme un contrôle qualité en bout de chaîne de montage qui inspecte chaque pièce au fur et à mesure.

### La portée (`scope`)

La portée, entre parenthèses, est un nom court désignant la zone du code touchée. Elle est facultative mais fortement recommandée dès que le projet dépasse une dizaine de modules. Exemples : `feat(auth):`, `fix(panier):`, `docs(api):`, `refactor(parser):`. Elle facilite la recherche (`git log --grep "(auth)"`) et la génération de changelogs par section.

### Ruptures de compatibilité

> **Que veut dire « rupture de compatibilité » ?** Une rupture de compatibilité (en anglais *breaking change*) est un changement qui oblige les utilisateurs de votre code à modifier le leur pour continuer à fonctionner. Exemple : renommer un bouton dans un menu ne casse rien, mais changer le nom d'une fonction que tout le monde appelle force chacun à mettre à jour ses appels. C'est comme changer le format d'une prise électrique : tous les appareils branchés dessus doivent s'adapter.

> **Que veut dire « JWT » ?** JWT (prononcé « jot ») veut dire *JSON Web Token*, un « jeton web au format JSON ». C'est un petit laissez-passer signé que le serveur remet à un utilisateur connecté ; à chaque requête, l'utilisateur le présente pour prouver son identité, comme un bracelet d'entrée de festival qu'on montre à chaque stand.

Une rupture se signale de deux manières, idéalement les deux à la fois :

1. Un `!` après le type/scope : `feat(api)!: nouvelle signature de /users`.
2. Un pied `BREAKING CHANGE:` décrivant la rupture et la procédure de migration.

```text
feat(api)!: passage de l'authentification de session à JWT

BREAKING CHANGE: les clients doivent désormais transmettre un en-tête
Authorization: Bearer <token>. Les cookies de session ne sont plus lus.
```

### À éviter

```text
update
fixed bug
WIP
.
asdf
mes modifs du vendredi
```

### À préférer

```text
feat(panier): autoriser les codes promo cumulables

Une étude utilisateur a montré que 18 % des paniers abandonnés sont liés
à l'impossibilité de cumuler une remise fidélité avec un code marketing.
Refs #482
```

```text
fix(auth): corriger la fuite mémoire sur l'expiration du token

Le timer n'était pas annulé quand l'utilisateur se déconnectait avant
expiration, ce qui retenait l'objet User en mémoire.
```

### Bonnes pratiques

| Règle | Pourquoi |
|-------|----------|
| Verbe à l'impératif (`ajoute`, `corrige`) | Cohérent avec le style des messages générés par Git (`Merge`, `Revert`). |
| 50 caractères pour le sujet, 72 pour le corps | Lisibilité dans `git log --oneline` et les outils tiers. |
| Un commit = un changement atomique | Permet `git revert` et `git bisect` ciblés. |
| Pas de point final sur le sujet | Convention partagée. |
| Référencer l'issue | Trace bidirectionnelle entre code et ticket. |
| Séparer sujet et corps par une ligne vide | Sinon, certains outils traitent tout comme une ligne unique. |
| Citer un SHA en cas de revert ou de fix de régression | `Reverts: 3e1f7c2`, `Fixes: a91d3b4`. |

### Outillage de validation

- [`commitlint`](https://commitlint.js.org/) couplé à un hook `commit-msg` rejette tout message non conforme.
- [`commitizen`](https://commitizen-tools.github.io/commitizen/) propose un assistant interactif (`cz commit`) qui guide la rédaction.
- [`semantic-release`](https://semantic-release.gitbook.io/) lit les Conventional Commits pour calculer le prochain numéro de version (selon SemVer, détaillé dans la section [Tags et versionnage](#tags-et-versionnage)) et publier automatiquement le changelog.

### Limites et critiques de Conventional Commits

Conventional Commits est une norme utile, mais ce n'est pas une recette miracle. Plusieurs critiques récurrentes méritent d'être connues avant de l'imposer à toute une équipe.

#### Le type ne tombe pas toujours juste

Que faire d'un commit qui :

- Améliore la lisibilité d'une fonction *et* corrige un edge-case mineur ? `refactor` ou `fix` ?
- Ajoute un test *et* corrige le bug que ce test révèle ? `test` ou `fix` ?
- Met à jour une dépendance pour fermer une CVE ? `chore`, `fix`, `build`, `security` ? (Une *CVE*, pour *Common Vulnerabilities and Exposures*, « failles et expositions répertoriées », est l'identifiant officiel d'une faille de sécurité connue publiquement, par exemple `CVE-2024-12345`.)
- Renomme un dossier interne sans changer le comportement ? `refactor`, `chore`, `style` ?

La spécification ne tranche pas ; chaque équipe arbitre, et les arbitrages divergent. Sans guide d'équipe, le type devient une loterie.

#### `chore:` se transforme en poubelle

`chore` est censé désigner les tâches d'outillage. En pratique, il devient le fourre-tout par défaut quand on hésite. Sur certains dépôts, 40 % des commits sont en `chore:`, ce qui ruine la valeur informative du préfixe et brouille les changelogs générés.

Antidotes :

- Documenter clairement le périmètre de chaque type dans un `CONTRIBUTING.md`.
- Ajouter des types personnalisés si pertinent : `security:`, `deps:`, `infra:`, `i18n:`.
- Réviser la liste de types tous les six mois en rétrospective.

#### Les scopes deviennent un mini-référentiel à maintenir

`feat(panier):`, `feat(checkout):`, `feat(billing):` : sur un projet de cinquante modules, la liste des scopes devient elle-même un objet à gérer. Quand un module est renommé, les anciens commits gardent l'ancien scope, et les recherches `git log --grep "(panier)"` retournent une vue partielle. Sur un monorepo, on peut être tenté d'utiliser le chemin (`feat(libs/ui):`), ce qui devient verbeux.

#### L'over-formalisation peut ralentir les revues

Sur des projets très formels, un commit rejeté par `commitlint` parce qu'il dépasse 72 caractères ou que le scope n'est pas dans la liste autorisée déclenche une friction : le développeur réécrit le message, push à nouveau, le revieweur revoit. Pour des contributions rares (open source, contributeurs occasionnels), c'est dissuasif.

#### `BREAKING CHANGE` sous-utilisé

Beaucoup d'équipes oublient le `!` ou le footer `BREAKING CHANGE:`, ce qui fait que `semantic-release` produit une version mineure pour un changement majeur. Le rituel de revue doit explicitement vérifier ce point sur les PR à risque.

#### Quand l'utiliser, quand l'éviter

| Contexte | Recommandation |
|----------|----------------|
| Bibliothèque publique avec releases automatiques (npm, PyPI) | **Oui**, indispensable. SemVer + changelog en dépend. |
| Application interne avec changelog manuel | **Oui**, mais sans `commitlint` strict : un guide d'équipe suffit. |
| Hackathon, prototype, expérimentation courte | **Non**, friction inutile. |
| Dépôt avec contributeurs externes occasionnels | **Oui**, mais avec assistant (commitizen) et indulgence côté mainteneurs. |
| Mono-développeur sur projet personnel | À volonté. Bénéfice marginal sauf publication automatisée. |

#### Alternatives partielles

- **Gitmoji** : remplace le type par un emoji. Visuellement parlant mais peu adapté à un environnement professionnel formel.
- **Angular convention** (variante historique de Conventional Commits) : très proche, plus restrictive.
- **Pas de norme** : laissez les développeurs écrire des messages clairs en prose. Marche très bien pour des équipes mûres ; échoue dès que les nouveaux arrivent.

[Retour en haut de page](#table-des-matières)

---

[← Fondations et concepts](01-fondations-et-concepts.md) · [↑ Sommaire](../README.md#table-des-matières) · [Branches et stratégies de workflow →](03-branches-et-strategies-de-workflow.md)
