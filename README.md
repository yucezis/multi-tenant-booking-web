# BeautyCenterSaaS — Beauty Center Appointment Management System

A multi-tenant SaaS appointment platform for beauty centers. When a business joins the platform, it gets its own storefront page and management panel, with full data isolation between tenants.

> Portfolio project — built to practice Clean Architecture, ASP.NET Core, SQL Server, and modern .NET patterns.

---

## Features

- **Multi-tenant architecture** — Host hundreds of businesses on a single platform with full data isolation
- **Dynamic storefront pages** — Unique URL slug per business (`/lale-guzellik`, `/gold-estetik`)
- **Smart slot calculation** — Evaluates staff availability, leave blocks, and device/room capacity together
- **Email OTP verification** — Prevents fake appointments; code valid for 15 minutes
- **Magic link cancellation** — One-click appointment cancellation without creating an account
- **Real-time notifications** — Instant updates via SignalR without page refresh
- **GDPR compliance** — Consent logging with timestamp
- **Bot protection** — Google reCAPTCHA v3 and IP-based rate limiting

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Backend | ASP.NET Core Web API / MVC (C#) |
| Architecture | Clean Architecture |
| Database | Microsoft SQL Server |
| ORM | Entity Framework Core |
| Caching | Redis |
| Real-time | SignalR |
| Background Jobs | Hangfire |
| Mail | MailKit / SMTP |
| Frontend (Admin) | ASP.NET Core MVC + Razor Pages |
| Frontend (Customer) | Razor Views + Vanilla JavaScript |
| Calendar | FullCalendar.js |
| UI | Bootstrap / Tailwind CSS |
| Auth | ASP.NET Core Identity |
| Image Storage | Cloudinary / wwwroot |
| Deployment | Cloud ECS (Linux VM) |
| Monitoring | ASP.NET Core Health Checks |

---

## Architecture

The project is divided into 4 layers following **Clean Architecture** principles. Outer layers depend on inner ones; the dependency inversion principle is strictly followed.

```
BeautyCenterSaaS/
├── BeautyCenterSaaS.Domain/          # Entities, Value Objects, Domain Events
├── BeautyCenterSaaS.Application/     # Use Cases, DTOs, Interface definitions
├── BeautyCenterSaaS.Infrastructure/  # EF Core, Repository, Hangfire, Redis, MailKit
└── BeautyCenterSaaS.Web/             # Controllers, Razor Views, SignalR Hub, Middleware
    ├── Areas/
    │   ├── Admin/                    # Tenant Admin panel
    │   ├── Staff/                    # Staff panel
    │   └── SuperAdmin/               # System management panel
    ├── Controllers/                  # Customer storefront controllers
    ├── Views/
    └── wwwroot/
        ├── js/
        │   ├── calendar.js
        │   ├── reservation.js
        │   └── signalr-client.js
        └── lib/fullcalendar/
```

### Multi-Tenant Isolation

A **Shared Database, Shared Schema** approach is used; every table carries a `TenantId` column.

1. **Tenant Resolution Middleware** fires on every HTTP request
2. `TenantId` is resolved from the URL slug (`lale-guzellik`) and written to `HttpContext`
3. EF Core's `HasQueryFilter` automatically appends `WHERE TenantId = @currentTenant` to every query
4. `TenantId` is included in the JWT/Cookie claim on admin login

> The slugs `admin`, `super`, `hangfire`, and `health` are reserved and cannot be assigned to any business.

---

## User Roles

| Role | Login Method | Permissions |
|------|--------------|-------------|
| **Customer (Guest)** | No account required | Create appointments, cancel, view storefront |
| **Staff (Expert)** | Email + Password | View own calendar, mark appointments as completed / no-show |
| **Tenant Admin** | Email + Password | Full management panel, staff & service management, reports |
| **Super Admin** | Email + Password | Tenant onboarding, system metrics, all businesses |

---

## Appointment Flow

### Customer Side (7 Steps)

```
1. Service Selection    → Categorized list with duration and price
2. Staff Selection      → Staff who can provide the service / "No preference"
3. Date & Time          → FullCalendar.js — only available slots are shown
4. Personal Info Form   → Name, surname, phone, email + reCAPTCHA v3
5. GDPR Consent         → Form cannot be submitted without consent
6. Appointment Created  → Status: Pending
7. Email OTP            → Verify within 15 minutes → Confirmed
                          Timeout → Auto-deleted (Hangfire)
```

### Appointment State Machine

```
Form Submit ──────────────────► Pending
                                   │
              ┌────────────────────┤
              │ OTP / Link verified │ 15 min timeout
              ▼                    ▼
           Confirmed          Auto-Deleted
              │
    ┌─────────┼──────────┐
    │         │          │
    ▼         ▼          ▼
Completed  NoShow    Cancelled
```

| Transition | Who Can Trigger |
|------------|-----------------|
| Pending → Confirmed | System (OTP verification) |
| Confirmed → Completed | Admin, Staff |
| Confirmed → NoShow | Admin, Staff |
| Pending/Confirmed → Cancelled | Customer (magic link, 24h rule), Admin |
| Pending → Deleted | System (Hangfire, 15 min) |

### Magic Link Cancellation

- The confirmation email contains a unique, single-use GUID token
- Cancellation is allowed if the appointment is **more than 24 hours** away
- If less than 24 hours remain, the link responds with `"Cancellation period has expired"`
- When cancelled, an instant notification is pushed to the admin panel via **SignalR**

---

## Getting Started

### Prerequisites

- [.NET 9 SDK](https://dotnet.microsoft.com/download)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (for SQL Server, Redis, MailHog)

### Steps

```bash
# 1. Clone the repository
git clone https://github.com/your-username/beauty-center-saas.git
cd beauty-center-saas

# 2. Start dependencies with Docker
docker compose up -d

# 3. Apply migrations
cd BeautyCenterSaaS.Web
dotnet ef database update

# 4. Run the application
dotnet run
```

The app runs at `https://localhost:5001`. MailHog UI is available at `http://localhost:8025`.

---

## Environment Variables

Fill in `appsettings.Development.json` with your own values:

```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=BeautyCenterSaaS;..."
  },
  "Redis": {
    "ConnectionString": "localhost:6379"
  },
  "Mail": {
    "SmtpHost": "localhost",
    "SmtpPort": 1025,
    "Username": "",
    "Password": ""
  },
  "ReCaptcha": {
    "SecretKey": "YOUR_SECRET_KEY"
  },
  "Cloudinary": {
    "CloudName": "YOUR_CLOUD_NAME"
  },
  "App": {
    "BaseUrl": "https://localhost:5001"
  }
}
```

---

## URL Structure

### Customer Panel (Public Access)

| Route | Description |
|-------|-------------|
| `/{slug}` | Business storefront page |
| `/{slug}/reserve` | Appointment booking flow |
| `/{slug}/confirm?token=` | Email confirmation link |
| `/{slug}/cancel?token=` | Magic link cancellation |

### Admin Panel

| Route | Description |
|-------|-------------|
| `/admin/login` | Tenant Admin / Staff login |
| `/admin/dashboard` | Main dashboard |
| `/admin/calendar` | Interactive calendar (drag-drop) |
| `/admin/appointments` | Appointment list and management |
| `/admin/staff` | Staff management |
| `/admin/services` | Service catalog |
| `/admin/resources` | Device/room management |
| `/admin/unavailability` | Leave and holiday blocks |
| `/staff/calendar` | Staff calendar (read-only) |

### Super Admin Panel

| Route | Description |
|-------|-------------|
| `/super/login` | Super Admin login |
| `/super/dashboard` | System metrics |
| `/super/tenants` | Tenant list and onboarding |
| `/hangfire` | Job dashboard (SuperAdmin only) |
| `/health` | Health check endpoint |

---

## Background Jobs (Hangfire)

| Job | Schedule | Description |
|-----|----------|-------------|
| `SendReminderEmailJob` | Every hour | Sends reminder emails to Confirmed appointments 24 hours before start time |
| `ExpireOtpJob` | Every 5 minutes | Deletes Pending appointments whose OTP has expired |
| `ExpireTokenJob` | Once a day | Clears expired magic link tokens (sets to null) |

> The Hangfire dashboard (`/hangfire`) is accessible only to the `SuperAdmin` role.

---

## Security

### Authentication & Authorization

- **ASP.NET Core Identity** — Cookie-based auth
- Role-based authorization: `[Authorize(Roles="Admin")]`, `[Authorize(Roles="Staff")]`, `[Authorize(Roles="SuperAdmin")]`
- Customer panel allows anonymous access; appointment form is verified via OTP

### Bot & Spam Protection

- **Google reCAPTCHA v3** — Bot protection on the appointment form
- **IP Rate Limiting** — Maximum 3 requests per minute from the same IP
- **OTP Spam Limit** — Maximum 3 Pending appointments per email address within the last hour

### Concurrency Control

Since multiple customers can submit the form for the same slot simultaneously, two-layer protection is applied:

1. **Redis Distributed Lock (RedLock)** — 30-second lock acquired before writing an appointment
2. **EF Core Optimistic Concurrency** — `DbUpdateConcurrencyException` via `RowVersion`

```
Acquire RedLock → Check DB for conflicts → Save appointment → Release lock
```

### Redis Cache Strategy

Live appointment data is always fetched from the DB. Only rarely-changing data is cached:

| Data | Cache Duration |
|------|---------------|
| Tenant info (slug → TenantId) | 60 minutes |
| Service catalog | 30 minutes |
| Staff working hours | 60 minutes |
| Leave blocks | 15 minutes |
| Today's existing appointments | **No cache** — always from DB |

---

## Development Plan

| Sprint | Scope |
|--------|-------|
| Sprint 1 | Clean Architecture skeleton, EF Core, Tenant Resolution Middleware, Global Query Filter |
| Sprint 2 | ASP.NET Core Identity, roles, login/logout, Hangfire & Redis integration |
| Sprint 3 | Tenant onboarding, staff management, service catalog, working hours |
| Sprint 4 | Dynamic slug routing, storefront page, FullCalendar.js (read-only view) |
| Sprint 5 | Slot calculation algorithm, appointment form, reCAPTCHA, OTP, magic link |
| Sprint 6 | Interactive admin calendar, drag-drop, audit log |
| Sprint 7 | SignalR, Hangfire jobs, dashboard metrics and charts |
| Sprint 8 | Rate limiting hardening, health check, integration tests, deployment |

---

## ✨ About the Developer

This project is developed and maintained by **Zişan Yüce**.

**Connect with me:**
- 💼 Full-Stack Developer 
- 🌐 Portfolio: [zisan-yuce.vercel.app](https://zisan-yuce.vercel.app/)
- 📧 Email: yucezisan@gmail.com
- 🔗 LinkedIn: [linkedin.com/in/yucezisan](www.linkedin.com/in/zisanyuce)
- 🐙 GitHub: [@yucezis](https://github.com/yucezis)

*Open to collaboration and feedback!*

---
## License

This project is licensed under the [MIT License](LICENSE).

---


⭐ **If you like this project, don't forget to give it a star!!**
