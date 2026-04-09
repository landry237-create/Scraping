# 🏗️ Architecture Technique — Horizon Mobility DB

## Vue d'ensemble système

```
┌─────────────────────────────────────────────────────────────────┐
│                      HORIZON MOBILITY DB                        │
│                   Architecture Microservices                    │
└─────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                         COUCHE PRÉSENTATION                      │
│                                                                  │
│   ┌────────────────┐        ┌────────────────────────────────┐  │
│   │   Dashboard     │        │       API REST (FastAPI)        │  │
│   │  (HTML/JS/CSS) │◄──────►│  http://localhost:8000          │  │
│   │  /dashboard    │        │  Swagger: /docs                 │  │
│   └────────────────┘        └──────────┬─────────────────────┘  │
│                                         │                        │
└─────────────────────────────────────────│────────────────────────┘
                                          │
┌─────────────────────────────────────────▼────────────────────────┐
│                        COUCHE MÉTIER                             │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────────┐ │
│  │   Scrapers  │  │   Pipeline  │  │       Workers (Celery)    │ │
│  │             │  │             │  │                           │ │
│  │ - Europages │  │ - Cleaner   │  │ - scrape_task()           │ │
│  │ - Indeed    │  │ - Validator │  │ - validate_emails_task()  │ │
│  │ - YellowPg  │  │ - Dedup     │  │ - export_task()           │ │
│  │ - LinkedIn  │  │ - Classifier│  │ - campaign_task()         │ │
│  │ - Custom    │  │             │  │                           │ │
│  └──────┬──────┘  └──────┬──────┘  └──────────────────────────┘ │
│         │                │                       ▲               │
└─────────│────────────────│───────────────────────│───────────────┘
          │                │                       │
┌─────────▼────────────────▼───────────────────────│───────────────┐
│                        COUCHE DONNÉES                            │
│                                                                  │
│  ┌──────────────────┐   ┌──────────────┐   ┌───────────────┐    │
│  │   PostgreSQL     │   │    Redis     │   │  File System  │    │
│  │                  │   │              │   │               │    │
│  │  - companies     │   │ - Task Queue │   │  data/raw/    │    │
│  │  - contacts      │   │ - Cache      │   │  data/processed│   │
│  │  - campaigns     │   │ - Sessions   │   │  data/exports/ │   │
│  │  - scrape_jobs   │   │              │   │  logs/        │    │
│  │  - validations   │   └──────────────┘   └───────────────┘    │
│  └──────────────────┘                                           │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Flux de données

```
SOURCES EXTERNES          SCRAPERS          PIPELINE          BASE DE DONNÉES
─────────────────         ────────          ────────          ───────────────

Europages        ──►  EuropagesScraper ──►  Cleaner    ──►  companies table
Indeed           ──►  IndeedScraper    ──►  Validator  ──►  contacts table
Yellow Pages     ──►  YPScraper        ──►  Deduplicat ──►  validations table
LinkedIn         ──►  LinkedInScraper  ──►  Classifier ──►  
Custom URLs      ──►  CustomScraper    ──►             ──►  

                                                           ▼
                                                    EXPORTS / CAMPAGNES
                                                    ─────────────────
                                                    CSV / Excel
                                                    Lemlist JSON
                                                    Brevo JSON
                                                    Mailchimp JSON
                                                    HubSpot JSON
```

---

## 🗄️ Schéma de base de données

```sql
-- Table principale : entreprises
companies
├── id (UUID, PK)
├── name (VARCHAR 500)              -- Nom de l'entreprise
├── country (VARCHAR 3)             -- Code ISO (DE, FR, CA...)
├── city (VARCHAR 200)
├── region (VARCHAR 200)
├── sector (VARCHAR 100)            -- BTP, LOGISTICS, HEALTH...
├── sub_sector (VARCHAR 200)
├── website (VARCHAR 500)
├── source_url (TEXT)               -- URL de collecte
├── source_name (VARCHAR 100)       -- Nom du scraper source
├── priority_score (FLOAT)          -- Score 0-10 (ML-based)
├── created_at (TIMESTAMP)
├── updated_at (TIMESTAMP)
└── is_active (BOOLEAN)

-- Table contacts (liée à companies)
contacts
├── id (UUID, PK)
├── company_id (UUID, FK → companies)
├── email (VARCHAR 320)             -- Email professionnel
├── email_status (ENUM)             -- valid, invalid, risky, unknown
├── phone (VARCHAR 50)
├── contact_name (VARCHAR 300)
├── contact_role (VARCHAR 200)      -- DRH, Recruteur, Manager...
├── linkedin_url (VARCHAR 500)
├── source_url (TEXT)
├── validated_at (TIMESTAMP)
├── created_at (TIMESTAMP)
└── is_unsubscribed (BOOLEAN)       -- Conformité RGPD

-- Table campagnes
campaigns
├── id (UUID, PK)
├── name (VARCHAR 300)
├── country_filter (VARCHAR 3)
├── sector_filter (VARCHAR 100)
├── min_score (FLOAT)
├── platform (ENUM)                 -- lemlist, brevo, mailchimp
├── status (ENUM)                   -- draft, active, completed
├── contacts_count (INT)
├── sent_count (INT)
├── reply_count (INT)
├── created_at (TIMESTAMP)
└── exported_at (TIMESTAMP)

-- Table jobs de scraping
scrape_jobs
├── id (UUID, PK)
├── scraper_name (VARCHAR 100)
├── config (JSONB)                  -- Paramètres du job
├── status (ENUM)                   -- pending, running, done, error
├── records_scraped (INT)
├── records_valid (INT)
├── started_at (TIMESTAMP)
├── finished_at (TIMESTAMP)
└── error_log (TEXT)
```

---

## 🕷️ Architecture Scrapers

```
BaseScraper (classe abstraite)
│
├── Attributs communs
│   ├── proxy_manager        # Rotation IP automatique
│   ├── rate_limiter         # Respect des délais
│   ├── session              # Session HTTP avec retry
│   └── logger               # Logging structuré
│
├── Méthodes abstraites (à implémenter)
│   ├── scrape(config)       # Point d'entrée principal
│   ├── parse_company(html)  # Extraction données entreprise
│   └── get_next_page(resp)  # Pagination
│
├── Méthodes communes (héritées)
│   ├── fetch(url)           # GET avec retry + proxy
│   ├── fetch_js(url)        # GET via Playwright (JS-heavy)
│   ├── save_raw(data)       # Sauvegarder données brutes
│   └── send_to_pipeline()   # Envoyer au pipeline
│
└── Implémentations
    ├── EuropagesScraper     # B2B européen (≈ 3M entreprises)
    ├── IndeedScraper        # Job boards (emails RH)
    ├── YellowPagesScraper   # Pages jaunes multi-pays
    ├── LinkedInScraper      # Réseau professionnel
    └── CustomScraper        # Configurable via YAML
```

---

## ⚡ Architecture Workers (Celery)

```
Redis Broker
     │
     ▼
Celery App
     │
     ├── Queue: scraping (priorité haute)
     │   ├── scrape_source_task(source, config)
     │   └── refresh_company_task(company_id)
     │
     ├── Queue: pipeline (priorité moyenne)
     │   ├── validate_batch_task(contact_ids)
     │   ├── deduplicate_task()
     │   └── classify_task(company_ids)
     │
     └── Queue: export (priorité basse)
         ├── export_campaign_task(campaign_id)
         └── generate_report_task(filters)
```

---

## 🌐 API REST — Endpoints

```
/api/
├── contacts/
│   ├── GET    /              # Lister (paginé, filtrable)
│   ├── POST   /              # Créer
│   ├── GET    /{id}          # Détail
│   ├── PUT    /{id}          # Modifier
│   ├── DELETE /{id}          # Supprimer
│   └── POST   /bulk          # Import en masse
│
├── stats/
│   ├── GET    /overview      # KPIs globaux
│   ├── GET    /by-country    # Répartition par pays
│   ├── GET    /by-sector     # Répartition par secteur
│   └── GET    /scraping      # Historique jobs scraping
│
├── campaigns/
│   ├── GET    /              # Lister campagnes
│   ├── POST   /              # Créer campagne
│   ├── GET    /{id}          # Détail
│   └── POST   /{id}/export   # Exporter pour emailing
│
└── jobs/
    ├── POST   /scrape         # Lancer un scraping
    ├── GET    /{id}/status    # Statut d'un job
    └── POST   /{id}/cancel    # Annuler un job
```

---

## 🔐 Sécurité & Conformité

```
Sécurité Technique
├── Rate limiting sur l'API (100 req/min par IP)
├── Authentification API Key pour les endpoints sensibles
├── Validation stricte des inputs (Pydantic)
└── Logs d'audit pour toutes les opérations d'export

Conformité RGPD
├── Collecte limitée aux données professionnelles publiques
├── Horodatage et traçabilité de chaque donnée
├── Mécanisme d'opt-out (unsubscribe) intégré
├── Durée de rétention configurable (défaut: 24 mois)
└── Export des données d'une entité sur demande (droit d'accès)

Anti-Scraping Defense
├── Rotation automatique des User-Agents
├── Rotation des proxies (résidentiels recommandés)
├── Respect des fichiers robots.txt
├── Délais aléatoires entre requêtes (1-5 secondes)
└── Simulation comportement humain (Playwright)
```

---

## 📦 Stack Technologique

| Composant | Technologie | Justification |
|---|---|---|
| Language | Python 3.10+ | Ecosystème data/scraping |
| Web Framework | FastAPI | Performance, typage, Swagger auto |
| ORM | SQLAlchemy 2.0 | Flexible, migrations, async |
| Base de données | PostgreSQL 14 | Robustesse, JSONB, full-text search |
| Cache/Queue | Redis | Rapide, fiable pour Celery |
| Workers | Celery | Tâches distribuées, scheduling |
| Scraping statique | Scrapy + httpx | Performance, middleware, pipelines |
| Scraping JS | Playwright | Sites SPA/React/Vue |
| Parsing HTML | BeautifulSoup4 | Souplesse, lisibilité |
| Validation | Pydantic v2 | Typage strict, performance |
| Logging | Loguru | Structuré, coloré, rotatif |
| Dashboard | HTML/CSS/JS | Vanille, zéro dépendance frontend |
| Charts | Chart.js | Léger, performant |
| Containerisation | Docker Compose | Reproductibilité |

---

## 📈 Scalabilité

```
Phase 1 (0–50k contacts)     → SQLite + 1 worker
Phase 2 (50k–500k)           → PostgreSQL + 3 workers Celery
Phase 3 (500k–5M)            → PostgreSQL + Read Replicas + 10 workers
Phase 4 (5M–10M)             → PostgreSQL Cluster + Redis Cluster + 20+ workers
                               + Elasticsearch pour la recherche full-text
```

---

*Horizon Multiservices Guinée — Architecture v1.0 — Conakry 🇬🇳*