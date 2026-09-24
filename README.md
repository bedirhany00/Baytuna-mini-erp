<div align="center">

# 🏭 Baytuna Mini ERP

**A small two-service ERP system that brings customers, orders, stock, and invoicing under one roof.**

![Next.js](https://img.shields.io/badge/Next.js-16-000000?logo=nextdotjs&logoColor=white)
![React](https://img.shields.io/badge/React-19-61DAFB?logo=react&logoColor=black)
![FastAPI](https://img.shields.io/badge/FastAPI-Python%203.12-009688?logo=fastapi&logoColor=white)
![.NET](https://img.shields.io/badge/ASP.NET%20Core-.NET%2010-512BD4?logo=dotnet&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16%20%7C%2018-4169E1?logo=postgresql&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)

</div>

Developed as a two-person internship project at Baytuna Grup A.Ş. The system consists of **two independently deployable backend services** and a **Next.js frontend** that talks to both. Each service owns its own database; they communicate over HTTP and share a single JWT authentication scheme.

---

## ✨ Features

- 🔐 **Role-based authorization:** `admin`, `sales`, and `warehouse` roles, enforced in both the backend and the UI
- 📦 **Product catalog:** an admin sets the profit margin and the system computes the sale price (`average cost × (1 + margin / 100)`)
- 🏬 **Stock management:** every goods receipt updates the weighted average cost; stock movements are append-only and never deleted
- 🧾 **Order flow:** order → stock reservation → confirmed/rejected → automatic invoice and customer email
- 📄 **Invoice PDFs:** generated with QuestPDF and stored on a persistent Docker volume
- ⚠️ **Critical stock alerts:** when stock drops below 10 units, warehouse staff are notified by email
- 🤖 **AI summary and Q&A:** weekly revenue commentary and natural-language questions about your data, powered by Google Gemini *(optional)*
- 👥 **Staff management:** auto-generated email/password, password reset, soft delete
- 📊 **Dashboard:** summary cards, revenue chart, critical stock list, recent orders

---

## 🏗️ Architecture

```
                     ┌──────────────────────┐
                     │  Frontend (Next.js)  │
                     └───────────┬──────────┘
                                 │  HTTPS + JWT
              ┌──────────────────┴──────────────────┐
              │                                     │
      ┌───────▼────────┐    HTTP (JWT forwarded)   ┌─▼──────────────┐
      │   Service A     │◄─────────────────────────►│   Service B     │
      │ Python/FastAPI  │   stock reservation,      │ C#/ASP.NET Core │
      │     :8000       │   product data, reports   │      :8080      │
      └───────┬─────────┘                           └────────┬────────┘
              │                                              │
      ┌───────▼────────┐                            ┌────────▼───────┐
      │  postgres-a     │                            │   postgres-b    │
      │ (service_a_db)  │                            │  (MiniErpDb)    │
      └─────────────────┘                            └────────────────┘
```

| Component | Technology | Responsibility |
|---|---|---|
| **Service A** | Python 3.12, FastAPI, SQLAlchemy, Alembic | Authentication and JWT issuance, user/staff management, product catalog, stock movements and pricing, internal stock reservation, reports and AI summary |
| **Service B** | C#, ASP.NET Core (.NET 10), EF Core | Customers, order processing and state machine, invoice records, invoice PDFs, order confirmation email |
| **Frontend** | Next.js 16, React 19, Tailwind 4, shadcn/ui | Web UI consuming both services |

> **Note:** Only Service A issues JWTs. Service B never issues tokens, it only validates them, so `JWT_SECRET`, `JWT_ISSUER`, and `JWT_AUDIENCE` must be **identical** in both services.

---

## 🔄 Order Lifecycle

```
                ┌─ enough stock ──────► confirmed ──► invoice + PDF + email
pending ────────┤
                └─ insufficient stock ► rejected (rejection_reason is set)
```

1. A sales user creates an order. Service B fetches product prices from Service A and stores them as **snapshots**, so past invoices stay correct even if a product's name or price changes later.
2. The order is saved as `pending`, and a reservation request is sent to Service A's `/internal/stock/reserve` endpoint.
3. Service A locks the affected product rows (always in `id` order to avoid deadlocks). **If every item is available**, stock is deducted; **if even one is short, nothing is deducted.**
4. Depending on the result, the order becomes `confirmed` or `rejected`. A confirmed order gets an invoice, a PDF is generated, and the customer receives it by email.

**Safety properties**

- **Idempotent reservation:** if the same `reservationId` arrives twice, stock is not deducted again.
- **If Service A is unreachable**, the order stays `pending` (a `503` is returned) instead of being wrongly confirmed or rejected.
- If sending the email fails, the order flow is not affected; the error is logged.

---

## 👤 Roles and Permissions

| Capability | Admin | Sales | Warehouse |
|---|:---:|:---:|:---:|
| Create / edit / delete products | ✅ | | |
| List products | ✅ | ✅ | ✅ |
| Stock entry and movement history | | | ✅ |
| Add and list customers | ✅ | ✅ | |
| Create orders | | ✅ | |
| List orders / invoice PDF | ✅ | ✅ | ✅ |
| Staff management | ✅ | | |
| Dashboard, AI summary and Q&A | ✅ | ✅ | ✅ |

---

## 🚀 Getting Started

### Prerequisites

- [Docker](https://www.docker.com/) and Docker Compose
- [Node.js](https://nodejs.org/) 22 (for the frontend)
- Git

### 1. Clone the repository

```bash
git clone https://github.com/bedirhany00/Baytuna-mini-erp.git
cd Baytuna-mini-erp
```

### 2. Configure environment variables

```bash
cp .env.example .env
```

Then edit `.env`. At a minimum, change the following:

| Variable | Description |
|---|---|
| `POSTGRES_B_PASSWORD` | Service B database password |
| `JWT_SECRET` | Shared secret for both services. Generate one with: `python -c "import secrets; print(secrets.token_hex(32))"` |
| `JWT_ISSUER` / `JWT_AUDIENCE` | Defaults: `MiniErp.ServiceA` / `MiniErp.ServiceB` |
| `CORS_ORIGINS` | Frontend origin(s), comma-separated. Locally: `http://localhost:3000` |
| `GITHUB_OWNER` | Used for the GHCR image name in `docker-compose.yml` |

Optional variables:

| Variable | Purpose | If left empty |
|---|---|---|
| `RESEND_API_KEY`, `RESEND_FROM` | Service A critical stock email (Resend is tried first) | Falls back to SMTP |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`, `SMTP_FROM` | Service B order confirmation email; fallback for Service A | No email is sent; the flow is not affected |
| `GEMINI_API_KEY`, `GEMINI_MODEL` | AI summary and Q&A ([get a key](https://aistudio.google.com/apikey)) | Feature stays disabled; the numeric dashboard keeps working |

### 3. Start the backend services

```bash
docker compose up -d
```

Service A's tables are not created automatically. Run the migrations and load the demo data (Service B applies its own migrations on startup):

```bash
docker compose exec service-a alembic upgrade head
docker compose exec service-a python seed.py
```

Health checks:

```bash
curl http://localhost:8000/health   # Service A
curl http://localhost:8080/health   # Service B
```

### 4. Run the frontend

```bash
cd frontend
npm install
cp .env.local.example .env.local
npm run dev
```

Open **http://localhost:3000** in your browser.

### 🔑 Demo Accounts

Available after running `seed.py`, which also loads 20 sample products. The last three products intentionally have stock below the critical threshold so the alert email can be triggered in a demo:

| Role | Email | Password |
|---|---|---|
| Admin | `admin@baytuna.com` | `admin123` |
| Sales | `satis@baytuna.com` | `satis123` |
| Warehouse | `depo@baytuna.com` | `depo123` |

> ⚠️ These accounts are for local development only. Change or remove them in any live environment.

---

## 📚 API Documentation

| Service | URL | Tool |
|---|---|---|
| Service A | http://localhost:8000/docs | Swagger UI (FastAPI) |
| Service B | http://localhost:8080/swagger | Swashbuckle |

<details>
<summary><b>Service A endpoint summary</b></summary>

| Method | Path | Access |
|---|---|---|
| `POST` | `/auth/login` | Public |
| `POST` | `/auth/register` | Admin |
| `GET` | `/products`, `/products/{id}` | Any authenticated user |
| `POST` `PUT` `DELETE` | `/products`, `/products/{id}` | Admin |
| `POST` `GET` | `/stock-movements` | Warehouse |
| `POST` | `/internal/stock/reserve` | Sales (called by Service B) |
| `GET` | `/reports/daily-summary` | Any authenticated user |
| `POST` | `/reports/ask` | Any authenticated user |
| `GET` `POST` | `/staff` | Admin |
| `DELETE` | `/staff/{id}` | Admin |
| `POST` | `/staff/{id}/reset-password` | Admin |
| `GET` | `/health` | Public |

</details>

<details>
<summary><b>Service B endpoint summary</b></summary>

| Method | Path | Access |
|---|---|---|
| `GET` `POST` | `/api/customers` | Sales, Admin |
| `POST` | `/api/orders` | Sales |
| `GET` | `/api/orders`, `/api/orders/{id}` | Any authenticated user |
| `GET` | `/api/invoices/{id}/pdf` | Any authenticated user |

</details>

---

## 🗂️ Project Structure

```
Baytuna-mini-erp/
├── service-a/                 # Python / FastAPI
│   ├── main.py                #   endpoints
│   ├── auth.py                #   JWT and role checks
│   ├── models.py              #   SQLAlchemy models
│   ├── reports.py, ai.py      #   dashboard summary and Gemini integration
│   ├── mailer.py              #   critical stock email (Resend / SMTP)
│   ├── staff.py               #   staff email/password generation
│   ├── seed.py                #   demo data
│   ├── alembic/               #   database migrations
│   └── tests/                 #   pytest tests
├── service-b/                 # C# / ASP.NET Core
│   ├── Controllers/           #   Customers, Orders, Invoices
│   ├── Clients/               #   HTTP clients for Service A
│   ├── Services/              #   PDF generation, email
│   ├── Models/, Data/         #   EF Core models and seeding
│   └── Migrations/            #   EF Core migrations
├── frontend/                  # Next.js (App Router)
│   ├── app/                   #   pages
│   ├── components/            #   app components and shadcn/ui
│   └── lib/                   #   API client, auth, types, navigation
├── contracts/database.md      # Shared data contract between services
├── docker-compose.yml
├── netlify.toml               # Frontend deploy config
└── .github/workflows/         # CI/CD
```

---

## 🧩 Data Contract

The two services are developed independently but must agree on a shared data model. That agreement lives in [`contracts/database.md`](./contracts/database.md), the single source of truth for table ownership, field names, and shared conventions:

- All timestamps are **UTC**, all ids are **UUIDs**
- Monetary fields use **`numeric`**, never `float`/`double`
- `order_items.product_id` is not a real foreign key (separate databases); integrity is enforced in the application layer
- `product_name_snapshot` and `unit_price_snapshot` are intentional; they keep past invoices correct
- Changes go through a PR and must be approved by both service owners before merging

| Database | Service | Tables |
|---|---|---|
| `service_a_db` (PostgreSQL 16) | A | `users`, `products`, `stock_movements`, `stock_reservations` |
| `MiniErpDb` (PostgreSQL 18) | B | `customers`, `orders`, `order_items`, `invoices` |

Invoice PDFs are stored on the `invoice_pdf_data` Docker volume, mounted at `/app/data/invoices` in the Service B container.

---

## 🧪 Testing

```bash
# Service A (pytest)
cd service-a
pip install -r requirements-dev.txt
pytest
```

The tests cover error scenarios, reporting, and staff management.

---

## ☁️ Deployment

| Component | Where | How |
|---|---|---|
| Service A and B | Linux VPS | On every push to `main`, GitHub Actions builds the images and pushes them to **GHCR**, then connects over SSH and updates only the relevant service on the VPS |
| Frontend | Netlify | The root `netlify.toml` contains the required settings; the `@netlify/plugin-nextjs` adapter is declared explicitly |

Set the following environment variables in the Netlify dashboard:

| Variable | Value |
|---|---|
| `NEXT_PUBLIC_SERVICE_A_URL` | Public URL of Service A |
| `NEXT_PUBLIC_SERVICE_B_URL` | Public URL of Service B |

> `NEXT_PUBLIC_*` variables are baked into the code **at build time**; after changing a value you must redeploy.
> The frontend origin must be added both to `CORS_ORIGINS` in Service A and to the `WithOrigins` list in Service B's `Program.cs`, otherwise the browser will block requests.

Required GitHub Actions secrets: `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`.

---

## 👥 Contributors

| Name | Responsibility |
|---|---|
| **Mehmet** | Service A, server and deployment configuration |
| **Bedirhan** | Service B, invoice generation, product listing and dashboard frontend pages |

---

## 📄 License

This project was developed for internship and educational purposes.
