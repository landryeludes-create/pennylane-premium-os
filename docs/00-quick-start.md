# 🚀 Guide de Démarrage Rapide - Pennylane Premium OS

## Prérequis absolus

```bash
# Node.js 20+
node --version  # v20.10.0 ou plus

# npm 10+
npm --version   # 10.0.0 ou plus

# Docker & Docker Compose
docker --version
docker-compose --version

# Git
git --version
```

## Installation (5 minutes)

### 1️⃣ Clone le repository

```bash
git clone https://github.com/landryeludes-create/pennylane-premium-os.git
cd pennylane-premium-os
```

### 2️⃣ Installation des dépendances

```bash
# Installe les dépendances root + tous les workspaces
npm install

# Vérification installation
npm run type-check
```

### 3️⃣ Configuration environnement

```bash
# Copie le fichier de config
cp .env.example .env.local

# Ouvre et édite si nécessaire (les valeurs par défaut fonctionnent)
cat .env.local
```

### 4️⃣ Démarre les services (Docker Compose)

```bash
# Démarre tous les services en arrière-plan
docker-compose up -d

# Vérification que tout est up
docker-compose ps

# Logs en temps réel
docker-compose logs -f
```

Attends 30 secondes que PostgreSQL soit prêt ⏳

### 5️⃣ Migrations base de données

```bash
# Exécute les migrations Prisma
npm run db:migrate

# Optionnel: Seed données test
npm run db:seed
```

### 6️⃣ Démarre l'application en développement

```bash
# Lance tous les services en parallèle (via Turborepo)
npm run dev
```

✅ **C'est lancé !** Ouvre ton navigateur :

```
Frontend Web  : http://localhost:3000
Admin Panel   : http://localhost:3001
API REST      : http://localhost:3000/api/v1
GraphQL       : http://localhost:3000/graphql
MailHog       : http://localhost:8025 (test emails)
Postgres      : localhost:5432
Redis         : localhost:6379
```

---

## 🎯 Premier test - Créer un compte

### Via l'interface web

```
1. Va sur http://localhost:3000
2. Clique "Register"
3. Remplis:
   Email: test@example.com
   Mot de passe: SecurePassword123!
   Entreprise: My Company
4. Clique "Create Account"
```

### Via API REST

```bash
curl -X POST http://localhost:3000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "SecurePassword123!",
    "firstName": "John",
    "lastName": "Doe",
    "company": "Acme Corp"
  }'
```

**Réponse attendue:**
```json
{
  "success": true,
  "data": {
    "id": "usr_123",
    "email": "test@example.com",
    "organization": {
      "id": "org_456",
      "name": "Acme Corp"
    }
  }
}
```

---

## 📝 Créer ta première écriture comptable

### Via GraphQL

```bash
curl -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{
    "query": "mutation { createEntry(input: { date: \"2026-05-12\", description: \"Test entry\", lines: [{ accountId: \"acc_411\", debit: 100 }, { accountId: \"acc_707\", credit: 100 }] }) { id status } }"
  }'
```

---

## 📂 Importer un fichier Excel

### Préparation du fichier

```csv
Date,Description,Amount,Account
2026-05-01,Facture client,1000,411000
2026-05-02,Salaires,-3000,641000
2026-05-03,Loyer,-1500,613000
```

### Upload via API

```bash
curl -X POST http://localhost:3000/api/v1/imports/validate \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -F "file=@transactions.csv" \
  -F "type=bank_transactions" \
  -F "dateFormat=DD/MM/YYYY"
```

---

## 🐳 Commandes Docker Compose

```bash
# Démarrer tous les services
docker-compose up -d

# Voir les logs (tout les services)
docker-compose logs -f

# Logs d'un service spécifique
docker-compose logs -f postgres
docker-compose logs -f redis

# Arrêter les services
docker-compose down

# Arrêter et supprimer volumes (nettoyage complet)
docker-compose down -v

# Redémarrer un service
docker-compose restart postgres

# Exécuter une commande dans un container
docker-compose exec postgres psql -U postgres -d pennylane_dev
```

---

## 💾 Accès à la base de données

### PostgreSQL CLI

```bash
# Depuis Docker
docker-compose exec postgres psql -U postgres -d pennylane_dev

# Ou localement (si installé)
psql -h localhost -U postgres -d pennylane_dev

# Mot de passe: password
```

### Commandes SQL utiles

```sql
-- Lister les tables
\dt

-- Voir la structure d'une table
\d journal_entries

-- Compter les entrées
SELECT COUNT(*) FROM journal_entries;

-- Voir les dernières entrées
SELECT * FROM journal_entries ORDER BY created_at DESC LIMIT 10;

-- Quitter
\q
```

---

## 🔧 Développement

### Ajouter une nouvelle dépendance

```bash
# Dans un service spécifique
cd services/accounting-service
npm install express

# Ou dans un package shared
cd packages/types
npm install axios
```

### Builder l'app

```bash
# Builder tous les services
npm run build

# Builder un service spécifique
cd services/auth-service && npm run build
```

### Tests

```bash
# Lancer tous les tests
npm run test

# Watch mode
npm run test -- --watch

# Coverage
npm run test:cov
```

---

## 🐛 Troubleshooting

### "Port 3000 already in use"

```bash
# Trouver le processus qui utilise le port
lsof -i :3000

# Tuer le processus
kill -9 <PID>

# Ou change le port dans .env.local
APP_PORT=3005
```

### "Cannot connect to PostgreSQL"

```bash
# Vérifier que Docker est démarré
docker-compose ps

# Redémarrer PostgreSQL
docker-compose restart postgres

# Attendre 10 secondes et réessayer
```

### "npm install failed"

```bash
# Vider le cache npm
npm cache clean --force

# Supprimer node_modules
rm -rf node_modules package-lock.json

# Réinstaller
npm install
```

### Vérifier la santé des services

```bash
# PostgreSQL
docker-compose exec postgres pg_isready -U postgres

# Redis
docker-compose exec redis redis-cli ping

# Elasticsearch
curl http://localhost:9200/_cluster/health
```

---

## 📊 Monitoring et Debugging

### Logs en temps réel

```bash
# Tous les services
docker-compose logs -f --tail=50

# Service spécifique
docker-compose logs -f postgres --tail=100
```

### PostgreSQL

```bash
# Voir les requêtes actives
SELECT pid, usename, query, query_start FROM pg_stat_activity;

# Taille des tables
SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) 
FROM pg_tables WHERE schemaname != 'pg_catalog' ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### Redis

```bash
# Stats Redis
docker-compose exec redis redis-cli INFO

# Monitor les commandes
docker-compose exec redis redis-cli MONITOR
```

---

## 🚀 Production Setup

Voir: [`docs/05-infrastructure.md`](./05-infrastructure.md)

---

## 📚 Prochaines étapes

1. ✅ Créer un compte utilisateur
2. ✅ Importer des données
3. ✅ Créer des écritures comptables
4. 📖 Lire la doc API: [`docs/02-api.md`](./02-api.md)
5. 🏗️ Lire l'architecture: [`docs/01-architecture.md`](./01-architecture.md)
6. 🔐 Sécurité: [`docs/04-security.md`](./04-security.md)

---

**Besoin d'aide ?** Créer une issue: https://github.com/landryeludes-create/pennylane-premium-os/issues
