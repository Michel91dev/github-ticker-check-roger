# 📈 Analyse Boursière — Ticker-Check-Roger v3.0.8

Application d'analyse d'actions boursières personnelle pour **Michel, Roger et Romain**.
**Hébergée sur VPS Hostinger** (Docker + Streamlit), données via Yahoo Finance, persistance MySQL avec sessions 30 jours.

---

## Table des matières

1. [Fonctionnalités](#fonctionnalités)
2. [Architecture du projet](#architecture-du-projet)
3. [Interface utilisateur](#interface-utilisateur)
4. [Indicateurs techniques](#indicateurs-techniques)
5. [Persistance MySQL — Infrastructure](#persistance-mysql--infrastructure)
6. [Configuration Streamlit Cloud](#configuration-streamlit-cloud)
7. [Installation et lancement local](#installation-et-lancement-local)
8. [Déploiement](#déploiement)
9. [Gestion des versions](#gestion-des-versions)
10. [Dépendances](#dépendances)
11. [Maintenance et opérations](#maintenance-et-opérations)

---

## Fonctionnalités

### Sidebar de navigation
- **Sélection d'utilisateur** : Michel, Roger, Romain — chacun avec ses propres portefeuilles
- **Liste PEA / TITRES** : affichage en cartouches séparés par catégorie
- **Signal de recommandation coloré** : 🟢 Acheter / 🟡 Attente / 🔴 Vendre / ⚪ Neutre
- **Sélection visuelle** : boule de signal entourée en rouge sur la ligne active, flèches ▶▶▶ ◄◄◄
- **Tri des actions** : alphabétique, par signal Acheter en premier, par signal Vendre en premier
- **Affichage ISIN** : optionnel, affiché entre parenthèses dans chaque cartouche

### Gestion des ISIN
- **Cartouche "🔑 Gérer les ISIN"** : 2 onglets distincts
  - **➕ Ajouter** : saisir un nouveau ticker (Yahoo Finance), son nom, son ISIN et sa catégorie
  - **✏️ Modifier** : sélectionner un ticker existant pour corriger son ISIN ou le supprimer
- **Validation format** : regex `^[A-Z]{2}[A-Z0-9]{9,12}$`
- **Persistance MySQL** : chaque modification est immédiatement sauvegardée en base de données
- **Par utilisateur** : Michel, Roger et Romain ont chacun leurs propres ISIN indépendants
- **Fallback** : si MySQL inaccessible, les ISIN par défaut (hardcodés) sont utilisés sans erreur visible

### Analyse technique
- Graphique de cours interactif (Plotly) avec chandeliers ou ligne
- Moyennes mobiles MA50 / MA200
- MACD (Moving Average Convergence Divergence)
- RSI (Relative Strength Index)
- Bandes de Bollinger
- Golden Cross / Death Cross détectés automatiquement
- Volume de transactions

### Informations société
- Fiche descriptive de l'entreprise (secteur, pays, capitalisation, P/E ratio…)
- Actualités récentes récupérées via Yahoo Finance

---

## Architecture du projet

```
greg-sp500-demo/
├── streamlit_sp500_demo.py   # Application principale (code unique)
├── requirements.txt          # Dépendances Python (pip)
├── version.txt               # Numéro de version courante (ex: 2.6.1)
├── populate_isin.sql         # Script SQL de population initiale des ISIN
└── README.md                 # Cette documentation
```

### Flux de données

```
Yahoo Finance (yfinance)
        │
        ▼
streamlit_sp500_demo.py
        │
        ├── Calcul indicateurs techniques (MACD, RSI, BB, MA...)
        ├── Affichage graphiques (Plotly)
        ├── Lecture ISIN personnalisés ──► MySQL (VPS Hostinger)
        └── Sauvegarde ISIN modifiés ───► MySQL (VPS Hostinger)
```

---

## Interface utilisateur

### Cartouches sidebar (pattern validé v2.6.0 ⭐)

Chaque ligne de ticker utilise **3 colonnes Streamlit** `[1, 8, 1]` :

```
[🟢]  [ ASML → Acheter (NL0000285116)  ]  [🗑️]
 ↑           ↑ bouton centré               ↑
 boule      (signal + ISIN en texte)     poubelle
 markdown
```

- La **boule** est un `st.markdown` HTML libre (permet le CSS)
- Le **bouton** est un `st.button` standard (pas de HTML dans le label)
- La **boule est entourée** d'un cercle rouge `border: 4px solid #C62828` sur la ligne active
- Le **texte du bouton** affiche `▶▶▶ nom → signal (ISIN) ◄◄◄` sur la ligne active

### CSS global appliqué

```css
/* Alignement gauche du contenu des boutons */
[data-testid="stSidebarContent"] .stButton > button {
    display: flex;
    justify-content: flex-start;
    align-items: center;
}
/* Réduction espacement entre colonnes */
[data-testid="stSidebarContent"] [data-testid="stHorizontalBlock"] {
    gap: 4px;
}
```

---

## Indicateurs techniques

| Indicateur | Période | Signal généré |
|------------|---------|---------------|
| MA50 / MA200 | 50 / 200 jours | Acheter si MA50 > MA200 |
| MACD | 12 / 26 / 9 | Acheter si MACD > Signal |
| RSI | 14 jours | Survente < 30, Surachat > 70 |
| Bandes de Bollinger | 20 jours, σ=2 | Cassure haute/basse |
| Golden Cross | MA50 > MA200 | Signal haussier fort |
| Death Cross | MA50 < MA200 | Signal baissier fort |

**Recommandation finale** : combinaison pondérée de ces indicateurs → Acheter / Attente / Vendre / Neutre.

---

## Infrastructure VPS Hostinger

### Serveur

| Élément | Valeur |
|---------|--------|
| Hébergeur | Hostinger VPS |
| IP publique | `76.13.49.53` |
| OS | Ubuntu 24.04 LTS |
| Runtime | Docker |
| **App Streamlit** | Conteneur `streamlit-bourse`, port `8501` |
| **MySQL** | Conteneur `mysql-bourse` (image `mysql:8.0`), port `3306` |
| URL publique | http://76.13.49.53:8501 |
| Deploy script | `/opt/streamlit-bourse/deploy.sh` |

### Base de données MySQL

```
Base    : bourse_isin
User    : bourse_user
Tables  : isin_utilisateurs, sessions, utilisateurs
```

**Nouveautés v3.0.x** :
- Table `sessions` : persistance login 30 jours via token UUID
- Table `utilisateurs` : authentification bcrypt (admin/user)
- SQLAlchemy QueuePool (pool_size=5, max_overflow=10)
- Secrets via `.env` sur VPS (python-dotenv)

### Schéma de la table

```sql
CREATE TABLE isin_utilisateurs (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    utilisateur VARCHAR(50)  NOT NULL,
    ticker      VARCHAR(20)  NOT NULL,
    isin        VARCHAR(14)  NOT NULL,
    categorie   VARCHAR(20)  NOT NULL DEFAULT 'PEA',
    nom         VARCHAR(100) NOT NULL DEFAULT '',
    emoji       VARCHAR(10)  NOT NULL DEFAULT '📈',
    date_modif  TIMESTAMP    DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY unique_user_ticker (utilisateur, ticker)
);
```

> **Note :** `VARCHAR(14)` car certains ISIN (ex: `FR001400AYG6`) dépassent 12 caractères.
> Les colonnes `nom` et `emoji` remplacent le dict hardcodé `actions_par_utilisateur` depuis la v2.7.0.

### Fonctions Python dans le code

```python
get_connexion_mysql()                                              # Connexion via st.secrets["mysql"]
charger_isin_mysql(utilisateur: str) -> dict                       # SELECT ticker→isin
charger_tickers_mysql(utilisateur: str) -> dict                    # SELECT tickers+nom+emoji+categorie → sidebar
sauvegarder_ticker_mysql(utilisateur, ticker, isin, cat, nom, emoji)  # INSERT complet (ajout nouveau ticker)
sauvegarder_isin_mysql(utilisateur, ticker, isin, cat)             # UPDATE ISIN seul (onglet Modifier)
supprimer_isin_mysql(utilisateur: str, ticker: str)                # DELETE par utilisateur + ticker
```

### Données initiales

74 tickers/ISIN dans `isin_utilisateurs` (plus de hardcode dans le code depuis v2.7.0) :
- **Michel** : 9 PEA + 15 TITRES = 24 lignes
- **Romain** : 16 PEA + 2 TITRES = 18 lignes
- **Roger** : 7 PEA + 25 TITRES = 32 lignes

Scripts SQL de référence :
- `populate_isin.sql` — insertion initiale des ISIN
- `update_noms.sql` — mise à jour des noms et emojis

### Commandes de maintenance MySQL

```bash
# Vérifier que le conteneur tourne
docker ps | grep mysql-bourse

# Voir tous les ISIN
docker exec -i mysql-bourse mysql -u bourse_user -pBoursePass2024! bourse_isin \
  -e "SELECT utilisateur, COUNT(*) as nb FROM isin_utilisateurs GROUP BY utilisateur;"

# Voir les ISIN d'un utilisateur
docker exec -i mysql-bourse mysql -u bourse_user -pBoursePass2024! bourse_isin \
  -e "SELECT ticker, isin, categorie FROM isin_utilisateurs WHERE utilisateur='Michel' ORDER BY categorie, ticker;"

# Repeupler depuis le script SQL (si besoin de reset)
docker exec -i mysql-bourse mysql -u bourse_user -pBoursePass2024! bourse_isin < populate_isin.sql

# Redémarrer le conteneur MySQL si arrêté
docker start mysql-bourse
```

---

## Configuration VPS

Les credentials MySQL sont stockés dans **`.env`** sur le VPS (jamais dans le code source).

### Fichier `/opt/streamlit-bourse/.env`

```bash
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_DATABASE=bourse_isin
MYSQL_USER=bourse_user
MYSQL_PASSWORD=...
```

> ⚠️ **Sécurité** : `.env` est dans `.gitignore`. Un fichier `.env.example` est fourni dans le repo.

### Compatibilité Streamlit Cloud conservée

Le code lit d'abord `os.environ` (`.env` VPS), puis `st.secrets` en fallback (Streamlit Cloud). L'app reste déployable sur Streamlit Cloud si besoin.

---

## Installation et lancement local

### Prérequis

- Python 3.11+ (via Homebrew sur macOS)
- Accès MySQL (VPS Hostinger ou local)

### Installation

```bash
# Cloner le dépôt
git clone https://github.com/Michel91dev/greg-sp500-demo.git
cd greg-sp500-demo

# Créer l'environnement virtuel
python3 -m venv .venv
source .venv/bin/activate

# Installer les dépendances
pip install -r requirements.txt
```

### Configuration MySQL en local

Créer le fichier `.streamlit/secrets.toml` (non versionné) :

```toml
[mysql]
host = "76.13.49.53"
port = 3306
database = "bourse_isin"
user = "bourse_user"
password = "..."
```

### Lancement

```bash
streamlit run streamlit_sp500_demo.py
```

L'application s'ouvre à http://localhost:8501

---

## Déploiement VPS

- **Plateforme** : VPS Hostinger (Docker)
- **Branche déployée** : `main`
- **Deploy manuel** : SSH + script `/opt/streamlit-bourse/deploy.sh`

### Workflow de déploiement

```bash
# 1. Développer en local
git add -A && git commit -m "Description du changement"
git push origin main

# 2. Déployer sur VPS (depuis local)
ssh root@76.13.49.53
cd /opt/streamlit-bourse
./deploy.sh
```

### Script `deploy.sh`

```bash
#!/bin/bash
cd /opt/streamlit-bourse
git pull origin main
docker stop streamlit-bourse
docker rm streamlit-bourse
docker build -t streamlit-bourse .
docker run -d --name streamlit-bourse --env-file .env -p 8501:8501 streamlit-bourse
date
```

### Déploiement automatique (TODO)

Webhook GitHub → `deploy.sh` auto à chaque push sur `main`

---

## Gestion des versions

Le fichier `version.txt` contient le numéro de version courant, affiché dans la sidebar de l'app.

| Version | Description |
|---------|-------------|
| 2.9.2 | Dernière version Streamlit Cloud |
| **3.0.0** | **Migration VPS** — SQLAlchemy pool + .env secrets |
| 3.0.1 | Sessions MySQL persistantes 30 jours (token UUID) |
| 3.0.2 | Sidebar : fond coloré ticker sélectionné (CSS) |
| 3.0.3 | Fix CSS alignement boutons |
| 3.0.4 | Fix encodage UTF-8 (Émergents) |
| 3.0.5 | Sidebar HTML pur (test) |
| 3.0.6 | JS DOM pour fond coloré ticker sélectionné |
| 3.0.7 | Fix encodage UTF-8 connect_args + CSS poubelle 36px |
| **3.0.8** | **Mode personnalisé déplacé avant ADMINISTRATION** |

---

## Dépendances

```
streamlit>=1.29.0       # Framework web interactif
yfinance>=0.2.28        # Données boursières Yahoo Finance
plotly>=5.17.0          # Graphiques interactifs
pandas>=2.2.0           # Manipulation et analyse de données
pymysql>=1.1.0          # Driver MySQL
sqlalchemy>=2.0.0       # ORM + connection pooling
python-dotenv>=1.0.0    # Gestion .env secrets
bcrypt>=4.0.0           # Hash mots de passe
```

---

## Maintenance et opérations

### Ajouter un nouveau ticker pour un utilisateur

1. Ajouter le ticker dans `actions_par_utilisateur` dans `streamlit_sp500_demo.py`
2. Ajouter l'ISIN dans le dict `isin_actions` (valeur par défaut)
3. Mettre à jour `populate_isin.sql` avec la nouvelle ligne
4. Exécuter l'INSERT sur le VPS ou via l'interface "🔑 Gérer les ISIN" dans l'app
5. Incrémenter `version.txt` et pousser sur `main`

### Vérifier la santé de l'infrastructure

```bash
# Sur le VPS Hostinger (terminal web hPanel ou SSH)
docker ps                          # Vérifier que mysql-bourse tourne
docker logs mysql-bourse --tail=20 # Voir les derniers logs MySQL
ufw status                         # Vérifier que le port 3306 est ouvert
```

### Redémarrage d'urgence MySQL

```bash
docker restart mysql-bourse
# Attendre 10 secondes puis vérifier
docker exec -i mysql-bourse mysql -u bourse_user -pBoursePass2024! bourse_isin -e "SHOW TABLES;"
```

### Sauvegarde des données

```bash
# Export complet de la table ISIN
docker exec mysql-bourse mysqldump -u bourse_user -pBoursePass2024! bourse_isin isin_utilisateurs > backup_isin_$(date +%Y%m%d).sql
```
