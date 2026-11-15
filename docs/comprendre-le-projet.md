# Comprendre le projet de A à Z

Guide de lecture du dépôt pour quelqu'un qui découvre à la fois le code **et** les
concepts (RAG, embeddings, agent LLM…). À lire dans l'ordre.

---

## 1. Ce que fait le projet, en une phrase

C'est un **chatbot de support client pour CAGECFI** (société togolaise d'ingénierie
informatique, produit phare : le logiciel *Perfect-Vision*). Il répond aux questions
des visiteurs du site **uniquement à partir d'une base documentaire maison** —
plaquettes commerciales, pages du site, FAQ — et jamais à partir de ce que le modèle
« croit savoir ».

La contrainte centrale du projet, celle qui explique 80 % des choix techniques :
**interdiction d'inventer**. Un chatbot d'entreprise qui invente un tarif ou un
numéro de téléphone est pire qu'un chatbot absent.

> ⚠️ Le dossier s'appelle `MongoDB-RAG-Agent` pour des raisons historiques : le
> projet est parti d'un template MongoDB. **Il n'y a plus de MongoDB.** La base
> utilisée est PostgreSQL, hébergée chez Supabase. Voir §9.

---

## 2. Les concepts, expliqués sans jargon

### 2.1 LLM (Large Language Model)

Un modèle de langage (ici `gpt-4o-mini` d'OpenAI) est un programme qui prédit du
texte. Il est excellent pour **rédiger**, mauvais pour **savoir**. Il ne connaît pas
CAGECFI, et si on lui pose la question, il inventera une réponse plausible. C'est ce
qu'on appelle une **hallucination**.

### 2.2 RAG (Retrieval-Augmented Generation)

La parade. « Génération augmentée par la recherche », en 3 temps :

1. **Retrieval** (recherche) — on cherche dans nos documents les passages qui
   ressemblent à la question.
2. **Augmentation** — on colle ces passages dans le message envoyé au LLM.
3. **Generation** — on demande au LLM : « réponds en te basant *uniquement* sur ça ».

Le LLM ne sert plus que de **rédacteur**. Le savoir vient de nos fichiers. C'est tout
le principe du projet.

### 2.3 Chunk (morceau)

On ne peut pas envoyer 42 documents entiers au LLM à chaque question : trop long,
trop cher, et il se perd. On découpe donc chaque document en **chunks** de ~512
tokens (≈ 350 mots). Un chunk est l'unité de recherche : c'est ce qu'on retrouve et
ce qu'on injecte dans le prompt.

Le découpage n'est pas naïf : le `HybridChunker` de Docling coupe **aux frontières
logiques** (titres, paragraphes, tableaux) et rajoute la hiérarchie des titres en
tête de chaque chunk. Ainsi un chunk isolé garde son contexte : on sait qu'il parle
de « PERFECT > Fonctionnalités > Épargne » et pas d'un texte flottant.

### 2.4 Embedding (vecteur sémantique)

Un **embedding** transforme un texte en une liste de nombres — ici **1536 nombres**
(le modèle `text-embedding-3-small` d'OpenAI). Cette liste est un « point » dans un
espace à 1536 dimensions.

Propriété magique : **deux textes de sens proche donnent deux points proches**, même
sans mot commun. « Combien coûte une licence ? » et « Quel est le tarif du
logiciel ? » atterrissent presque au même endroit.

On mesure la proximité par la **similarité cosinus** (l'angle entre les deux
vecteurs). En SQL, `pgvector` fournit l'opérateur `<=>` : c'est la *distance*
cosinus (0 = identiques). Le code la reconvertit en score de similarité 0–1 :

```sql
1 - (c.embedding <=> $1::vector) / 2 AS similarity
```

### 2.5 Recherche vectorielle vs recherche plein texte

| | Recherche **vectorielle** (sémantique) | Recherche **plein texte** (mots-clés) |
|---|---|---|
| Principe | proximité de sens | présence des mots |
| Forte sur | reformulations, synonymes | noms propres, sigles, références |
| Faible sur | sigles rares (`SYCEBNL`, `ISO 8583`) | question posée avec d'autres mots |

Les deux sont complémentaires. PostgreSQL fait la seconde nativement avec
`to_tsvector('french', …)` — le `'french'` compte : il gère la racinisation
(« formation » ↔ « formations ») et les mots vides français.

### 2.6 Recherche hybride + RRF

On lance **les deux recherches en parallèle**, puis on fusionne les deux
classements avec **RRF** (*Reciprocal Rank Fusion*) :

```
score(doc) = Σ  1 / (k + rang_dans_la_liste_i)      avec k = 60
```

Traduction : un document bien classé dans **les deux** listes remonte au sommet.
L'astuce du RRF est qu'il n'utilise que le **rang**, jamais le score brut — ce qui
évite d'avoir à comparer une similarité cosinus (0–1) avec un `ts_rank` (échelle
totalement différente). Élégant et robuste. Implémentation :
[tools_supabase.py:222](../src/tools_supabase.py#L222).

### 2.7 Agent et « tools »

Un **agent** est un LLM à qui on donne des **outils** (des fonctions Python) qu'il
peut décider d'appeler lui-même. Ici l'outil est `search_knowledge_base`. Le
scénario : l'utilisateur demande « Qui est CAGECFI ? » → le LLM répond « j'appelle
l'outil de recherche avec cette question » → le code exécute la recherche → le
résultat est renvoyé au LLM → il rédige la réponse finale.

C'est **2 allers-retours** avec le LLM. Le projet propose aussi un mode plus direct
(voir §6.2).

### 2.8 System prompt

Le mode d'emploi permanent donné au LLM avant toute conversation. Ici il tient dans
[prompts.py](../src/prompts.py) et code les règles métier : réponds en français,
ne salue pas si on te pose une question, ne réponds jamais hors sujet, et si tu ne
trouves rien → donne l'e-mail et le téléphone de CAGECFI.

### 2.9 Async / `await`

Toutes les opérations lentes (réseau, base de données) sont **asynchrones**. Pendant
qu'on attend la réponse d'OpenAI, le programme peut faire autre chose — par exemple
lancer la recherche plein texte en même temps que la recherche vectorielle
(`asyncio.gather`). D'où les `async def` et `await` partout.

### 2.10 Pydantic

Bibliothèque qui valide les données à l'exécution à partir des annotations de type.
Deux usages ici :
- **Modèles de données** (`SearchResult`, `ProductSheet`) : garantissent la forme des
  objets qui circulent.
- **Pydantic Settings** ([settings_supabase.py](../src/settings_supabase.py)) : lit
  le fichier `.env` et le transforme en objet typé. Si une clé obligatoire manque,
  l'application refuse de démarrer avec un message clair — plutôt que de planter
  bizarrement plus tard.

---

## 3. La carte du dépôt

```
MongoDB-RAG-Agent/
│
├── src/                     ← ⭐ LE CODE DU PROJET (version Supabase, active)
│   ├── settings_supabase.py     configuration lue depuis .env
│   ├── providers_supabase.py    fabrique l'objet « modèle LLM »
│   ├── dependencies_supabase.py connexion PostgreSQL + client embeddings
│   ├── prompts.py               les consignes données au LLM
│   ├── tools_supabase.py        ⭐ les 6 fonctions de recherche + le RRF
│   ├── agent_supabase.py        l'agent Pydantic AI et ses outils
│   ├── rag_chat.py              ⭐ le mode « recherche forcée » (utilisé en prod)
│   ├── api.py                   serveur web FastAPI (/chat, /health, /)
│   ├── cli_supabase.py          interface terminal (Rich)
│   ├── streamlit_app_supabase.py interface web locale (Streamlit)
│   └── ingestion/               ⭐ la chaîne qui remplit la base (§5)
│
├── api/index.py             ← point d'entrée Vercel (3 lignes utiles)
├── frontend/index.html      ← page de démo + widget de chat (619 lignes, autonome)
├── vercel.json              ← routage : toutes les URL → api/index.py
│
├── documents/               ← la matière première
│   ├── cagecfi/                 53 pages du site, converties en Markdown
│   ├── plaquettes/              PDF bruts téléchargés (ignoré par git, 255 Mo)
│   ├── plaquettes_md/           42 Markdown extraits (21 plaquettes + 21 fiches)
│   └── plaquettes_audit.json    rapport d'audit des PDF
│
├── tests/                   ← 63 tests pytest
├── docs/                    ← documentation (dont ce fichier)
├── .claude/                 ← configuration Claude Code (plans, commandes, refs)
│
├── supabase_setup_cagecfi.sql  ← ⭐ le schéma SQL à exécuter dans Supabase
├── pyproject.toml           ← dépendances et configuration du paquet
├── .env                     ← 🔑 secrets et configuration (jamais commité)
├── README.md                ← guide d'installation
├── CLAUDE.md                ← consignes de développement pour l'IA
└── DEPLOY_VERCEL.md         ← procédure de déploiement
```

### L'héritage MongoDB, désormais supprimé

Le projet est parti d'un template MongoDB Atlas. Il en restait deux dossiers de code
mort — `examples/` (la version MongoDB complète) et `test_scripts/` (des scripts qui
importaient des modules disparus depuis la migration). **Ils ont été supprimés**, avec
les doublons de dépendances (`requirements*.txt`) et la doc `mongodb-patterns.md`.

Ce qu'il faut en retenir si tu croises une mention de MongoDB quelque part :

| | Ancien template | Projet actuel |
|---|---|---|
| Base | MongoDB Atlas | PostgreSQL / Supabase |
| Pilote | Motor | asyncpg |
| Recherche hybride | `$rankFusion` (natif Mongo) | RRF codé en Python |

Seuls le **nom du dépôt** et quelques documents d'archive (`.claude/PRD.md`,
`.claude/plans/`) portent encore la trace de MongoDB. En cas de doute, **`src/` fait
foi**.

---

## 4. La base de données

Tout est dans [supabase_setup_cagecfi.sql](../supabase_setup_cagecfi.sql), à
exécuter une fois dans l'éditeur SQL de Supabase. Deux tables :

### `cagecfi_documents` — le texte intégral

| Colonne | Rôle |
|---|---|
| `id` | UUID de la ligne |
| `file_id` | UUID **du fichier** — partagé par toutes ses parties |
| `title`, `source` | titre et chemin d'origine |
| `content` | le texte de cette partie (~2000 caractères) |
| `part_number` / `total_parts` | « partie 3 sur 7 » |
| `metadata` | JSON libre (provenance de l'extraction, etc.) |

Un document long est découpé en **parties** de 2000 caractères, toutes reliées par le
même `file_id`. C'est ce qui permet de reconstituer le document complet.

### `cagecfi_chunks` — les morceaux vectorisés

| Colonne | Rôle |
|---|---|
| `id`, `file_id` | identité et rattachement au fichier |
| `content` | le texte du chunk |
| `embedding` | `vector(1536)` — le vecteur sémantique |
| `token_count`, `metadata` | mesures et provenance |

### Les index (et pourquoi ils comptent)

```sql
-- Index vectoriel HNSW : trouve les voisins les plus proches sans tout parcourir
CREATE INDEX … USING hnsw (embedding vector_cosine_ops) WITH (m=16, ef_construction=64);

-- Index plein texte français
CREATE INDEX … USING GIN (to_tsvector('french', content));
```

**HNSW** (*Hierarchical Navigable Small World*) est un graphe de navigation :
au lieu de comparer la question aux 364 chunks un par un, on saute de voisin en
voisin. Sur 364 chunks c'est du luxe ; sur 100 000 c'est indispensable.

> ⚠️ Piège majeur : la colonne `embedding` est déclarée `vector(1536)`. Si on change
> de modèle d'embedding (par ex. retour à Ollama `nomic-embed-text` en 768
> dimensions), **tous les vecteurs existants deviennent inutilisables** — il faut
> `DROP TABLE` puis tout ré-ingérer. Le commentaire est dans le fichier SQL.

---

## 5. La chaîne d'ingestion (remplir la base)

C'est la partie la plus élaborée du projet, et la plus instructive.

### 5.1 Le problème réel qui a dicté la conception

Les plaquettes commerciales de CAGECFI sont des PDF exportés depuis un outil de PAO :
**94 % des pages ne contiennent aucun texte extractible**, ce sont des images. Une
ingestion naïve aurait rempli la base de documents vides **sans lever la moindre
erreur** — le chatbot aurait répondu « je n'ai pas cette information » à tout, sans
qu'on comprenne pourquoi.

Pire : deux PDF contiennent une fausse couche texte faite de glyphes non mappés
(`/gid00010/gid00023/…`). La bibliothèque `pypdf` y compte les lettres `g`, `i`, `d`
et conclut « ce document est riche en texte » — `INTEROPERABILITE.pdf` affichait
**34 430 caractères pour 19 mots réels**.

D'où une chaîne en 6 étapes, chacune produisant un **artefact sur disque** relisible
et chaque étape rejouable seule.

### 5.2 Les 6 étapes

```
Google Drive
     │  ① drive_source.py — télécharge les 32 PDF, vérifie l'intégrité (%%EOF)
     ▼
documents/plaquettes/*.pdf
     │  ② pdf_audit.py — mesure les caractères/page, classe TEXTE / MIXTE / IMAGE,
     │                    dédoublonne par MD5, exclut les doublons et l'anglais
     ▼
documents/plaquettes_audit.json      →  32 audités, 21 retenus
     │  ③ extract_plaquettes.py — routage :
     │        TEXTE (≥400 car./page) → Docling sans OCR   (4 documents)
     │        MIXTE / IMAGE          → Mistral OCR        (17 documents)
     ▼
documents/plaquettes_md/*.md         →  8 078 mots réels, 0 glyphe résiduel
     │
     │  ⚠️  RELECTURE HUMAINE  ⚠️  ← le point de contrôle : on lit le Markdown
     │                               avant de le vectoriser
     │  ④ product_sheet.py — un LLM résume chaque plaquette en fiche produit
     │                        structurée (nom, catégorie, fonctionnalités…)
     ▼
documents/plaquettes_md/*_fiche.md   →  21 fiches
     │  ⑤ ingest_supabase.py — découpe, calcule les embeddings, écrit en base
     ▼
Supabase                             →  42 documents, 364 chunks
     │  ⑥ verify_ingestion.py — recette bloquante :
     │        · chaque document a-t-il au moins un chunk ?
     │        · le nombre de documents en base = celui attendu par l'audit ?
     ▼
                                     →  42/42 ✅
```

### 5.3 Pourquoi ce routage et pas un réglage uniforme

Mesuré, pas supposé : **l'OCR dégrade** les documents qui ont déjà du texte (il fusionne
les espaces et perd les accents) et **sauve** ceux qui n'en ont pas. Sur l'unique
document MIXTE du corpus, Docling extrayait 15 mots mutilés (`PERFECT- VI SI O N`)
là où Mistral OCR en tire 177 mots propres. Un réglage unique est perdant dans un
sens ou dans l'autre.

Détail pratique : au-delà de 20 Mo, l'API Mistral renvoie une erreur 503 sur le PDF
entier. Le repli, dans [mistral_ocr.py](../src/ingestion/mistral_ocr.py) : rendre
chaque page en JPEG avec `pypdfium2` (0,3–0,4 Mo par page) et les envoyer une par une.

### 5.4 Pourquoi des « fiches produit » en plus des chunks

À la question « Que fait Perfect-Vision ? », un fragment de plaquette isolé répond
mal. Une fiche de synthèse répond bien. Les fiches sont donc indexées **en plus** des
chunks bruts, qui restent nécessaires aux questions de détail. Les deux cohabitent en
base, distingués par `doc_type` dans les métadonnées.

### 5.5 L'autre source : le site web

[crawl_cagecfi.py](../src/ingestion/crawl_cagecfi.py) parcourt le sitemap WordPress de
`cagecfi.com`, télécharge chaque page et l'exporte en Markdown dans
`documents/cagecfi/` (53 fichiers). Chaîne beaucoup plus simple : pas d'OCR, le HTML
est déjà du texte.

### 5.6 Ce que fait `ingest_supabase.py` en détail

C'est le plus gros fichier du projet (877 lignes). Pour chaque fichier trouvé :

1. **Lire** — selon l'extension : Markdown (avec extraction du *front-matter* YAML),
   PDF/Word/PowerPoint/Excel/HTML via Docling, audio via Whisper.
2. **Extraire le titre** et les métadonnées.
3. **Découper en parties** de 2000 caractères → table `cagecfi_documents`.
4. **Chunker** avec le `HybridChunker` (512 tokens, conscient de la structure).
5. **Vectoriser** par lots de 100 chunks (un appel API par lot, pas par chunk).
6. **Écrire** chunks + embeddings dans `cagecfi_chunks`.

Le tout dans une **transaction** : si une insertion échoue, rien n'est écrit à moitié.

Lancement :
```powershell
uv run python -m src.ingestion.ingest_supabase -d ./documents
```

---

## 6. Le chemin d'une question (runtime)

Le projet a **deux implémentations du chat**. C'est déroutant au début, alors voici
la distinction.

### 6.1 Mode « agent » — `src/agent_supabase.py`

Utilisé par la **CLI** et l'**app Streamlit**.

```
question ──▶ LLM (passe 1) ──▶ « j'appelle search_knowledge_base(...) »
                                          │
                                          ▼
                              hybrid_search → PostgreSQL
                                          │
             résultats ◀──────────────────┘
                  │
                  ▼
             LLM (passe 2) ──▶ réponse rédigée
```

Avantage : le LLM décide s'il faut chercher (inutile pour « Bonjour »). Inconvénient :
**2 appels LLM** — lent et coûteux —, et les petits modèles décident mal.

### 6.2 Mode « recherche forcée » — `src/rag_chat.py`

Utilisé par l'**API web** (donc en production sur Vercel). C'est le mode retenu.

```
question
   │
   ├── salutation détectée par regex ? ──▶ réponse instantanée, 0 appel LLM
   │
   └── sinon ──▶ hybrid_search (toujours) ──▶ contexte injecté
                                                    │
                                                    ▼
                                          LLM (une seule passe) ──▶ réponse
```

Plus rapide (1 appel au lieu de 2), plus fiable (on ne dépend pas du bon vouloir du
modèle), et les salutations coûtent zéro. Si la recherche ne remonte rien, on répond
la phrase de repli **sans même appeler le LLM** — impossible d'halluciner.

### 6.3 Les garde-fous anti-hallucination, récapitulés

1. Le **system prompt** interdit explicitement d'inventer et de sortir du périmètre.
2. La **température à 0.1** rend le modèle quasi déterministe (peu créatif).
3. Le résultat de recherche est **encadré d'instructions** (`« Utilise UNIQUEMENT les
   informations ci-dessous »`).
4. Zéro résultat → **réponse de repli codée en dur**, LLM court-circuité.
5. La phrase de repli donne toujours un **contact humain** (e-mail + téléphone).

---

## 7. Les interfaces

| Interface | Fichier | Lancement | Usage |
|---|---|---|---|
| **CLI** | `src/cli_supabase.py` | `uv run python -m src.cli_supabase` | tests rapides, debug |
| **Streamlit** | `src/streamlit_app_supabase.py` | `uv run streamlit run src/streamlit_app_supabase.py` | démo locale |
| **Web / API** | `src/api.py` + `frontend/index.html` | `uv run uvicorn src.api:app --reload --port 8000` | la vraie cible |

L'API expose trois routes :
- `GET /` → sert `frontend/index.html`
- `POST /chat` → `{message, history}` → texte brut de la réponse
- `GET /health` → `{"status": "ok"}` (surveillance)

`frontend/index.html` est une **page de démonstration autonome** (fictive, « NOVAFIN »)
qui simule un site client avec le widget de chat intégré. Elle appelle `/chat` en
`fetch`. Aucun framework, aucune compilation : un seul fichier HTML.

---

## 8. Le déploiement (Vercel)

**Serverless** signifie : pas de serveur allumé en permanence. À chaque requête,
Vercel démarre une fonction, exécute le code, l'éteint. On paie à l'usage, mais la
première requête après une pause est lente (*cold start*).

Conséquences visibles dans le code :

- `api/index.py` — 3 lignes utiles : ajoute la racine au `sys.path` et expose `app`.
  Vercel détecte automatiquement l'objet ASGI nommé `app`.
- `vercel.json` — un *rewrite* catch-all : toutes les URL vont vers `api/index.py`.
- `pyproject.toml` — les dépendances lourdes (Docling, Whisper, Streamlit) sont en
  **extras optionnels**, pas dans `dependencies`. Vercel installe depuis
  `pyproject.toml` ; une dépendance de trop et le build dépasse la limite de taille.
  ```powershell
  uv pip install -e ".[ingestion]"   # ingestion locale
  uv pip install -e ".[ui]"          # CLI + Streamlit
  ```
- `rag_chat.py` construit l'agent **paresseusement** (au premier appel, pas à
  l'import) : importer le module ne doit avoir aucun effet de bord ni exiger les
  variables d'environnement, sinon le build échoue.
- Le *warmup* au démarrage est **désactivé par défaut** (`ENABLE_WARMUP=0`) : sur
  serverless, chaque cold start relancerait un appel LLM facturé pour rien.
- `statement_cache_size=0` sur le pool `asyncpg` : le *pooler* pgbouncer de Supabase
  ne supporte pas les *prepared statements*.

En production, la configuration ne vient **pas** du `.env` mais des variables
d'environnement du dashboard Vercel.

---

## 9. Configuration : ce qui est réellement branché

Le `.env` (non commité) contient la vérité. État actuel :

| Réglage | Valeur |
|---|---|
| LLM | OpenAI **`gpt-4o-mini`** |
| Embeddings | OpenAI **`text-embedding-3-small`** (1536 dimensions) |
| Base | Supabase PostgreSQL + pgvector |
| Tables | `cagecfi_documents`, `cagecfi_chunks` |
| Température | 0.1 |
| OCR | Mistral (`mistral-ocr-latest`), ingestion locale uniquement |

> ⚠️ **Trois sources se contredisent** sur ce point, attention :
> - le `README.md` décrit **Ollama** (`qwen2.5:7b` + `nomic-embed-text`, 768 dim.),
> - les valeurs par défaut de `settings_supabase.py` pointent aussi vers Ollama,
> - le `.env` réel utilise **OpenAI**.
>
> C'est l'héritage d'une migration « local → cloud » faite pour permettre le
> déploiement Vercel (Ollama tourne sur ta machine, Vercel n'y a pas accès). Le
> `.env` gagne toujours : Pydantic Settings écrase les valeurs par défaut.

Le code reste **agnostique au fournisseur** : tout passe par une API compatible
OpenAI. Changer de fournisseur = changer trois lignes du `.env` (à condition de
ré-ingérer si la dimension des embeddings change).

---

## 10. Les tests

63 tests avec **pytest**, dans `tests/`, qui reflètent l'arborescence de `src/`.

```powershell
uv run pytest tests/ -v
uv run pytest tests/ -m unit          # sans réseau
uv run pytest tests/ -m integration   # avec service externe
```

Ils couvrent presque exclusivement la chaîne d'ingestion — c'est là que sont les
règles métier subtiles (routage, exclusions, troncature, réconciliation), donc là que
les tests rapportent le plus.

`docs/tests-chatbot-cagecfi.md` est d'une autre nature : une **liste de questions de
recette manuelle**, chacune ancrée sur un fait réellement présent en base (« CAGECFI
a été créée en **2001** », « capital social **200 000 000 F CFA** »). Toute réponse qui
s'en écarte signale soit une extraction fautive, soit un problème de recherche, soit
une hallucination.

---

## 11. Les outils du quotidien

- **UV** — gestionnaire de paquets Python (comme `pip`, mais beaucoup plus rapide).
  `uv run <cmd>` exécute dans l'environnement du projet sans l'activer.
- **Docling** — convertit PDF / Word / PowerPoint / Excel / HTML / audio en Markdown
  propre, et fournit le `HybridChunker`.
- **Pydantic AI** — le framework d'agent (modèle + outils + prompt système + streaming).
- **asyncpg** — client PostgreSQL asynchrone, avec pool de connexions.
- **Rich** — jolis affichages dans le terminal (la CLI).
- **FastAPI** — le serveur web.
- **`.claude/`** — configuration de Claude Code : plans de développement, commandes
  personnalisées, documents de référence. N'a aucun effet sur l'exécution du chatbot.

---

## 12. Par où commencer à lire le code

Dans cet ordre, c'est le chemin le plus court vers la compréhension :

1. [settings_supabase.py](../src/settings_supabase.py) — 130 lignes, la config. Tu vois
   tout de suite tous les réglages du projet.
2. [prompts.py](../src/prompts.py) — 72 lignes, les règles métier en français.
3. [rag_chat.py](../src/rag_chat.py) — 121 lignes, **le cœur** : le trajet complet
   d'une question.
4. [tools_supabase.py](../src/tools_supabase.py) — les 6 recherches et le RRF. C'est
   ici qu'on voit le SQL réel.
5. [api.py](../src/api.py) — comment tout ça devient un serveur web.
6. [ingestion/pdf_audit.py](../src/ingestion/pdf_audit.py) puis
   [ingestion/extract_plaquettes.py](../src/ingestion/extract_plaquettes.py) — la
   partie la plus intéressante en termes d'ingénierie.

Tout le code actif tient dans `src/` : il n'y a plus de dossier « legacy » à éviter.

---

## 13. Démarrage complet, de zéro

```powershell
# 1. Environnement
uv venv
.venv\Scripts\activate
uv pip install -e ".[ingestion,ui]"

# 2. Configuration — copier un .env et le remplir
#    (SUPABASE_URL, DATABASE_URL, LLM_API_KEY, EMBEDDING_API_KEY…)

# 3. Base de données — exécuter supabase_setup_cagecfi.sql
#    dans le SQL Editor de Supabase

# 4. Remplir la base
$env:PYTHONUTF8='1'; uv run python -m src.ingestion.ingest_supabase -d ./documents

# 5. Vérifier
uv run python -m src.ingestion.verify_ingestion

# 6. Tester en terminal
uv run python -m src.cli_supabase

# 7. Ou lancer le site + widget
uv run uvicorn src.api:app --reload --port 8000
#    puis http://localhost:8000
```

> 🪟 Sur Windows, la console (cp1252) ne sait pas afficher les emojis des scripts.
> Préfixer par `$env:PYTHONUTF8='1';` évite les `UnicodeEncodeError`.

---

## 14. Résumé en une image

```
   PRÉPARATION (hors ligne, une fois)          UTILISATION (à chaque question)
   ─────────────────────────────────           ──────────────────────────────

   Drive / site cagecfi.com                     Visiteur du site
          │                                            │
          ▼                                            ▼
   audit → OCR → Markdown                        frontend/index.html
          │                                            │  POST /chat
          ▼  relecture humaine                         ▼
   fiches produit                                 src/api.py
          │                                            │
          ▼                                            ▼
   chunks + embeddings                          src/rag_chat.py
          │                                            │
          ▼                                     ┌──────┴──────┐
   ┌─────────────────┐                          │  recherche  │
   │    SUPABASE     │ ◀────────────────────────┤   hybride   │
   │ pgvector + FTS  │ ─────── passages ───────▶│    (RRF)    │
   └─────────────────┘                          └──────┬──────┘
                                                       ▼
                                                gpt-4o-mini rédige
                                                       │
                                                       ▼
                                                réponse ancrée,
                                                sans invention
```
