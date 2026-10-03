# FinPay — Campus Wallet

Production-oriented campus wallet implementation using **Next.js App Router + Prisma + SQLite (local) + Razorpay Test Mode**.

## Safety model
- Razorpay credentials are read only from environment variables.
- Startup/payment creation refuses keys that are not `rzp_test_...`, refuses `RAZORPAY_MODE` other than `test`, and refuses `ALLOW_RAZORPAY_LIVE=true`.
- The Razorpay secret is server-only.
- Wallet credits occur only after server-side signature verification.
- All financial amounts are integer paise.
- Merchant payments use database transactions, conditional wallet debit, and a unique request ID.
- Cashback uses `Math.floor` semantics and monthly category caps are applied by the payment service, not the raw calculator.

## Requirements
Node.js 20+ recommended.

## Setup
```bash
npm install
copy .env.example .env
# edit .env and add Razorpay TEST credentials only
npx prisma generate
npx prisma db push
npm run db:seed
npm run dev
```
Open http://localhost:3000.

Demo accounts (seeded):
- Student: `student@finpay.local` / `FinPay@12345`
- Admin: `admin@finpay.local` / `FinPay@12345`
- Merchant: `canteen@finpay.local` / `FinPay@12345`

## Razorpay Test Mode
Create a Razorpay test account/key pair and put only the test credentials into `.env`:
`RAZORPAY_KEY_ID=rzp_test_...`, `RAZORPAY_KEY_SECRET=...`, `RAZORPAY_WEBHOOK_SECRET=...`.
Do not paste live credentials into this project. The app deliberately rejects non-test key IDs.

## Tests
```bash
npm test
```
The test suite covers paise conversion and cashback flooring, tiers, categories, TOPUP exclusion, and monthly cap scaling. The API design additionally supports integration/E2E tests for authentication, Razorpay verification, duplicate callbacks/webhooks, idempotency, concurrency, and reconciliation.

## API
- `POST /api/auth/login` — authenticate and create HttpOnly session cookie
- `POST /api/auth/logout` — clear session
- `GET /api/auth/me` — current user
- `GET /api/wallet` — student wallet
- `GET /api/merchants` — active merchants for students; all merchants for admin/merchant
- `POST /api/payments` — atomic student merchant payment
- `GET /api/transactions` — student ledger
- `GET /api/cashback` — monthly cashback summary
- `POST /api/topup/order` — create Razorpay Test Mode order
- `POST /api/topup/verify` — server-side signature verification + one-time wallet credit
- `POST /api/razorpay/webhook` — signed/idempotent webhook receiver
- `GET /api/admin/stats` — admin metrics
- `GET /api/admin/reconciliation` — non-mutating reconciliation checks
- `GET /api/admin/users` — user inspection
- `PATCH /api/admin/merchants` — activate/deactivate merchant with audit log

## Database
The Prisma schema contains users, wallets, merchants, transactions, cashback records, payment references, idempotency records, and audit logs, with unique constraints and indexes for financial identifiers.

For production, migrate the datasource to PostgreSQL and run Prisma migrations; the domain logic remains paise-based and transaction-oriented.

## Project structure
```text
app/                 Next.js routes and pages
components/          UI navigation
lib/                 auth, database, money, Razorpay, cashback, payment service
prisma/              schema + seed
 tests/              unit tests
.env.example         safe configuration template
Dockerfile           container build
```

## Requirements-to-implementation audit
| Requirement | Implementation |
|---|---|
| Student registration/login/logout | `/register`, `/login`, HttpOnly JWT session, `lib/auth.ts` |
| Role authorization | `middleware.ts` plus backend `requireRole()` checks |
| Integer paise | `lib/money.ts`, Prisma `Int` fields, payment service APIs |
| Wallet consistency | Prisma transactions + conditional `balancePaise >= amountPaise` update |
| Razorpay Test Mode | `lib/config.ts`, test-key prefix guard, server-side signature verification |
| Top-up idempotency | unique Razorpay order/payment references + already-successful check |
| Merchant payment idempotency | unique transaction `requestId` and transactional lookup/create |
| Cashback rates/tiers | `lib/cashback.ts` using integer basis-point values and `Math.floor` |
| Monthly category caps | `monthlyCap()` + payment-service remaining allowance calculation |
| No TOPUP cashback | raw cashback returns 0 for `TOPUP` |
| Merchant management | admin merchant status API/UI + audit log |
| Admin activity/reconciliation | stats + non-mutating discrepancy checks |
| Transaction history | `/transactions` and `/api/transactions` |
| Merchant portal | `/merchant` with received payments |
| Error handling | consistent `{error:{code,message}}` API responses |
| Automated tests | Vitest tests in `tests/` for money/cashback edge cases |
| Live-payment prevention | non-test Razorpay keys/mode are rejected in code |
