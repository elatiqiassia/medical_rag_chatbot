# Chatbot Médical Intelligent basé sur RAG — Contexte Marocain

Projet de fin de module — Programmation Python Avancée
Université Chouaïb Doukkali, Faculté des Sciences, El Jadida (AU 2025/2026)

## 🎯 Objectif

Chatbot médical basé sur une architecture **RAG (Retrieval-Augmented Generation)**,
s'appuyant uniquement sur des sources médicales marocaines officielles
(Ministère de la Santé, OMS Maroc, CHU, campagnes nationales), avec un support
bonus de la **Darija marocaine**.

## 🏗️ Architecture

```
Utilisateur → (Optionnel) Traduction Darija→Français → Embedding de la requête
→ Recherche vectorielle (FAISS) → Récupération des documents pertinents
→ LLM + Contexte → Réponse générée → (Optionnel) Génération en Darija
```

## 📁 Structure du projet

```
medical_rag_chatbot/
├── config.py                      # Configuration centrale (API, chunking, prompts)
├── main.py                        # Exécute tout le pipeline de bout en bout
├── requirements.txt
├── .env.example                   # Modèle pour vos clés API
│
├── data/
│   ├── raw/                       # Textes/PDF bruts collectés
│   ├── processed/                 # Textes nettoyés
│   ├── chunks/                    # chunks.jsonl (segments avec overlap)
│   └── index/                     # Index FAISS + métadonnées
│
├── src/
│   ├── scraping/
│   │   ├── web_scraper.py         # Scraping des pages web sources
│   │   └── pdf_extractor.py       # Extraction de texte depuis les PDF
│   ├── preprocessing/
│   │   ├── cleaner.py             # Nettoyage (bruit, en-têtes/pieds de page)
│   │   └── chunker.py             # Segmentation 300-800 tokens avec overlap
│   ├── indexing/
│   │   ├── embedder.py            # Génération des embeddings (API)
│   │   └── vector_store.py        # Construction/recherche index FAISS
│   ├── rag/
│   │   ├── retriever.py           # Récupération des chunks pertinents
│   │   ├── generator.py           # Génération de la réponse via LLM
│   │   └── pipeline.py            # Orchestration complète du RAG
│   ├── translation/
│   │   └── darija_translator.py   # Bonus : détection + traduction Darija
│   └── evaluation/
│       └── evaluator.py           # Évaluation qualité (pour le rapport)
│
├── app/
│   └── streamlit_app.py           # Interface web
│
└── tests/
    └── test_chunker.py
```

## 🚀 Fonctionnalités avancées

- **Recherche hybride** : combine recherche sémantique (FAISS) et recherche
  lexicale par mots-clés (BM25) via Reciprocal Rank Fusion, pour une
  meilleure pertinence sur les termes médicaux précis (sigles, noms de
  maladies). Désactivable via le paramètre `use_hybrid` ou la case à cocher
  de l'interface.
- **Citations obligatoires** : chaque réponse cite ses sources directement
  dans le texte (`[Source 1]`, `[Source 2]`...), pour une traçabilité claire.
- **Garde-fou de pertinence** : si aucune source indexée n'est suffisamment
  proche de la question posée, le chatbot répond directement qu'il ne
  dispose pas de l'information, sans appeler le LLM (évite les
  hallucinations sur des sujets hors périmètre).
- **Disclaimer systématique** : un rappel médical est ajouté après chaque
  réponse, indépendamment du contenu généré par le LLM.

## ⚙️ Installation

```bash
# 1. Créer un environnement virtuel
python -m venv venv
source venv/bin/activate   # Windows : venv\Scripts\activate

# 2. Installer les dépendances
pip install -r requirements.txt

# 3. Configurer les clés API
cp .env.example .env
# Éditez .env et renseignez GITHUB_TOKEN (voir section ci-dessous)
```

## 🔑 Obtenir une clé API gratuite (GitHub Models)

Ce projet utilise par défaut **GitHub Models**, un service officiel et gratuit
(avec limites de débit) compatible avec le SDK OpenAI :

1. Aller sur [github.com/settings/tokens](https://github.com/settings/tokens)
2. **Token classique** : cliquer "Generate new token (classic)", aucune case à
   cocher n'est nécessaire pour Github Models (les tokens classiques y ont accès
   par défaut).
   **Token fine-grained** (plus sécurisé) : dans *Account permissions*, activer
   **Models → Read-only**.
3. Copier le token généré dans `.env` sous `GITHUB_TOKEN`

> ⚠️ Les identifiants de modèles GitHub Models utilisent le préfixe du
> fournisseur, ex. `openai/gpt-4.1-mini`, `openai/text-embedding-3-small`.
> Si un modèle renvoie une erreur "model not found", vérifie l'ID exact sur
> [github.com/marketplace?type=models](https://github.com/marketplace?type=models)
> et ajuste `CHAT_MODEL` / `EMBEDDING_MODEL` dans `.env`.

> Vous pouvez remplacer `API_BASE_URL` dans `.env` pour utiliser directement
> l'API officielle OpenAI si vous disposez d'une clé payante (dans ce cas,
> retirer le préfixe `openai/` des noms de modèles).

## 🚀 Utilisation

### Pipeline complet (étape par étape)

```bash
python -m src.scraping.web_scraper       # 1. Scraping des pages + 2. téléchargement des PDF officiels
python -m src.scraping.pdf_extractor     # 3. Extraction du texte des PDF
python -m src.preprocessing.cleaner      # 4. Nettoyage
python -m src.preprocessing.chunker      # 5. Chunking
python -m src.indexing.vector_store      # 6. Construction de l'index vectoriel FAISS
python -m src.indexing.bm25_index        # 7. Construction de l'index lexical BM25
```

> `web_scraper.py` inclut déjà une liste de **10 PDF officiels vérifiés** du
> Ministère de la Santé et de la Protection Sociale (cardiovasculaire,
> hypertension, pied diabétique, vaccination, surveillance de la rougeole).
> Complétez `SOURCES`/`PDF_SOURCES` avec d'autres documents si besoin.

Ou tout en une seule commande :

```bash
python main.py
```

### Lancer l'interface web

```bash
streamlit run app/streamlit_app.py
```

L'interface (Streamlit ≥ 1.50) comprend : conversations multiples, réponses en flux avec
citations `[Source N]` en pastilles, cartes de sources (score, extrait, passage complet),
retour 👍/👎, export Markdown/JSON, tableau de bord de session (latence, pertinence,
documents sollicités, export CSV) et un indicateur d'état des index et de la clé API.
Lancez la commande depuis la racine du projet pour que `.streamlit/config.toml` (thème) soit pris en compte.

### Lancer l'évaluation des performances

```bash
python -m src.evaluation.evaluator
```

### Lancer les tests

```bash
python -m pytest tests/
```

## 📝 À compléter par l'étudiant

- [x] URLs réelles renseignées dans `src/scraping/web_scraper.py` (10 PDF du Ministère + 2 PDF OMS)
- [x] Motifs de nettoyage (`NOISE_PATTERNS`) enrichis pour sites gouvernementaux + PDF OMS
- [x] Jeu de questions de test enrichi (16 questions, 7 catégories) dans `src/evaluation/evaluator.py`
- [ ] Rédiger le rapport d'analyse des performances (livrable final) — lancer `python -m src.evaluation.evaluator` pour les vrais chiffres
- [ ] Adapter les motifs de nettoyage (`NOISE_PATTERNS`) aux sources réelles
- [ ] Enrichir le jeu de questions de test dans `src/evaluation/evaluator.py`
- [ ] Rédiger le rapport d'analyse des performances (livrable final)

## ⚠️ Avertissement

Ce chatbot fournit une information médicale générale à but éducatif.
Il ne remplace en aucun cas une consultation avec un professionnel de santé.
