---
name: save-article
description: >
  Sauvegarde un article web en markdown dans le vault Obsidian (dossier Veille/).
  Triggers: /save-article <url>, "sauvegarde cet article", "enregistre cet article dans Veille"
---

# Skill : Save Article

Sauvegarde un article web en fichier markdown formaté pour Obsidian dans le dossier `Veille/` du vault.

## Chemins

- **Vault Obsidian** : `/Users/dinum-327811/Library/CloudStorage/GoogleDrive-benoitvinceneux@gmail.com/Mon Drive/Work/DINUM - EIG/ETALAB/ETALAB-Obsidian`
- **Dossier cible** : `Veille/` (dans le vault ci-dessus)
- **Dossier téléchargements** : `/Users/dinum-327811/Downloads`

## Format de sortie

Le fichier markdown doit suivre ce template exact :

```yaml
---
titre: "Titre de l'article"
source: "https://url-complete.com/article"
date_publication: "YYYY-MM-DD"
résumé: "Résumé en 2 phrases généré par l'IA."
---
```

Suivi du contenu markdown de l'article (titre h1, paragraphes, listes, citations, blocs de code).

## Workflow

### Étape 1 — Récupérer l'URL

L'URL est passée en argument : `/save-article <url>`
Si aucune URL n'est fournie, demander à l'utilisateur.

### Étape 2 — Vérifier les doublons

Avant de récupérer le contenu, vérifier si l'article n'a pas déjà été sauvegardé :

1. Utiliser `Grep` pour chercher l'URL exacte dans les frontmatter `source:` de tous les fichiers du dossier `Veille/` :
   - Pattern : l'URL fournie par l'utilisateur
   - Glob : `*.md`
   - Path : le dossier `Veille/` du vault
2. **Si un fichier match** → informer l'utilisateur avec le chemin du fichier existant et lui demander s'il veut :
   - Écraser le fichier existant
   - Créer un nouveau fichier (suffixe numéroté)
   - Annuler
3. **Si aucun match** → continuer sans interruption

### Étape 3 — Extraire le contenu via WebFetch

Utiliser `WebFetch` avec l'URL et le prompt d'extraction suivant :

> Extrais le contenu de cet article de manière structurée. Retourne :
> 1. **Titre** : le titre principal de l'article
> 2. **Date de publication** : au format YYYY-MM-DD si disponible, sinon vide
> 3. **Contenu** : le contenu complet de l'article formaté en markdown propre (titres h2/h3, paragraphes, listes, citations avec >, blocs de code avec ```, images avec ![alt](src))
>
> Format de réponse :
> TITRE: ...
> DATE: ...
> CONTENU:
> ...

**Fallback Chrome** — Si WebFetch échoue (erreur réseau, page bloquée, contenu vide ou insuffisant < 100 caractères) :

1. Appeler `tabs_context_mcp` pour obtenir le contexte navigateur
2. Créer un nouvel onglet avec `tabs_create_mcp`
3. Naviguer vers l'URL avec `navigate`
4. Attendre 3 secondes avec `computer` action `wait` pour laisser la page charger
5. Utiliser `javascript_tool` pour extraire titre, date et contenu (querySelectorAll sur h1-h4, p, ul, ol, blockquote, pre, img)
6. Si l'extraction JS retourne peu de contenu (< 100 caractères), utiliser `get_page_text` comme fallback supplémentaire

### Étape 4 — Générer le résumé

À partir du contenu extrait, rédiger un résumé en **2 phrases** en français qui synthétise les points clés de l'article. Le résumé doit être informatif et concis.

### Étape 5 — Construire le fichier markdown

Assembler le fichier avec :
- Le frontmatter YAML (titre, source, date_publication, résumé)
- Le contenu markdown extrait

Le nom du fichier sera le titre de l'article, nettoyé des caractères spéciaux : remplacer `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|` par `-`, et tronquer à 100 caractères max. Extension `.md`.

### Étape 6 — Écrire le fichier

Utiliser le tool `Write` pour écrire directement le fichier dans le dossier Veille/ du vault :

```
{vault}/Veille/{nom-fichier-nettoyé}.md
```

### Étape 7 — Confirmer

Afficher à l'utilisateur :
- Le titre de l'article
- Le résumé généré
- Le chemin du fichier créé

## Gestion d'erreurs

- **WebFetch échoue** : tenter automatiquement le fallback Chrome (étape 3). Si Chrome n'est pas connecté, informer l'utilisateur d'ouvrir Chrome avec l'extension Claude et réessayer.
- **Page JS-heavy / SPA (fallback Chrome)** : si l'extraction JS retourne un contenu vide ou très court (< 100 caractères), utiliser `get_page_text` comme fallback pour récupérer le texte brut, puis le formater en markdown.
- **Date introuvable** : laisser `date_publication` vide (`""`).
- **Titre introuvable** : utiliser le titre de la page comme fallback.
- **Écriture échouée** : informer l'utilisateur et proposer d'écrire dans Downloads/ à la place.
- **Doublon détecté** : ne jamais écraser silencieusement — toujours demander confirmation à l'utilisateur.
