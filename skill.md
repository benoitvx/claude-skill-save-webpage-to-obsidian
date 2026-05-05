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
- **Dossier racine veille** : `Veille/` (dans le vault ci-dessus)
- **Dossier téléchargements** : `/Users/dinum-327811/Downloads`

### Sous-dossiers

Les articles sont rangés dans des sous-dossiers de `Veille/` selon leur tag principal :

| Tag principal | Dossier cible |
|---|---|
| `mcp` | `Veille/MCP/` |
| `ai-coding` | `Veille/AI Coding/` |
| `strategie`, `design-produit`, `infrastructure`, `souverainete`, `secteur-public` | `Veille/Stratégie/` |
| `open-source` | `Veille/Open Source/` |

Le **tag principal** est le premier tag attribué à l'article. En cas de doute, le dossier `Stratégie/` est le fourre-tout par défaut.

## Format de sortie

Le fichier markdown doit suivre ce template exact :

```yaml
---
titre: "Titre de l'article"
source: "https://url-complete.com/article"
date_publication: "YYYY-MM-DD"
type: "article"
résumé: "Résumé en 2 phrases généré par l'IA."
tags:
  - tag1
  - tag2
---
```

### Champ `type`

| Valeur | Usage |
|--------|-------|
| `article` | Article texte classique (par défaut) |
| `vidéo` | Page contenant principalement une vidéo (YouTube, Vimeo, etc.) |
| `podcast` | Page contenant principalement un épisode audio |
| `mixte` | Article texte accompagné d'une vidéo ou d'un audio significatif |

### Tags disponibles

Choisir 1 à 3 tags parmi la taxonomie suivante :

| Tag | Usage |
|-----|-------|
| `mcp` | Model Context Protocol, écosystème et architecture |
| `ai-coding` | Assistants de code IA, vibe coding, productivité dev |
| `strategie` | Tendances marché, prédictions, vision industrielle |
| `design-produit` | UX/UI, interfaces IA, design process |
| `open-source` | Gouvernance OSS, contributions IA, communautés |
| `infrastructure` | Couches techniques, cloud, local-first |
| `souverainete` | Souveraineté numérique, indépendance tech |
| `secteur-public` | Administration, État plateforme, politiques publiques |

Si aucun tag existant ne convient :
1. Proposer un nouveau tag à l'utilisateur (nom en kebab-case, description courte)
2. Si l'utilisateur valide, ajouter le tag au tableau de taxonomie dans `Veille/Index.md` (section "## Taxonomie")
3. Créer une nouvelle section `## Nom du tag` en bas de l'`Index.md` avec un tableau vide prêt à recevoir des articles
4. Utiliser le nouveau tag dans l'article

Suivi du contenu markdown de l'article (titre h1, paragraphes, listes, citations, blocs de code).

## Workflow

### Étape 1 — Récupérer l'URL

L'URL est passée en argument : `/save-article <url>`
Si aucune URL n'est fournie, demander à l'utilisateur.

### Étape 2 — Vérifier les doublons

Avant de récupérer le contenu, vérifier si l'article n'a pas déjà été sauvegardé :

1. Utiliser `Grep` pour chercher l'URL exacte dans les frontmatter `source:` de tous les fichiers du dossier `Veille/` (y compris les sous-dossiers) :
   - Pattern : l'URL fournie par l'utilisateur
   - Glob : `**/*.md`
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

### Étape 3bis — Détecter et transcrire les contenus audio/vidéo

Après l'extraction du contenu, vérifier si la page contient des médias audio ou vidéo. La détection se fait en deux temps : d'abord dans le contenu extrait, puis si nécessaire via Chrome.

#### Détection

Chercher dans le contenu extrait (ou via `javascript_tool` en fallback Chrome) la présence de :
- **YouTube** : URLs contenant `youtube.com/watch`, `youtu.be/`, ou iframes `youtube.com/embed/`
- **Vimeo** : URLs contenant `vimeo.com/`
- **Podcasts** : balises `<audio>`, players embarqués (Spotify, Apple Podcasts, Anchor, Buzzsprout, Podbean, etc.), ou liens vers des fichiers `.mp3`, `.m4a`, `.ogg`
- **HTML5 video** : balises `<video>`, fichiers `.mp4`, `.webm`
- **Autres plateformes** : Dailymotion, Twitch clips, PeerTube, etc.

Si aucun média n'est détecté → passer à l'étape 4, `type` = `article`.

#### Extraction du transcript

Si un média est détecté, extraire le transcript dans cet ordre de priorité :

**1. Transcript intégré à la page**
Chercher sur la page elle-même un transcript/une transcription :
- Via `javascript_tool` : chercher des éléments avec des classes/IDs contenant `transcript`, `transcription`, `captions`, `subtitles`
- Chercher des sections titrées "Transcript", "Transcription", "Show notes", "Retranscription"
- Si trouvé, extraire le texte et l'utiliser directement

**2. YouTube — sous-titres via la page**
Si c'est une vidéo YouTube :
1. Ouvrir la page YouTube dans Chrome (`tabs_create_mcp` + `navigate`)
2. Utiliser `javascript_tool` pour cliquer sur le bouton "...Plus" / "Show transcript" ou accéder au menu sous-titres
3. Extraire le texte du panneau de transcription via `javascript_tool` :
   ```js
   // Tenter d'ouvrir le transcript YouTube
   document.querySelector('button[aria-label*="transcript" i], button[aria-label*="transcription" i], ytd-button-renderer tp-yt-paper-button')?.click();
   ```
4. Après 2 secondes, extraire le contenu :
   ```js
   document.querySelectorAll('ytd-transcript-segment-renderer .segment-text, yt-formatted-string.segment-text')
   ```
5. Si le transcript YouTube n'est pas disponible via l'UI, tenter `get_page_text` pour récupérer ce qui est visible

**3. Outil de transcription externe (fallback)**
Si aucun transcript n'est trouvé par les méthodes ci-dessus :
1. Informer l'utilisateur qu'aucun transcript automatique n'a été trouvé
2. Proposer les options :
   - Continuer sans transcript
   - Fournir manuellement une URL vers un transcript ou un fichier `.srt`/`.vtt`
   - Coller le transcript directement

#### Nettoyage du transcript

Le transcript brut doit être nettoyé avant insertion :
- Supprimer les timestamps (formats `[00:00:00]`, `00:00 -`, etc.) sauf si l'utilisateur demande de les garder
- Fusionner les lignes fragmentées en paragraphes cohérents (regrouper par pauses naturelles / changements de sujet)
- Corriger la casse basique (majuscules en début de phrase)
- Si le transcript est dans une langue étrangère, le garder dans la langue originale (ne pas traduire)

#### Détermination du `type`

- Si la page est principalement un player vidéo/audio avec peu de texte → `vidéo` ou `podcast`
- Si la page est un article texte accompagné d'un média → `mixte`
- Si aucun média → `article`

### Étape 4 — Générer le résumé

À partir du contenu extrait, rédiger un résumé en **2 phrases** en français qui synthétise les points clés de l'article. Le résumé doit être informatif et concis.

### Étape 5 — Construire le fichier markdown

Assembler le fichier avec :
- Le frontmatter YAML (titre, source, date_publication, type, résumé, tags)
- Le contenu markdown extrait
- Si un transcript a été extrait à l'étape 3bis, l'ajouter à la fin du contenu sous une section dédiée :

```markdown
---

## Transcript

> Source : YouTube / Podcast / Page (préciser l'origine du transcript)

[Contenu du transcript nettoyé, organisé en paragraphes]
```

Pour les pages de type `vidéo` ou `podcast` avec peu de contenu textuel, le transcript constitue le corps principal du document (pas besoin de section séparée, il remplace le contenu).

Le nom du fichier sera le titre de l'article, nettoyé des caractères spéciaux : remplacer `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|` par `-`, et tronquer à 100 caractères max. Extension `.md`.

### Étape 6 — Écrire le fichier

Utiliser le tool `Write` pour écrire directement le fichier dans le sous-dossier approprié de Veille/ (voir section "Sous-dossiers" ci-dessus) :

```
{vault}/Veille/{sous-dossier}/{nom-fichier-nettoyé}.md
```

Le sous-dossier est déterminé par le tag principal de l'article.

### Étape 7 — Mettre à jour l'index

Ajouter l'article au fichier `Veille/Index.md` :

1. Lire le fichier `Index.md`
2. Identifier la section correspondant au tag principal de l'article (le premier tag)
3. Insérer une nouvelle ligne dans le tableau de cette section, triée par date décroissante :
   ```
   | YYYY-MM-DD | [[Nom du fichier sans .md]] | `tag1` `tag2` |
   ```
   Si la date est inconnue, utiliser `—` à la place.
4. Mettre à jour le compteur d'articles dans l'en-tête (ligne `> XX articles`)
5. Mettre à jour la date de dernière mise à jour

### Étape 8 — Pertinence IAE

L'`Index.md` contient une section "Pertinence IAE" qui relie les articles aux produits/sujets du département. Les produits IAE sont :

- **Data Platform** (données & MCP) — articles sur MCP, données, État plateforme
- **Albert Code** — articles sur AI coding, productivité dev, vibe coding
- **Albert API / L'assistant IA** — articles sur infra IA, souveraineté, interfaces chat
- **OpenGateLLM** — articles sur open source, gouvernance communautaire
- **EvalAP** — articles sur évaluation de modèles, benchmarks
- **ALLiaNCE** (incubateur) — articles sur tendances marché, maturité agentique

Après avoir identifié les tags de l'article :
1. Déterminer quel(s) produit(s) IAE sont concernés (un article peut être pertinent pour plusieurs)
2. Ajouter une entrée sous chaque produit concerné dans la section "Pertinence IAE" de l'`Index.md`, au format :
   ```
   - [[Nom de l'article]] — Explication courte de la pertinence pour ce produit
   ```
3. Si aucun produit IAE n'est pertinent, ne rien ajouter

### Étape 9 — Confirmer

Afficher à l'utilisateur :
- Le titre de l'article
- Le type (article / vidéo / podcast / mixte)
- Les tags attribués
- La pertinence IAE (produits liés)
- Le résumé généré
- Si un transcript a été extrait : sa source et sa longueur approximative (nombre de mots)
- Le chemin du fichier créé

## Gestion d'erreurs

- **WebFetch échoue** : tenter automatiquement le fallback Chrome (étape 3). Si Chrome n'est pas connecté, informer l'utilisateur d'ouvrir Chrome avec l'extension Claude et réessayer.
- **Page JS-heavy / SPA (fallback Chrome)** : si l'extraction JS retourne un contenu vide ou très court (< 100 caractères), utiliser `get_page_text` comme fallback pour récupérer le texte brut, puis le formater en markdown.
- **Date introuvable** : laisser `date_publication` vide (`""`).
- **Titre introuvable** : utiliser le titre de la page comme fallback.
- **Écriture échouée** : informer l'utilisateur et proposer d'écrire dans Downloads/ à la place.
- **Doublon détecté** : ne jamais écraser silencieusement — toujours demander confirmation à l'utilisateur.
- **Transcript indisponible** : si la vidéo/audio n'a pas de sous-titres ou transcript accessible, informer l'utilisateur et proposer de continuer sans transcript. Ne jamais bloquer la sauvegarde de l'article pour cette raison.
- **Transcript très long (> 5000 mots)** : informer l'utilisateur de la taille et proposer soit le transcript complet, soit une version condensée (résumé structuré des points clés avec citations).
