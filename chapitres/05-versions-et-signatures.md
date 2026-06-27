[← Intégration : fusion, conflits et revue par couches](04-integration-fusion-conflits-et-revue-par-couches.md) · [↑ Sommaire](../README.md#table-des-matières) · [Réécrire et corriger l'historique →](06-reecrire-et-corriger-lhistorique.md)

# 5. Versions et signatures

## Tags et versionnage

> **Que veut dire « tag » ?** Un *tag* (« étiquette ») marque un commit précis avec un nom mémorable, typiquement un numéro de version comme `v1.4.0`. Par convention, une fois posé il ne bouge plus. C'est le marque-page permanent que l'on colle sur la page « version livrée au public », pour la retrouver instantanément des mois plus tard. Il en existe deux sortes : le tag *léger* (un simple raccourci vers une empreinte SHA) et le tag *annoté* (un vrai objet Git qui porte un auteur, une date, un message et peut être signé).

Un tag sert à marquer une version publiée pour la retrouver, la comparer ou la reconstruire plus tard.

### Tags annotés vs légers

```bash
# Tag annoté (objet Git complet : auteur, date, message, signature possible)
git tag -a v1.4.0 -m "Release 1.4.0"

# Tag léger (simple alias de SHA, pas d'auteur ni de message)
git tag v1.4.0
```

Préférez **toujours** les tags annotés pour les releases publiques : ils portent une signature et un message, et `git describe` ne fonctionne correctement qu'avec eux.

### SemVer

> **Que veut dire « SemVer » ?** SemVer veut dire *Semantic Versioning*, « versionnage sémantique ». C'est une convention pour numéroter les versions sous la forme `MAJEUR.MINEUR.CORRECTIF` (par exemple `2.4.1`), où chaque nombre a une signification précise. Au lieu d'un numéro choisi au hasard, le numéro raconte ce qui a changé : ainsi, en lisant `1.4.0 → 2.0.0`, un utilisateur sait immédiatement qu'il y a une rupture de compatibilité.

[Semantic Versioning](https://semver.org/lang/fr/) impose le format `MAJEUR.MINEUR.CORRECTIF` :

| Incrément | Signal envoyé |
|-----------|---------------|
| `MAJEUR` (1.x.x → 2.0.0) | Rupture d'API publique. |
| `MINEUR` (1.4.x → 1.5.0) | Ajout rétrocompatible. |
| `CORRECTIF` (1.4.0 → 1.4.1) | Correction de bug rétrocompatible. |

Les *pré-releases* se notent avec un suffixe : `1.5.0-rc.1`, `2.0.0-beta.3`. Les *métadonnées de build* utilisent un `+` : `1.5.0+build.42`.

```bash
git tag -a v2.0.0 -m "Release 2.0.0 - BREAKING: nouvelle API d'authentification"
git push origin v2.0.0          # un tag ne se pousse pas tout seul
git push --tags                 # ou pousser tous les tags d'un coup
```

### Supprimer un tag

```bash
git tag -d v1.4.0                # local
git push origin :refs/tags/v1.4.0  # distant
```

À éviter sauf erreur évidente : un tag publié devrait être considéré comme immuable.

[Retour en haut de page](#table-des-matières)

## Commits et tags signés

> **Que veut dire « commit signé » ?** Un *commit signé* porte une signature cryptographique qui prouve que l'auteur affiché est bien celui qui a créé le commit. Sans elle, le nom de l'auteur n'est qu'une déclaration que personne ne vérifie. La signature est comme un sceau de cire ou une signature manuscrite officielle : elle garantit l'identité et que rien n'a été modifié après coup.

> **Que veut dire « GPG », « SSH », « clé publique » et « clé privée » ?** Ces outils reposent sur la *cryptographie à clés*, qui fonctionne par paire : une *clé privée* que vous gardez secrète et une *clé publique* que vous distribuez. Ce que vous signez avec la privée peut être vérifié par tout le monde grâce à la publique, sans jamais révéler la privée, comme une serrure (publique) que seule votre clé (privée) ouvre. *GPG* (*GNU Privacy Guard*) et *SSH* (*Secure Shell*) sont deux logiciels qui gèrent ces paires de clés ; *S/MIME* est une troisième méthode, basée sur des certificats.

Sans signature, le champ `author` d'un commit est purement déclaratif : n'importe qui peut écrire `Linus Torvalds <torvalds@linux-foundation.org>`. Pour les projets sensibles (sécurité, *supply chain*, c'est-à-dire la chaîne d'approvisionnement logicielle : toutes les briques externes dont dépend un produit), GitHub et GitLab affichent un badge « Verified » uniquement sur les commits signés dont la clé publique est enregistrée sur le compte.

### Signature SSH (recommandée depuis Git 2.34)

Plus simple à mettre en place que GPG si vous avez déjà une clé SSH.

```bash
# Indiquer à Git d'utiliser SSH pour signer
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub

# Activer la signature automatique des commits et des tags
git config --global commit.gpgsign true
git config --global tag.gpgsign  true
```

Sur GitHub : **Settings → SSH and GPG keys → New SSH key → Key type : Signing Key**, et coller la clé publique.

### Signature GPG (historique, toujours valide)

```bash
# Générer une clé GPG (algorithme moderne, sans expiration courte)
gpg --full-generate-key            # choisir RSA 4096 ou ed25519

# Lister les clés et récupérer l'ID
gpg --list-secret-keys --keyid-format=long

# Configurer Git
git config --global user.signingkey <ID>
git config --global commit.gpgsign true
git config --global tag.gpgsign  true

# Exporter la clé publique pour la coller dans GitHub
gpg --armor --export <ID>
```

### Signer ponctuellement

```bash
git commit -S -m "feat(auth): ..."   # commit signé une seule fois
git tag   -s v1.4.0 -m "Release 1.4.0"  # tag signé
git verify-commit <SHA>
git verify-tag    v1.4.0
```

### Politique d'équipe

- Activer la règle « Require signed commits » sur la branche `main` pour les projets critiques.
- Documenter dans le README la procédure de génération et d'enregistrement de la clé.
- Ne jamais commiter une clé privée (voir la section [Fichiers sensibles](#fichiers-sensibles)).

[Retour en haut de page](#table-des-matières)

---

[← Intégration : fusion, conflits et revue par couches](04-integration-fusion-conflits-et-revue-par-couches.md) · [↑ Sommaire](../README.md#table-des-matières) · [Réécrire et corriger l'historique →](06-reecrire-et-corriger-lhistorique.md)
