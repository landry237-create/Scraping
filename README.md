# 🌍 Horizon Mobility DB — Plateforme de Prospection Internationale

**Horizon Multiservices Guinée | Département Mobilité Internationale**

Système complet de collecte, validation, enrichissement et prospection de contacts d'entreprises internationales pour le placement de la main-d'œuvre africaine.

---

## 📋 Table des matières

- [Vue d'ensemble](#vue-densemble)
- [Architecture](#architecture)
- [Prérequis](#prérequis)
- [Installation](#installation)
- [Configuration](#configuration)
- [Utilisation](#utilisation)
- [Scrapers disponibles](#scrapers-disponibles)
- [Pipeline de données](#pipeline-de-données)
- [API REST](#api-rest)
- [Dashboard](#dashboard)
- [Export & Campagnes](#export--campagnes)
- [KPIs & Monitoring](#kpis--monitoring)
- [Conformité RGPD](#conformité-rgpd)
- [FAQ](#faq)

---

## 🎯 Vue d'ensemble

Ce système permet à Horizon Multiservices Guinée de :

| Fonctionnalité | Description |
|---|---|
| 🕷️ **Scraping automatisé** | Collecte de millions de contacts via Scrapy + Playwright |
| 🧹 **Nettoyage des données** | Déduplication, standardisation, validation |
| ✉️ **Validation d'emails** | Vérification syntaxique, DNS, SMTP |
| 🗄️ **Base de données structurée** | PostgreSQL segmentée par pays/secteur |
| 📊 **Dashboard temps réel** | Interface web de suivi et analyse |
| 🚀 **Export campagnes** | CSV/JSON prêts pour Lemlist, Brevo, Mailchimp |
| ⚡ **Workers asynchrones** | Celery + Redis pour le traitement en masse |

---

## 🏗️ Architecture

```
horizon-mobility-db/
├── config/                  # Configuration centralisée
│   └── settings.py          # Variables d'environnement & paramètres
├── database/                # Couche de persistance
│   ├── models.py            # Modèles SQLAlchemy (ORM)
│   └── connection.py        # Gestion connexion DB + migrations
├── scrapers/                # Moteurs de collecte
│   ├── base_scraper.py      # Classe abstraite parente
│   ├── europages_scraper.py # Annuaire B2B européen
│   ├── indeed_scraper.py    # Offres d'emploi internationales
│   ├── yellowpages_scraper.py # Pages jaunes multi-pays
│   ├── linkedin_scraper.py  # LinkedIn (via API publique)
│   └── custom_scraper.py    # Scraper générique configurable
├── pipeline/                # Traitement & enrichissement
│   ├── cleaner.py           # Nettoyage et standardisation
│   ├── validator.py         # Validation emails, téléphones
│   ├── deduplicator.py      # Détection et fusion doublons
│   └── classifier.py        # Classification secteur/pays/priorité
├── api/                     # API REST FastAPI
│   ├── main.py              # Point d'entrée API
│   ├── schemas.py           # Schémas Pydantic
│   └── routes/              # Endpoints
│       ├── contacts.py      # CRUD contacts
│       ├── stats.py         # Statistiques & KPIs
│       └── campaigns.py     # Gestion campagnes
├── workers/                 # Tâches asynchrones
│   ├── celery_app.py        # Configuration Celery
│   └── tasks.py             # Tâches background
├── exports/                 # Génération d'exports
│   └── exporter.py          # CSV, Excel, JSON, Mailchimp
├── utils/                   # Utilitaires partagés
│   ├── logger.py            # Logging structuré
│   ├── proxy_manager.py     # Rotation de proxies
│   └── email_validator.py   # Moteur validation emails
├── dashboard/               # Interface web
│   └── index.html           # Dashboard complet (SPA)
├── data/                    # Données locales
│   ├── raw/                 # Données brutes scrappées
│   ├── processed/           # Données nettoyées
│   └── exports/             # Fichiers d'export
├── logs/                    # Journaux système
├── main.py                  # Point d'entrée principal CLI
├── requirements.txt         # Dépendances Python
├── docker-compose.yml       # Stack Docker complète
├── .env.example             # Template variables d'environnement
├── README.md                # Ce fichier
└── architecture.md          # Architecture détaillée
```

---

## ⚙️ Prérequis

- **Python** 3.10+
- **PostgreSQL** 14+ (ou SQLite pour dev)
- **Redis** 7+ (pour Celery)
- **Docker & Docker Compose** (recommandé)
- **Playwright** (pour le scraping JS-heavy)

---

## 🚀 Installation

### Option 1 — Docker (Recommandé)

```bash
# 1. Cloner le projet
git clone https://github.com/horizon-mg/mobility-db.git
cd horizon-mobility-db

# 2. Copier et configurer les variables d'environnement
cp .env.example .env
nano .env  # Remplir les valeurs

# 3. Lancer toute la stack
docker-compose up -d

# 4. Initialiser la base de données
docker-compose exec api python main.py init-db

# 5. Accéder au dashboard
open http://localhost:8000/dashboard
```

### Option 2 — Installation locale

```bash
# 1. Créer un environnement virtuel
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 2. Installer les dépendances
pip install -r requirements.txt

# 3. Installer Playwright
playwright install chromium

# 4. Configurer l'environnement
cp .env.example .env
nano .env

# 5. Initialiser la base de données
python main.py init-db

# 6. Démarrer l'API
python main.py serve

# 7. Démarrer les workers (dans un autre terminal)
python main.py worker
```

---

## ⚙️ Configuration

Copier `.env.example` vers `.env` et remplir :

```env
# Base de données
DATABASE_URL=postgresql://user:password@localhost:5432/horizon_db

# Redis (pour Celery)
REDIS_URL=redis://localhost:6379/0

# Proxies (optionnel mais recommandé)
PROXY_LIST=proxy1:port,proxy2:port

# Rate limiting
SCRAPE_DELAY_MIN=1.0
SCRAPE_DELAY_MAX=3.0

# Email validation
VALIDATE_EMAIL_SMTP=true

# API Keys (services tiers)
HUNTER_API_KEY=your_key
APOLLO_API_KEY=your_key
```

---

## 🕷️ Utilisation

### Lancer un scraper

```bash
# Scraper Europages (annuaire B2B)
python main.py scrape --source europages --country DE --sector BTP --limit 10000

# Scraper Indeed (offres d'emploi)
python main.py scrape --source indeed --country CA --sector logistics

# Scraper générique (URL custom)
python main.py scrape --source custom --url "https://example.com/companies"

# Lancer TOUS les scrapers en parallèle
python main.py scrape --all --countries DE,FR,BE,CA --sectors BTP,logistics,health
```

### Pipeline de traitement

```bash
# Nettoyer les données brutes
python main.py pipeline --step clean

# Valider les emails
python main.py pipeline --step validate

# Dédupliquer
python main.py pipeline --step deduplicate

# Pipeline complet
python main.py pipeline --all
```

### Export pour campagnes

```bash
# Export CSV segmenté par pays
python main.py export --format csv --segment country

# Export JSON pour Lemlist
python main.py export --format lemlist --country DE --sector BTP --limit 5000

# Export Excel avec scoring
python main.py export --format excel --min-score 7
```

---

## 📊 Scrapers disponibles

| Source | Type | Pays | Données |
|---|---|---|---|
| Europages | Annuaire B2B | EU | Nom, email, tél, secteur |
| Indeed | Job board | Mondial | Entreprise, email RH |
| Yellow Pages | Annuaire | US, CA, AU | Coordonnées complètes |
| LinkedIn (public) | Réseau pro | Mondial | Entreprise, contacts |
| Custom | URL générique | Configurable | Configurable |

---

## 🔌 API REST

L'API tourne sur `http://localhost:8000`

| Endpoint | Méthode | Description |
|---|---|---|
| `/api/contacts` | GET | Lister les contacts (paginé, filtrable) |
| `/api/contacts/{id}` | GET | Détail d'un contact |
| `/api/contacts` | POST | Ajouter un contact |
| `/api/stats` | GET | KPIs globaux |
| `/api/stats/by-country` | GET | Stats par pays |
| `/api/campaigns` | GET/POST | Gestion campagnes |
| `/api/export` | POST | Déclencher un export |
| `/docs` | GET | Documentation Swagger |

---

## 📈 KPIs & Objectifs

| Phase | Délai | Objectif |
|---|---|---|
| Phase 1 | 2–4 semaines | 10 000 – 50 000 contacts vérifiés |
| Phase 2 | 1–2 mois | 100 000 – 500 000 contacts qualifiés |
| Phase 3 | 3–6 mois | 1 million+ contacts structurés |
| Objectif final | 12 mois | 5–10 millions d'enregistrements |

---

## 🛡️ Conformité RGPD

- Seules les données **professionnelles publiques** sont collectées
- Chaque email collecté inclut sa **source et date de collecte**
- Système de **désabonnement** (unsubscribe) intégré
- Les données sont segmentées pour permettre la **suppression ciblée**
- Respect des délais `robots.txt` et des CGU des sources

---

## 🐛 FAQ

**Q: Le scraping est-il légal ?**
> Oui, dans les limites fixées par les CGU des sites et le RGPD. Ce système scrape uniquement des données professionnelles publiques.

**Q: Comment gérer les blocages IP ?**
> Configurer une liste de proxies dans `.env`. Le `ProxyManager` effectue une rotation automatique.

**Q: Peut-on intégrer avec HubSpot ?**
> Oui, via l'export JSON + l'API HubSpot. Un connecteur dédié est en roadmap.

---

*Développé pour Horizon Multiservices Guinée — Conakry, Guinée 🇬🇳*
