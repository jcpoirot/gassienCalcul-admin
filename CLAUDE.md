# Gassien Paris – Admin Workspace

## Vue d'ensemble générale

Workspace pour le back office Gassien Paris, comprenant API backend et interface frontend React. Système de gestion complet pour commandes B2B, devis, factures, stocks et intégrations tierces.

## Architecture globale

```
admin/
├── gassienCalcul/          # API Node.js/Express (Backend)
├── gassienCalculFront/     # React 18 App (Frontend)
├── gassienInitBdd/         # Scripts init DB (secondaire)
├── gassienProxy/           # Config Nginx proxy (secondaire)
├── makerServerless/        # Fonctions serverless (secondaire)
└── stocks/                 # Gestion stocks (secondaire)
```

## Repos principaux

### gassienCalcul (Backend API)
**Type** : Node.js/Express API
**Rôle** : API REST back office métier
**Deploy** : PM2 sur AWS Lightsail Ubuntu 24.04
**Port** : 8080 (Nginx reverse proxy depuis 80)
**Doc** : Voir `gassienCalcul/CLAUDE.md`

### gassienCalculFront (Frontend)
**Type** : React 18 SPA
**Rôle** : Interface utilisateur back office
**Build** : Webpack 4, bundle vers backend
**Dev port** : 8083 (HMR)
**Deploy** : Bundle copié dans `gassienCalcul/public/js/`
**Doc** : Voir `gassienCalculFront/CLAUDE.md`

## Stack technique global

### Backend (gassienCalcul)
- **Runtime** : Node.js 24 LTS (nvm)
- **Framework** : Express 4.16.2
- **Process Manager** : PM2
- **Database** : AWS DynamoDB
- **Storage** : AWS S3
- **Auth** : AWS Cognito
- **PDF** : pdfmake
- **Email** : Brevo (API transactionnelle)

### Frontend (gassienCalculFront)
- **Framework** : React 18.2.0
- **Router** : React Router v5.1.2
- **UI Library** : Ant Design 5.14.0
- **Build** : Webpack 4 + Babel
- **State** : useReducer + Context API
- **Auth** : AWS Amplify (Cognito)

## Infrastructure AWS

### Services utilisés
- **Lightsail** : Instance Ubuntu 24.04 (52.47.115.90)
- **Cognito** : Auth users (pool: eu-west-1_mX9ktt0u2) **NE PAS MODIFIER**
- **DynamoDB** : Tables STOCKS, QUOTATION, INVOICE, MAKER
- **S3** : Buckets gassien-cmd-config, gassien-invoices-lock
- **Load Balancer** : Terminaison HTTPS

### Régions
- **eu-west-1** (Ireland) : DynamoDB, Cognito, S3
- **eu-west-3** (Paris) : Lightsail instance

## Flow d'authentification

```
User (Browser)
    ↓
Frontend React (Cognito Amplify UI)
    ↓
AWS Cognito (User Pool: eu-west-1_mX9ktt0u2)
    ↓
JWT Access Token
    ↓
Frontend → API (Header: accesstoken)
    ↓
Backend (cognito-express middleware)
    ↓
DynamoDB / S3 / Business Logic
```

**Particularités** :
- Token refresh auto (60s avant expiration)
- Middleware Cognito sur toutes routes `/api/*`
- Config Cognito **NE PAS MODIFIER** (instruction projet)

## Flow de déploiement

### Frontend → Backend → Serveur

```bash
# 1. Build frontend
cd gassienCalculFront
npm run build                    # → build/main.bundle.js (2.26 MB)

# 2. Deploy local (copie vers backend)
npm run deploy-local             # → ../gassienCalcul/public/js/

# 3. Deploy backend
cd ../gassienCalcul
npm run deploy                   # → Git push origin + lightsail

# 4. Sur serveur (post-receive hook)
cd /var/www/gassien
npm install --omit=dev
pm2 reload gassien-bo            # Zero-downtime reload
```

### Git bare repository
**Remote** : `ubuntu@52.47.115.90:/var/repo/gassien.git`
**Checkout** : `/var/www/gassien`

### PM2 Configuration
```javascript
// ecosystem.config.js
{
  name: 'gassien-bo',
  script: './src/index.js',
  env_production: {
    NODE_ENV: 'prod',
    AWS_PROFILE: 'dynamo'
  }
}
```

### Nginx Reverse Proxy
```
External (HTTPS) → Load Balancer
                 ↓
             Port 80 → Nginx
                     ↓
                 Port 8080 → Express App
```

## Flow de données

### Commande (Order Flow)

```
Frontend React
    ↓ (POST /api/createQuotation)
Backend API
    ↓
DynamoDB QUOTATION table
    ↓
S3 (ID auto-increment via ids.json)
    ↓ (Generate PDF)
pdfmake
    ↓
S3 (YYYY/MM/quotationId.pdf)
```

### Facture (Invoice Flow)

```
Frontend → POST /api/generateInvoice
         → Backend invoice.js
         → DynamoDB INVOICE table
         → S3 ID increment
         → pdfmake PDF generation
         → S3 upload (YYYY/MM/invoiceId.pdf)
         → Response with PDF URL
```

### Stocks (Stock Flow)

```
Cron (hourly) → syncStocksFromFile.js
              ↓
          Google Drive (CSV file)
              ↓
          Parse & transform
              ↓
          DynamoDB STOCKS table
              ↓
          Frontend → GET /api/stocks/getStocks
              ↓
          Display in CurrentStocks component
```

### WooCommerce Integration

```
WooCommerce Site (WordPress)
    ↓ (REST API)
Backend wcapi.js → GET orders
    ↓
Transform to internal format
    ↓
Frontend WebOrders component
    ↓ (User action)
POST /api/web/order → Create quotation
    ↓
Normal order flow
```

## Base de données

### DynamoDB Tables

#### STOCKS
- **PK** : sku (S), version (S)
- **GSI** : VERSION
- **Attributes** : availableStock, alertStock, description

#### QUOTATION
- **PK** : idQuotation (S)
- **Attributes** : Customer, line items, totals, status, dates
- **Indexes** : Status, date ranges

#### INVOICE
- **PK** : idInvoice (S)
- **Attributes** : Invoice data, linked to quotations
- **Indexes** : invoiceStatus, date ranges

#### MAKER
- **PK** : id (S), version (S)
- **GSI** : LASTUPDATE
- **Attributes** : Maker schemas with versioning

### S3 Structure

#### Bucket: gassien-cmd-config
```
ids.json              # Auto-increment counters
ids_dev.json          # Dev environment
tags.json             # Tag management
tags_dev.json         # Dev tags
```

#### Bucket: gassien-invoices-lock (prod)
```
YYYY/
  MM/
    invoiceId.pdf
    quotationId.pdf
```

## Tâches cron (serveur)

**Wrapper** : `/home/ubuntu/run-cron.sh` (charge nvm)

```bash
# Stocks sync
00 * * * * /home/ubuntu/run-cron.sh syncStocksFromFile

# Transports sync
05 * * * * /home/ubuntu/run-cron.sh syncTranportsFromFile

# Transport status
10 * * * * /home/ubuntu/run-cron.sh syncTranportsStatusFromFile

# Backup S3
30 * * * * /home/ubuntu/backup_gassien/s3-backup.sh
```

## Intégrations tierces

### 1. Egetra (Entrepôt)
**Type** : API XML
**Auth** : Session management
**Features** :
- Récupération stocks
- Statuts commandes
- Suivi transports

**Fichiers** :
- `gassienCalcul/src/modules/warehouse/egetra.js`
- `gassienCalcul/credentials/egetra.js`

### 2. WooCommerce
**Type** : REST API WordPress
**Features** :
- Récupération commandes web
- Transformation en devis
- Mise à jour statuts paiement

**Fichiers** :
- `gassienCalcul/src/modules/web/orders.js`
- `gassienCalcul/src/modules/web/wcapi.js`
- `gassienCalcul/credentials/woocommerce.js`

### 3. Google Drive
**Type** : Service Account API
**Features** :
- Backup fichiers
- Sync CSV stocks/transports

**Fichiers** :
- `gassienCalcul/src/modules/google/driveUtils.js`
- `gassienCalcul/credentials/google.json`

### 4. Brevo (Email transactionnel)
**Type** : API REST transactionnelle (SDK `@getbrevo/brevo` v5)
**Features** :
- Envoi emails via templates (variables Brevo `{{ params.xxx }}` en New Template Language)
- Envoi plain text/HTML sans template (subject + sender + textContent/htmlContent)
- Pièces jointes en base64 (PDFs factures/devis, CSV…)
- Module générique réutilisable

**Flow 1 : envoi facture par email client (manuel)** :
```
Frontend /invoices ou /order/:id → click SendOutlined → SendByEmailModal
    ↓
POST /api/sendInvoiceByEmail { idInvoice, recipients[], language }
    ↓
sendInvoiceMail.sendInvoiceByEmail()
    ↓
invoice.getInvoicePdf() → { pdfName, pdfBase64 }    (via pdfmake)
    ↓
mail.sendTransactionalEmail() (mode template, langue FR/EN)
    ↓
Email envoyé avec PDF en PJ
```

**Flow 2 : envoi facture vers PennyLane (auto + retry)** :
```
A. Création :
   POST /api/generateInvoiceFromOrder
        ↓
   invoice.createInvoiceFromOrder()
        ↓ (puis, synchroniquement)
   sendInvoiceToAccounting.sendInvoiceToAccounting()

B. Retry depuis le front (icône cartouche rouge ou défaut) :
   POST /api/sendInvoiceToAccounting { idInvoice }
        ↓
   sendInvoiceToAccounting.sendInvoiceToAccounting()

Dans les 2 cas :
   getInvoiceById() → invoiceDate
        ↓
   getInvoicePdf(idInvoice, 'fr') → PDF FR
        ↓
   mail.sendTransactionalEmail() (mode plain text, sans template,
     subject "Facture Vente Gassien <id>", textContent, PJ PDF FR)
        ↓
   Destinataire : ACCOUNTING_EMAIL (env-aware — prod = PennyLane, dev = jean@gassien.com)
        ↓
   updateInvoiceAccountingStatus() — persiste sentDate + status + error sur l'invoice
```

**Attributs persistés sur invoice (DynamoDB)** :
- `accountingSoftwareSentDate` (ISO)
- `accountingSoftwareSentStatus` (boolean — true=OK, false=erreur, absent=jamais)
- `accountingSoftwareSentError` (string|null)

**Configuration templates + compta** (versionnée) :
- `gassienCalcul/src/static/mailTemplates.js` :
  - `BREVO_TEMPLATES.invoice.{fr,en}` — IDs templates pour envoi client
  - `ACCOUNTING_EMAIL` — env-aware (`NODE_ENV === 'prod'` → PennyLane)
  - `ACCOUNTING_SENDER` — sender Brevo vérifié pour les envois plain text
- Modification → `pm2 reload gassien-bo` obligatoire

**⚠️ Sécurité compta** : la bascule env-aware sur `ACCOUNTING_EMAIL` garantit qu'on n'envoie JAMAIS de fausses factures de l'env de dev sur la prod PennyLane. Critère unique : `process.env.NODE_ENV === 'prod'`.

**Fichiers** :
- `gassienCalcul/src/modules/mail.js` – Module générique d'envoi (template OU plain text)
- `gassienCalcul/src/modules/sendInvoiceMail.js` – Envoi facture client
- `gassienCalcul/src/modules/sendInvoiceToAccounting.js` – Envoi facture compta + update DB
- `gassienCalcul/src/static/mailTemplates.js` – Config IDs templates + ACCOUNTING_EMAIL
- `gassienCalcul/src/routes/mail.js` – Route de test dev-only
- `gassienCalcul/credentials/brevo.js` – BREVO_API_KEY (NON versionné)
- `gassienCalculFront/src/components/shared/SendByEmailModal.js` – Modal envoi client
- `gassienCalculFront/src/components/shared/AccountingStatusBadge.js` – Badge statut compta (3 états)

### 5. Gemini AI
**Type** : Google AI API
**Features** :
- Génération images produits

**Fichiers** :
- `gassienCalcul/src/modules/ai/imageGeneration.js`
- `gassienCalcul/credentials/gemini.json`

## Données métier (CSV)

**Location** : `gassienCalcul/referential/`

### Fichiers référentiels (versionnés par date)

Les fichiers CSV sont **versionnés avec préfixe YYYYMMDD** et évoluent fréquemment :

**Fichiers actuels** :
- `20251130_compo.csv` – Compositions produits
- `20250829_referential.csv` – Référentiel produits
- `20251130_name_sku_v2.csv` – SKU → Name mapping
- `20251130_prices_v2.csv` – Grilles tarifaires

**Fichiers statiques** :
- `materials.js` – Définitions matériaux
- `isocode3.js` – Codes pays

### Mise à jour référentiels

**Process** :
1. Ajouter nouveau CSV daté : `YYYYMMDD_[nom].csv`
2. Modifier `gassienCalcul/src/static/urls.js`
3. Commit + deploy backend
4. **PM2 reload obligatoire** (fichiers chargés au boot)

**Config** : `gassienCalcul/src/static/urls.js`
```javascript
exports.COMPO_SRC = './referential/20251130_compo.csv'
exports.REFERENTIAL_SRC = './referential/20250829_referential.csv'
exports.NAME_SKU_SRC = './referential/20251130_name_sku_v2.csv'
exports.PRICES_SRC = './referential/20251130_prices_v2.csv'
```

**Chargement** : Au démarrage dans `src/modules/composition.js` (streams CSV)

## Conventions de code

### Backend (gassienCalcul)
- Routes dans `/src/routes`
- Logique métier dans `/src/modules`
- Utilitaires dans `/src/tools`
- Credentials dans `/credentials` (NON versionné)
- CSV data dans `/referential` (versionnés YYYYMMDD)
  - Fichiers référentiels : `YYYYMMDD_[nom].csv`
  - Config chemins : `src/static/urls.js`
  - Reload PM2 après modification

### Frontend (gassienCalculFront)
- Composants dans `/src/components` (par feature)
- Business logic dans `/src/engine`
- API calls dans `/src/hooks`
- State management dans `/src/hooksReducers`
- Config dans `/src/static`

### Git
- `.gitignore` inclut `/credentials`
- Frontend bundle **est versionné** dans backend
- Deploy via bare repo post-receive hook

## Variables d'environnement

### Backend
```bash
# Dev
NODE_ENV='dev'
AWS_PROFILE='default'      # DynamoDB local

# Prod
NODE_ENV='prod'
AWS_PROFILE='dynamo'       # DynamoDB AWS
```

### Frontend
Build config dans Webpack (pas de .env)
- Dev : Port 8083
- Prod : Bundle vers backend

## Credentials (NON versionnés)

**Location** : `gassienCalcul/credentials/`

### Fichiers requis
- `google.json` – Service account Google
- `brevo.js` – BREVO_API_KEY (envoi email transactionnel)
- `s3ForDev.js` – AWS S3 dev credentials
- `woocommerce.js` – WooCommerce API
- `egetra.js` – Egetra warehouse API
- `laruche.js` – La Ruche client config
- `gemini.json` – Gemini AI credentials

**⚠️ Important** : tous les credentials doivent exister sur la prod (`/var/www/gassien/credentials/` sur 52.47.115.90). Toute mise à jour d'un module qui consomme un nouveau credential doit être précédée du scp du fichier sur le serveur, sinon le boot PM2 plante.

**AWS Credentials** : `~/.aws/credentials`
- `[default]` – Dev
- `[dynamo]` – Production
- `[jcpoirot_amplify]` – Amplify

## Commandes essentielles

### Dev local complet

```bash
# Terminal 1: Backend
cd gassienCalcul
export NODE_ENV=dev
export AWS_PROFILE=default
npm run local              # Port 8080, nodemon

# Terminal 2: Frontend
cd gassienCalculFront
npm start                  # Port 8083, HMR

# Browser: http://localhost:8083
```

### Build & Deploy complet

```bash
# 1. Build frontend
cd gassienCalculFront
npm run build

# 2. Deploy vers backend
npm run deploy-local

# 3. Deploy backend vers serveur
cd ../gassienCalcul
npm run deploy

# OU tout en une fois depuis frontend
cd gassienCalculFront
npm run deploy            # Build + copy + git push
```

### Mise à jour référentiels CSV

```bash
# 1. Ajouter nouveau fichier daté dans gassienCalcul/referential/
cd gassienCalcul/referential
# Exemple: 20260115_compo.csv

# 2. Modifier la config pour pointer vers nouveaux fichiers
nano src/static/urls.js
# Modifier COMPO_SRC, REFERENTIAL_SRC, NAME_SKU_SRC, PRICES_SRC

# 3. Deploy backend
cd ..
npm run deploy

# 4. IMPORTANT: Reload PM2 pour recharger les CSV
ssh ubuntu@52.47.115.90
pm2 reload gassien-bo
pm2 logs gassien-bo | grep -i "compo\|referential"
```

### Serveur (SSH)

```bash
ssh ubuntu@52.47.115.90

# PM2 commands
pm2 list
pm2 logs gassien-bo
pm2 reload gassien-bo
pm2 monit

# Nginx
sudo systemctl status nginx
sudo systemctl reload nginx

# Logs
tail -f /var/log/nginx/access.log
pm2 logs --lines 100
```

## Points d'attention critiques

### 🔒 Sécurité
1. **Cognito config NE PAS MODIFIER** (instruction projet)
2. **Credentials NON versionnés** – Jamais commit
3. **JWT refresh auto** – Géré côté frontend (60s seuil)
4. **Toutes routes API = Auth** – Sauf health checks

### 🚀 Déploiement
1. **Frontend bundle dans backend** – Déployer frontend d'abord
2. **PM2 reload après deploy** – Zero-downtime
3. **Tester PM2 après modif backend** (instruction projet)
4. **Crons nécessitent wrapper nvm** – Gestion version Node

### 📦 Dependencies
1. **Webpack 4 (legacy)** – Migration future possible
2. **Node 24 LTS via nvm** – Flags OpenSSL legacy
3. **Ant Design 5** – Breaking changes depuis v4
4. **React Router v5** – v6 migration future

### 🗄️ Données
1. **IDs auto-incrémentés via S3** – Pas de DB sequences
2. **PDFs stockés S3** – Structure YYYY/MM/
3. **CSV référentiels versionnés par date** – Préfixe YYYYMMDD, update dans `src/static/urls.js` + PM2 reload
4. **DynamoDB update pattern** – Delete + recreate

## Monitoring & Logs

### Serveur
```bash
# PM2 monitoring
pm2 monit

# Logs temps réel
pm2 logs gassien-bo --lines 50

# Nginx logs
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

### AWS CloudWatch
- DynamoDB metrics (read/write capacity)
- S3 access logs
- Load Balancer health checks

## Troubleshooting

### Frontend ne charge pas
1. Vérifier bundle dans `gassienCalcul/public/js/main.bundle.js`
2. Console browser pour erreurs JS
3. Vérifier Cognito credentials (userPool.js)

### API 401 Unauthorized
1. Token expiré → Refresh browser
2. Cognito config incorrecte
3. Header `accesstoken` manquant

### PM2 app crash
```bash
pm2 logs gassien-bo --err --lines 100
pm2 restart gassien-bo
```

### DynamoDB local issues
```bash
# Vérifier DynamoDB local running
ps aux | grep dynamodb
# Start si nécessaire
java -Djava.library.path=./DynamoDBLocal_lib -jar DynamoDBLocal.jar -sharedDb
```

### Stocks pas à jour
```bash
# Vérifier cron running
crontab -l
# Run manual sync
cd /var/www/gassien
npm run syncStocksFromFile
```

### Prix/SKU/Compositions incorrects
**Cause** : Fichiers CSV référentiels pas chargés
```bash
# Vérifier fichiers dans gassienCalcul/referential/
ls -la gassienCalcul/referential/*.csv

# Vérifier config
cat gassienCalcul/src/static/urls.js

# Reload PM2 pour recharger CSV
pm2 reload gassien-bo
pm2 logs gassien-bo | grep -i "csv\|compo\|referential"
```

## Documentation détaillée

- **Backend API** : `gassienCalcul/CLAUDE.md`
- **Frontend** : `gassienCalculFront/CLAUDE.md`
- **Readme backend** : `gassienCalcul/readme.md`

## Liens utiles

- **Serveur** : 52.47.115.90
- **Pool Cognito** : eu-west-1_mX9ktt0u2
- **S3 Config** : gassien-cmd-config
- **S3 Invoices** : gassien-invoices-lock
- **Region DynamoDB** : eu-west-1
- **Region Lightsail** : eu-west-3

## Contact & Support

Pour questions/issues spécifiques :
- Backend : Voir `gassienCalcul/CLAUDE.md`
- Frontend : Voir `gassienCalculFront/CLAUDE.md`
- Infrastructure : AWS Console (eu-west-1/3)
