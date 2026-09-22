# PROPOSECAMPUS - UNIVERSITY MANAGEMENT ERP (LMD SYSTEM)

> Multi-tenant SaaS platform for higher education institutions, built around the LMD system (Licence-Master-Doctorat)

![Status](https://img.shields.io/badge/monolith-complete%20%26%20tested-success) ![Status](https://img.shields.io/badge/API%20%2B%20Desktop-active%20development-blue) ![Python](https://img.shields.io/badge/python-3.11.9-blue) ![Django](https://img.shields.io/badge/django-5.2.1-green) ![React](https://img.shields.io/badge/react-typescript-61DAFB) ![Electron](https://img.shields.io/badge/electron-desktop%20app-47848F) ![PostgreSQL](https://img.shields.io/badge/postgresql-14%2B-blue)

---

## 📌 Project Status

ProposeCampus exists today in two parallel, fully functional forms:

- A **monolithic Django application** (server-rendered templates) — complete, deployed, and tested end-to-end with demo/fictitious data (not yet used by a real university).
- An **API-first version** — Django REST Framework backend + React/TypeScript frontend, additionally packaged as a **desktop application via Electron**, offering offline-capable usage. This version is in active development and already fully functional for most workflows.

Both versions run in parallel and do not interfere with each other. The monolith remains the reference/fallback implementation while the API + Desktop version is progressively completed.

🌐 *A production domain (proposecampus.com) is reserved and will be activated here once the platform is publicly launched.*

---

## ⚠️ Repository Notice

This is a **technical showcase repository**. It documents the architecture, design decisions, and engineering challenges behind ProposeCampus, a proprietary product developed by **Propose Group**. Code excerpts shown are **simplified, illustrative versions** of the real implementation — the production codebase itself is not exposed.

**This repository includes:**
- System architecture documentation
- Offline-first strategy and implementation details
- Database & business logic design (academic statistics engine)
- Technical decision records
- Simplified code excerpts illustrating key patterns

---

## 📋 Table of Contents

- [Overview](#overview)
- [Problem & Solution](#problem--solution)
- [System Architecture](#system-architecture)
- [Technical Stack](#technical-stack)
- [Key Features](#key-features)
- [Offline-First Architecture](#offline-first-architecture)
- [Multi-Tenant Foundation](#multi-tenant-foundation)
- [Security](#security)
- [Technical Challenges Solved](#technical-challenges-solved)
- [Roadmap](#roadmap)
- [Screenshots](#screenshots)
- [Company Context](#company-context)

---

## 🎯 Overview

**ProposeCampus** is a comprehensive ERP designed specifically for higher education institutions operating under the **LMD system** (Licence-Master-Doctorat), widely used across French-speaking Africa and beyond.

It shares its foundational architecture with an earlier secondary-education management project (same core patterns: multi-tenancy, role-based access, subscription billing), but its **data model is entirely different**, built around university-specific concepts — filières, programmes, semesters, ECTS credits, and academic deliberation logic — rather than classes and school cycles.

### Quick Facts

| Aspect | Detail |
|--------|--------|
| **Monolith status** | Complete, deployed, tested (demo data) |
| **API + Desktop status** | Fully functional, active development |
| **User roles** | 7 distinct roles |
| **Subscription tiers** | Standard, Premium |
| **Offline strategy** | 3-tier classification, entity by entity |
| **Development period** | 2025 – Present |

---

## 🔍 Problem & Solution

### The Challenge

Higher education institutions using the LMD system face specific operational challenges rarely addressed by generic school software:

- ❌ **Complex academic structures** — cycles, filières, programmes, semesters, credit systems — poorly handled by tools designed for primary/secondary schools
- ❌ **Heavy deliberation logic** — annual pass/fail decisions depend on ECTS-weighted averages across multiple UEs (teaching units) and semesters, not simple grade averages
- ❌ **Unreliable connectivity** in many campus environments, disrupting day-to-day administrative work
- ❌ **Disconnected financial tracking** between the institution's subscription and individual student fee management
- ❌ **Generic tools force awkward workarounds** to represent LMD-specific academic paths

### Our Solution

A **purpose-built ERP for LMD institutions**, combining a solid multi-tenant foundation with university-specific business logic:

✅ **LMD-native data model** — cycles, filières, programmes, semesters, ECTS-weighted statistics
✅ **Automated academic deliberation engine** — real-time cascading statistics (UE → Semester → Year) driving pass/repeat/exclusion decisions
✅ **Two-tier subscription model** — institution-level billing that automatically provisions parent-level access
✅ **Offline-capable desktop application** — built with Electron, designed for low-connectivity campus environments
✅ **Tiered pricing** — Standard and Premium plans adapted to institution needs

---

## 🏗️ System Architecture

### Current State: Two Parallel Implementations

**1. Monolithic Django Application** (complete, tested)

Server-rendered Django templates, following the same modular app structure proven on the secondary-education project — 15+ apps, multi-tenant middleware, billing gate middleware, PostgreSQL backend, Cloudinary media storage.

**2. API-First Application** (active development)

```
┌───────────────────────────────────────────────┐
│   React + TypeScript Frontend (Vite + Tailwind)│
│   - Runs in browser AND as Electron desktop app│
└──────────────────┬──────────────────────────────┘
                    │  REST + JWT
                    ▼
┌───────────────────────────────────────────────┐
│      Django REST Framework API (Django 5.2.1)  │
│                                                 │
│   Multi-Tenant Middleware (inherited pattern)  │
│   Billing Gate Middleware                      │
│   15+ modular apps (see Key Features)          │
└──────────────────┬──────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────┐
│         PostgreSQL 14+ (Render managed)        │
└───────────────────────────────────────────────┘
```

**Electron Desktop Shell**

The React frontend is packaged as a native desktop application using Electron, with a strict security boundary between the UI (renderer) and system-level access (main process):

```javascript
// main.ts (simplified) — main process, has Node/OS access
const win = new BrowserWindow({
  webPreferences: {
    preload: path.join(__dirname, 'preload.mjs'),
    contextIsolation: true,   // renderer never touches Node directly
    nodeIntegration: false,
  },
})

// Encrypted offline session storage, using Electron's OS-level safeStorage
ipcMain.handle('auth:save-offline-session', (_event, data) => {
  const encrypted = safeStorage.encryptString(JSON.stringify(data))
  fs.writeFileSync(SESSION_FILE, encrypted)
})
```

```javascript
// preload.ts (simplified) — the only bridge between React and the OS
contextBridge.exposeInMainWorld('electronAuth', {
  saveOfflineSession: (data) => ipcRenderer.invoke('auth:save-offline-session', data),
  loadOfflineSession: () => ipcRenderer.invoke('auth:load-offline-session'),
})

// Lets React detect whether it's running as a desktop app or in a browser —
// the switching point used by the Repository pattern (see below)
contextBridge.exposeInMainWorld('isElectron', true)
```

This gives the desktop app an encrypted, OS-level session store (via Electron's `safeStorage`) instead of relying on browser storage — the foundation the offline strategy is built on.

---

## 🛠️ Technical Stack

### Backend
- **Framework:** Django 5.2.1, Python 3.11.9
- **API:** Django REST Framework + SimpleJWT
- **Database:** PostgreSQL 14+ (Render managed)
- **Media Storage:** Cloudinary
- **Email:** Anymail + Brevo
- **Messaging:** Twilio (WhatsApp integration)
- **Payments:** Fedapay (sandbox + production)

### Frontend
- **Framework:** React + TypeScript, built with Vite
- **Styling:** Tailwind CSS
- **State Management:** Zustand (auth, connectivity, dashboard context, toast stores)
- **Forms:** react-hook-form
- **Icons:** lucide-react
- **Desktop Shell:** Electron (secure IPC bridge, encrypted session storage)

### Architecture Patterns
- **Repository pattern** (`repositories/` folder) — designed to eventually let the same UI code transparently switch between calling the live API or a local SQLite store, based on runtime environment (`isElectron` flag)
- **Offline-aware data hooks** (`useOfflineCachedFetch`) and a dedicated `connectivityStore` tracking real-time online/offline status across the app

---

## ✨ Key Features

### 👥 Seven User Roles

Unlike the secondary-education version, ProposeCampus introduces a dedicated **Comptable (Accountant)** role, separate from Secretariat, reflecting the more complex financial operations of higher education institutions:

| Role | Responsibility |
|------|----------------|
| **Étudiant** | Student portal access |
| **Parent/Tuteur** | Follow-up on linked student(s), subscription-gated |
| **Enseignant** | Teaching staff, grading |
| **Saisisseur Pédagogique** | Academic data entry (grades) |
| **Secrétariat Administratif** | Enrollment, administrative workflows |
| **Comptable** | Financial tracking, cash management (dedicated dashboards) |
| **Administrateur Principal** | Full institutional control |

### 📦 Subscription Plans

**Standard** *(price per student/year — pricing not yet finalized)*
- Institutional administration
- Enrollment & re-enrollment workflows
- Academic structure setup (cycles, filières, semesters, programmes)
- Grade entry & report card generation
- Parent portal & notifications
- Basic statistics

**Premium** *(price per student/year — pricing not yet finalized)*

Everything in Standard, plus:
- Offline-capable desktop application
- Online tuition management, payment schedules, digital receipts
- Timetable management
- Advanced analytics
- Data exports (Excel / CSV / PDF)

### 🏫 Modular Application Structure

Following the same separation-of-concerns approach proven on the secondary-education project, with LMD-specific additions:

| App | Responsibility |
|-----|----------------|
| **adminGeneral** | Global admin, academic structure, multi-tenant middleware |
| **etudiant** | Student records, enrollment (LMD-specific model) |
| **enseignant** | Teaching staff management |
| **scolarite** | Tuition fees, tariffs, payment tracking |
| **etab_billing** | Institution-level subscription & billing gate |
| **abonnement** | Parent-level subscription (auto-provisioned, see below) |
| **lmdstats** | Academic statistics engine (UE/Semester/Year cascading stats) |
| **secretaire** | Enrollment & administrative workflows, incl. university-specific re-enrollment |
| **saisisseur** | Grade entry workflows |
| **timetable** | Class scheduling |
| **absence** | Attendance tracking |
| **services/** | Cross-app business logic (seat provisioning, entitlements, active year resolution, re-enrollment automation) |

---

## 📶 Offline-First Architecture

ProposeCampus targets low-connectivity campus environments through a **three-tier offline classification, decided entity by entity** — not a blanket "everything offline" approach, which would be unrealistic given data consistency and financial integrity requirements.

**Tier 1 — Full offline (read + write + sync)**
Local SQLite storage, client-generated IDs, push/pull synchronization on reconnect. *Designed as the target architecture; not yet implemented — see [Roadmap](#roadmap).*

**Tier 2 — Read-only offline (cached consultation)**
Data is cached locally after each successful fetch; if the network is unavailable, the last known cache is displayed with a clear offline indicator. **Implemented and working today.**

**Tier 3 — Online-only strict**
Actions requiring guaranteed consistency (creating a tariff, activating a subscription, editing a user) are disabled outright when offline, with explicit UI feedback rather than silently failing. **Implemented and working today.**

### Illustrative Pattern (simplified)

```typescript
// Reads: try live data, gracefully fall back to cache when offline
const charger = () => {
  listTarifsScolarite()
    .then((data) => {
      setTarifs(data)
      saveCache(CACHE_KEY, data)   // Tier 2: cache every successful read
    })
    .catch(() => {
      const cache = loadCache(CACHE_KEY)
      if (cache) {
        setTarifs(cache.data)
        toast.error('Offline: showing cached tariffs.')
      }
    })
}

// Writes: explicitly blocked offline (Tier 3), with clear user feedback
<Button
  onClick={ouvrirCreation}
  disabled={!isOnline}
  title={!isOnline ? 'Requires an internet connection' : undefined}
>
  New tariff
</Button>
```

This same pattern — cached reads, blocked writes — is applied consistently across financial and administrative screens (tariffs, user management) as a deliberate design choice: prioritize data integrity over write-availability for sensitive operations, while keeping the app fully browsable offline.

The encrypted Electron session store (see [System Architecture](#system-architecture)) is the foundation Tier 1 will build on once implemented, since a secure, persistent local store is a prerequisite for safe offline writes.

---

## 🔐 Multi-Tenant Foundation

ProposeCampus inherits its multi-tenant architecture from the same proven pattern used on the secondary-education project: middleware-based tenant injection, strict row-level filtering (`WHERE etablissement_id = X`), and compound unique constraints on all tenant-scoped models.

Adapting this foundation to a completely different academic data model (LMD structures instead of school cycles/classes) — while preserving the same isolation guarantees — was itself a significant engineering exercise, detailed below.

---

## 🔒 Security

- ✅ Email-based login, JWT authentication (SimpleJWT)
- ✅ Role-based permissions across 7 distinct roles
- ✅ Encrypted local session storage on desktop (Electron `safeStorage`, OS-level encryption)
- ✅ Strict process isolation in Electron (`contextIsolation: true`, `nodeIntegration: false`) — the React UI never has direct access to the file system or Node APIs, only to explicitly exposed, named functions via a secure preload bridge
- ✅ HTTPS enforced, CSRF protection, SQL injection prevention (Django ORM), XSS protection (template auto-escaping)
- ✅ Database encryption at rest (Render managed PostgreSQL)
- ❌ 2FA (not yet implemented, roadmap item)

---

## 💪 Technical Challenges Solved

### Challenge 1: Adapting a Secondary-School Data Model to LMD

**Problem:** The foundational architecture (multi-tenancy, middleware, billing gate, user roles) came from a project built for secondary schools — a fundamentally different academic structure (classes, cycles, grade levels) than higher education (filières, programmes, semesters, ECTS credits).

**Solution:** Rather than starting from scratch, the core architectural patterns (tenant isolation, middleware, authentication, billing gate) were preserved and extended, while the entire academic domain model was redesigned around LMD concepts. This required re-thinking enrollment logic, re-enrollment/promotion rules, and statistics from the ground up, while keeping the proven multi-tenant security guarantees intact.

**Outcome:** A completely different academic product, built on a hardened, previously validated architectural foundation — significantly reducing the risk typically associated with building multi-tenant SaaS from zero.

---

### Challenge 2: Real-Time Academic Deliberation Engine

**Problem:** LMD deliberation (deciding whether a student passes, repeats, or is excluded) depends on cascading, ECTS-weighted averages: a grade affects its Teaching Unit (UE) average, which affects the Semester average, which affects the Annual average — and annual decisions follow specific academic rules (Admis, Enjambement, Redoublement, Exclusion). Recomputing this on-demand for every student would be slow and repeated unnecessarily.

**Solution:** A dedicated statistics engine (`lmdstats` app) maintains three cascading, pre-computed models, updated automatically via Django signals whenever a grade changes:

```python
# Simplified illustrative structure

class UEStat(models.Model):
    """Instant stat per student x ProgrammeUE, recalculated on every grade change."""
    # ...

class SemestreStat(models.Model):
    """Semester-level stat per student, strict ECTS weighting."""
    # ...

class AnneeStat(models.Model):
    """Annual stat per student, combines both semesters. Feeds the deliberation decision."""
    DECISIONS = (
        ('ADMIS', 'Admis'),
        ('ENJAMBEMENT', 'Enjambement'),
        ('REDOUBLEMENT', 'Redoublement'),
        ('EXCLUSION', 'Exclusion'),
    )
    # ...

@receiver(post_save, sender=NotesUniv)
def update_stats_cascade(sender, instance, **kwargs):
    # Recomputes UEStat → SemestreStat → AnneeStat in cascade
    ...
```

**Outcome:** Deliberation-ready statistics are always up to date and instantly available, with no heavy recomputation needed at report-generation time — while encoding real academic decision rules (including "Enjambement", a conditional carry-over case specific to LMD systems) directly into the data model.

---

### Challenge 3: Two-Tier Subscription Model

**Problem:** Two different billing relationships needed to coexist: the **institution** pays an annual subscription (seat-based, per student — similar to the secondary-education project), while **parents** separately need to unlock access to their own child's academic follow-up.

**Solution:** Two distinct apps handle each layer. `etab_billing` manages the institution-level subscription (seats, activation, billing gate enforcement). Upon successful institutional activation, it automatically provisions the corresponding parent-level subscription records — handled by the `abonnement` app — for every parent linked to an enrolled student, rather than requiring parents to separately subscribe from scratch.

**Outcome:** A clean separation of concerns between institutional billing and parent access control, with automatic provisioning removing friction for end users while keeping each billing relationship independently manageable.

---

## 🚀 Roadmap

**Short-Term**
- [ ] Implement Tier 1 offline capability (local SQLite storage, client-generated IDs, push/pull sync) for select high-value entities (starting with grade entry and attendance)
- [ ] Build and distribute the first Electron desktop installer
- [ ] Finalize pricing for Standard and Premium plans
- [ ] Activate the production domain

**Medium-Term**
- [ ] Extend the Repository pattern to fully abstract API vs local SQLite access
- [ ] Complete migration away from the monolith once the API + Desktop version reaches full feature parity
- [ ] 2FA for administrative roles

**Long-Term**
- [ ] Multi-currency / multi-country support
- [ ] Advanced institutional analytics (cohort trends, retention, success rate forecasting)

---

## 📸 Screenshots

> **Note:** All screenshots use anonymized/demo data. No real student or institutional information is displayed.

**1. Login & Institution Setup** — Authentication and multi-tenant onboarding flow
**2. Admin Dashboard** — Institutional overview, quick actions
**3. Academic Structure Management** — Cycles, filières, programmes, semesters configuration
**4. Grade Entry** — Saisisseur workflow for academic data entry
**5. Student Statistics** — UE/Semester/Annual cascading stats, deliberation status
**6. Tariffs Management** — Tuition configuration, showing the offline-aware read/write pattern
**7. Parent Portal** — Subscription-gated child follow-up
**8. Desktop Application** — Electron shell running the same interface natively

*(Image files to be referenced in `docs/screenshots/`)*

---

## 🏢 Company Context

ProposeCampus is a proprietary product developed and owned by **Propose Group**. Unlike other projects in this portfolio built in partnership with external organizations, ProposeCampus is fully in-house: conceived, architected, and developed solo as lead developer.

**My role covers the entire product:**
- ✅ Full system architecture & design (backend, frontend, desktop)
- ✅ Solo development across Django monolith, DRF API, React frontend, and Electron shell
- ✅ Academic domain modeling (LMD-specific structures and deliberation logic)
- ✅ Multi-tenant architecture adaptation from an existing proven foundation
- ✅ Offline-first strategy design and implementation
- ✅ Billing architecture (institutional + parent subscription layers)
- ✅ DevOps & deployment (Render, Cloudinary)

---

## 📄 Technical Documentation

Detailed documentation available in `/docs`:
- Offline architecture guide
- Academic statistics engine (deliberation logic)
- Multi-tenant architecture (inherited pattern)
- API documentation

---

## 🛡️ License

**Proprietary Software** — Owned by Propose Group. The actual codebase is not open-source. This repository contains architectural documentation and simplified illustrative code excerpts for portfolio purposes only.

Documentation in this repository: MIT License

---

## 📧 Contact

- **Developer:** Gabaki Borise Balode - BGB
- **Email:** gborisebalode@gmail.com
- **LinkedIn:** [linkedin.com/in/gabakibalodebgb](https://linkedin.com/in/gabakibalodebgb)
- **Portfolio:** [bgb-portfolio.vercel.app](https://bgb-portfolio.vercel.app)

---

## 🙏 Acknowledgments

Built as part of Propose Group's product portfolio, extending proven multi-tenant SaaS patterns into the higher education space.

---

**Last Updated:** September 2026

---
