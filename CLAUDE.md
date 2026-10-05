# CLAUDE.md — Жавоҳир савдо тизими (ERP + Shop)

Бу файл Claude Code'нинг ҳар бир сессияда ўқийдиган хотира файли. Тўлиқ лойиҳалаш режаси (архитектура, DB схема, модуллар, иш режаси) шу ерда: https://claude.ai/code/artifact/8c5d39ed-3865-4895-9e48-9b4a5dff7569 — ноаниқлик бўлса, шу ҳужжатга қайтинг.

## Лойиҳа ҳақида

Иккита дастур, умумий PostgreSQL база:
1. **erp-backend / erp-frontend** — ички учет дастури (кирим, савдо, касса, ходимлар, кредит, ҳисоботлар). **Ҳозирги фокус — шу.**
2. **shop-backend / shop-frontend** — мижозлар учун онлайн-дўкон сайти. Кейинги фаза (режадаги 5-ҳафтадан бошланади).

Иккала backend бир базага уланади, лекин PostgreSQL ROLE даражасида ажратилган (`erp_role` — тўлиқ ҳуқуқ, `shop_role` — фақат `products`, `stocks`, `categories`, `customers`, `carts`, `orders`, `payments`, `online_credit_requests`).

Келажакда дастур SaaS сифатида бошқа заргарлик дўконларига ҳам сотилади (`organizations` tenant жадвали, 21 кунлик demo trial) — бу қисм ҳозирча backlog'да, лекин DB схемасини шунга тайёр қилиб лойиҳалаш керак.

## Техник стек

- **Backend:** Go 1.22+, Gofiber v3, GORM, `golang-jwt/jwt/v5`, `go-playground/validator/v10`, `golang-migrate`, `zerolog`, `redis/go-redis/v9`
- **Frontend:** Vue 3 (Composition API, `<script setup>`), Vite, Tailwind CSS, PrimeVue v4, axios, Pinia, `vue-i18n` (uz-Cyrl асосий), ag-Grid (катта жадваллар — маҳсулот рўйхати, тарих, ҳисоботлар учун; қолган UI PrimeVue'да)
- **База:** PostgreSQL 15+, SQL миграциялар `golang-migrate` орқали версияланади (AutoMigrate эмас)

## Папка тузилиши (backend)

```
cmd/api/main.go
internal/
  config/
  database/
  models/
  modules/
    auth/ products/ incoming/ sales/ cashbox/ employees/ credits/ reports/
  middleware/      # JWT auth, RBAC, logging, CORS, rate-limit
  router/
  pkg/response/ pkg/validator/ pkg/pagination/
migrations/
```

Ҳар бир модуль уч қатламдан иборат: `handler` (HTTP) → `service` (бизнес-логика) → `repository` (GORM сўровлар). Қатламлар орасида interface ишлатилади (unit-test учун mockgen билан).

## Папка тузилиши (frontend)

```
src/
  api/            # axios instance + har modul uchun api chaqiruvlari
  stores/         # Pinia: auth, cart, ui
  composables/
  components/common/ components/layout/
  views/
  router/
  locales/
```

## DB конвенциялари

- Ҳар бир асосий жадвалда: `created_at`, `updated_at`, `deleted_at` (soft delete), `filial_id`, `created_by`
- Пул суммалари **BIGINT** (тийин/сўмда, бутун сон) — float ишлатилмайди
- `product_images` — маҳсулотнинг бир неча расми учун алоҳида жадвал (one-to-many: `product_id`, `url`, `order`, `is_primary`)
- USD/UZS курс `currency_rates` жадвалида кунлик тарих сифатида сақланади; савдо/буюртма вақтидаги курс ўша ёзувда "музлатилган" ҳолда сақланади

## RBAC

`roles` + `permissions`, формат `module:action` (мас. `sales:create`, `reports:view`). JWT payload: `user_id`, `role`, `filial_id`, `exp`. Access token 15–30 дақ, refresh token 7–30 кун (DB'да сақланади, rotatsiya билан).

## API услуби

REST + JSON, `/api/v1/...`, хато формати `{"error": {"code":..., "message":...}}`, рўйхатлар учун `page`/`limit`/`sort`/`filter` query-параметрлари, Swagger (`swaggo/swag`) орқали автоматик ҳужжатлаштириш.

## Буйруқлар

```bash
# backend
go run cmd/api/main.go
go test ./...
golangci-lint run
migrate -path migrations -database "$DATABASE_URL" up

# frontend
npm run dev
npm run build
npm run lint
```

## Ҳозирги босқич (иш режаси бўйича)

Тўлиқ кунма-кун жадвал режа ҳужжатидаги "Иш режаси (2026-10-05 — 2026-11-20)" бўлимида. Ҳар бир сессия бошида шу бўлимни текшириб, қайси кун/вазифа устида ишланаётганини аниқланг. Асосий босқичлар:

1. Инфратузилма ва auth (1-ҳафта)
2. Маҳсулот ва кирим (2-ҳафта)
3. Савдо ва касса (3-ҳафта)
4. Ходимлар, ички кредит (4-ҳафта)
5. Ҳисоботлар, RBAC, сайт бошланиши (5-ҳафта)
6. Сайт: каталог, саватча, буюртма (6-ҳафта)
7. Шахсий кабинет, интеграция, якуний синов (7-ҳафта)

Payme/Click/Uzum реал интеграцияси, SMS/Telegram хабарнома, SaaS/demo режими — 21.11–01.12 оралиғига қолдирилган, ҳозирча амалга оширилмайди.

## Қоидалар

- Ҳар бир коммит фақат битта мантиқий ўзгаришни қамраб олсин, commit message'лар инглиз/ўзбек аралаш эмас — лотин ёки инглиз тилида қисқа ва аниқ ёзилсин
- Янги жадвал/устун қўшилганда миграция файли ёзилади, AutoMigrate ишлатилмайди
- Ҳар бир модулга минимал unit-test ёзиш тавсия этилади (айниқса `service` қатлами учун)
- Режадан четга чиқадиган катта қарорлар (янги кутубхона, архитектура ўзгариши) олдин режа ҳужжатида қайд этилиши керак
