# Medplum Migration Design Spec

## Overview

Migrate Trimly's backend from Plato Medical to a self-hosted Medplum (FHIR R4) server. Retain similar frontend flows. New Nx monorepo (`trimly-clinic`) containing the booking funnel, patient mobile app, Medplum bots, and shared packages. The marketing/landing site stays in a separate repo.

## Goals

- Replace Plato API with self-hosted Medplum (FHIR-compliant, open-source)
- Use Medplum's built-in auth, subscriptions, and bots to eliminate custom infrastructure (NextAuth, Prisma, BullMQ, Redis workers)
- Support multiple frontends: public booking funnel (Next.js), patient app (Expo/React Native), provider portal (Medplum built-in admin app)
- Deploy on Fly.io initially, with a path to AWS later

## Non-Goals

- Marketing/landing site migration (separate repo)
- In-app messaging (future roadmap)
- Custom provider portal (use built-in Medplum admin app for now)

---

## Repo Structure

Nx integrated monorepo with pnpm workspaces.

```
trimly-clinic/
├── nx.json
├── tsconfig.base.json
├── package.json
├── docker-compose.yml                  # Local dev: Postgres + Redis + Medplum server + admin app
│
├── apps/
│   ├── booking/                        # Next.js — public booking funnel
│   │   ├── project.json
│   │   ├── src/
│   │   │   ├── app/                    # Next.js App Router
│   │   │   │   ├── page.tsx            # Funnel entry point
│   │   │   │   ├── book/              # Slot selection, patient details, confirmation
│   │   │   │   └── layout.tsx
│   │   │   └── lib/
│   │   │       └── medplum-client.ts   # MedplumClient configured for booking client
│   │   └── next.config.ts
│   │
│   └── patient-app/                    # Expo (React Native)
│       ├── project.json
│       ├── app/                        # Expo Router (file-based routing)
│       │   ├── (auth)/                 # Login, register
│       │   ├── (tabs)/                 # Home, appointments, profile, medications
│       │   └── _layout.tsx
│       ├── src/
│       │   └── lib/
│       │       └── medplum-client.ts   # MedplumClient + @medplum/expo-polyfills
│       └── app.json
│
├── packages/
│   ├── bots/                           # Medplum Bots
│   │   ├── project.json
│   │   ├── medplum.config.json         # Bot ID <-> source mappings
│   │   ├── src/
│   │   │   ├── on-appointment-created.ts
│   │   │   ├── on-patient-registered.ts
│   │   │   ├── on-questionnaire-response.ts
│   │   │   ├── check-abandoned-carts.ts
│   │   │   ├── send-appointment-reminders.ts
│   │   │   ├── generate-slots.ts
│   │   │   └── process-payment.ts
│   │   └── esbuild.config.mjs
│   │
│   └── shared/                         # Shared code
│       ├── project.json
│       └── src/
│           ├── index.ts
│           ├── clinic-config.ts        # Calendar IDs, clinic details, constants
│           ├── fhir-helpers.ts         # Typed helpers for creating Patients, Appointments, etc.
│           └── validation.ts           # Zod schemas shared between booking + patient app
│
├── infra/
│   ├── fly.toml                        # Medplum server
│   ├── fly.admin.toml                  # Medplum admin app (provider portal)
│   ├── fly.booking.toml                # Booking funnel
│   ├── Dockerfile.medplum              # Wraps medplum/medplum-server with custom config
│   └── seed/                           # FHIR Bundle JSONs loaded on first deploy
│       ├── access-policies.json
│       ├── client-applications.json
│       ├── schedules.json
│       └── bot-subscriptions.json
│
└── scripts/
    ├── seed.sh                         # Load seed bundles into Medplum
    └── deploy-bots.sh                  # npx medplum bot deploy
```

---

## Data Model: Plato to FHIR Mapping

| Plato Concept | FHIR Resource | Notes |
|---|---|---|
| Patient (name, email, phone, dob, sex, address) | **Patient** | Email → `telecom[system=email]`, phone → `telecom[system=phone]`, address → `address[]` |
| Appointment (patient_id, starttime, endtime) | **Appointment** + **Slot** | Appointment.participant references Patient + Practitioner. Slot tracks availability. |
| Calendar / Available slots | **Schedule** + **Slot** | Schedule belongs to a Practitioner. Slots auto-generated weekly by a cron bot based on practitioner working hours. |
| Questionnaire responses (free-text patient notes) | **Questionnaire** + **QuestionnaireResponse** | Structured FHIR resource. Questionnaire defines the intake form. QuestionnaireResponse stores answers. |
| Video consultation link | **Appointment extension** | Daily.co room URL stored as extension on Appointment. |
| Clinical notes | **Encounter** + **DocumentReference** | Encounter records the consult. Doctor writes notes as DocumentReference. |
| Prescription | **MedicationRequest** + **MedicationDispense** | MedicationRequest `status: active` (prescribed) → `completed`. MedicationDispense tracks fulfillment: `preparation → in-progress → completed`. Delivery status via extension on MedicationDispense. |
| PlatoPay cards | **Patient extension** (Stripe Customer ID) | Stripe PaymentMethod saved during booking. Customer ID stored as extension on Patient. |
| Abandoned cart | **Task** | Task resource with type=abandoned-cart, deadline period. Cron bot checks for overdue tasks. |
| User auth (NextAuth) | **Medplum Auth** | Built-in OAuth2/OIDC. Replaces NextAuth entirely. |
| Verification tokens (Prisma) | **Medplum Auth** | Handled by Medplum's auth system. |
| Bookings table (Prisma) | **Appointment** | Appointment resource with status=booked. No separate tracking table needed. |

### What Gets Eliminated

- Prisma + PostgreSQL (own DB) — Medplum's Postgres stores everything
- BullMQ + Redis workers — replaced by Medplum Bots + built-in job queue
- NextAuth — replaced by Medplum OAuth2/OIDC
- Redis cache layer (patients, slots, cards) — Medplum handles caching internally

### External Services Retained

- **Stripe** — payment processing (replaces PlatoPay)
- **Twilio** — WhatsApp notifications
- **Mailgun / SMTP** — email notifications
- **Discord** — admin webhook alerts
- **Daily.co** — video consultations (replaces Plato Connect)

---

## Auth & Access Policies

### Client Applications

| Client | Auth Flow | Who Uses It |
|---|---|---|
| `booking-client` | Client credentials (server-side for slot browsing) → patient auth after registration | Public visitors booking appointments |
| `patient-client` | OAuth2 PKCE (patient login/registration) | Patients in the Expo app |
| `admin-client` | OAuth2 (practitioner login) | Doctors/staff using the built-in Medplum admin app |

### Access Policies

- **Booking Access Policy** — read `Schedule`, `Slot`, `Questionnaire`. Create `Patient`, `Appointment`, `QuestionnaireResponse`. Cannot read other patients' data.
- **Patient Access Policy** — scoped to own data. Read/write own `Patient`, `Appointment`, `QuestionnaireResponse`. Read-only: `MedicationRequest`, `MedicationDispense`, `Encounter`, `Communication`. Cannot see other patients.
- **Practitioner Access Policy** — full read/write on clinical resources. Managed by admin app defaults.

### Booking Funnel Auth Flow

1. **Unauthenticated phase** — booking app uses client credentials (server-side Next.js API routes) to fetch Schedules and Slots. No patient login needed to browse availability or fill out the quiz.
2. **Registration** — when patient submits details (name, email, phone), the booking app calls Medplum `/auth/newpatient` to create a Medplum account + FHIR Patient resource. Patient receives a session token.
3. **Authenticated phase** — subsequent actions (Stripe card setup, Appointment creation) happen under the patient's auth token.
4. **Returning patients** — if the patient already has an account, they log in first (email/password or Google OAuth), then proceed through the funnel.
5. **Cross-app auth** — same credentials work in the Expo patient app. Google OAuth supported via Medplum's external identity provider configuration.

---

## Bot Architecture

Runtime: `vmcontext` (in-process Node.js vm sandbox, no AWS Lambda needed).

**Server config requirements:** Medplum server must be configured with `vmContextBotsEnabled: true` and `cronEnabled: true` (or `workers.enabled` must include `"cron"`). Without `cronEnabled`, cron bots will silently not execute.

### Event-Triggered Bots (FHIR Subscriptions)

| Bot | Trigger | What It Does |
|---|---|---|
| `on-appointment-created.ts` | Appointment created, `status=booked` | 1. Creates Daily.co room via API, patches URL onto Appointment. 2. Sends confirmation email (SMTP). 3. Sends WhatsApp confirmation (Twilio). 4. Sends Discord admin alert. |
| `on-patient-registered.ts` | Patient created | Creates a Task resource (type=abandoned-cart) with a 2-hour deadline period. |
| `on-questionnaire-response.ts` | QuestionnaireResponse created | Validates responses, links to Patient. |
| `process-payment.ts` | Triggered via Bot $execute (admin action) | Charges the patient's saved Stripe PaymentMethod. Updates `paymentStatus` extension on MedicationDispense. |

### Cron Bots

| Bot | Schedule | What It Does |
|---|---|---|
| `check-abandoned-carts.ts` | Every 30 min | Searches for Task resources (abandoned-cart type) past deadline where Patient has no booked Appointment. Respects quiet hours (10pm-8am SGT) — skips sending and defers to next run after 8am. Sends WhatsApp reminder via Twilio. |
| `send-appointment-reminders.ts` | Daily 7am SGT | Searches for Appointments starting in next 24h. Sends reminder WhatsApp + email. |
| `generate-slots.ts` | Weekly (Sunday midnight SGT) | Creates Slot resources for the upcoming 2 weeks based on practitioner Schedule business rules (configurable working hours, break times). Marks existing unfilled past slots as `busy-unavailable`. |

### Bot Secrets

Stored on each Bot resource (not in .env files):
- `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`
- `SMTP_HOST`, `SMTP_USER`, `SMTP_PASS`
- `DISCORD_WEBHOOK_URL`
- `DAILY_API_KEY`
- `STRIPE_SECRET_KEY`

---

## Payment Flow

1. **Booking funnel** — after patient registration, collect card via Stripe Elements (Setup Intent). No charge.
2. **Store on Patient** — save Stripe Customer ID as an extension on the FHIR Patient resource. PaymentMethod ID associated with the Stripe Customer.
3. **Post-consultation** — doctor creates MedicationRequest in admin app.
4. **Admin charges** — clinic staff triggers payment from admin app (calls `process-payment` bot via `$execute`). Bot charges the saved PaymentMethod via Stripe API.
5. **Status tracking** — payment status tracked as an extension on the MedicationDispense resource (`paymentStatus: pending → charged → failed`).

---

## Video Consultation (Daily.co)

- **Room creation:** `on-appointment-created` bot calls `POST https://api.daily.co/v1/rooms` with expiry set to appointment end time + 30 min buffer.
- **URL storage:** Room URL patched onto the Appointment resource as an extension.
- **Patient joins:** Patient app opens the Daily.co URL (Daily.co has a React Native SDK).
- **Doctor joins:** Admin app displays the URL; doctor clicks to join in browser.
- **Cost:** ~$0.16 per 20-min consultation (2 participants x $0.004/min).

---

## Clinical Workflow

1. Patient books appointment via booking funnel
2. Confirmations sent automatically (bot)
3. Doctor joins video call via Daily.co link in admin app
4. Doctor records **Encounter** (the consult) in admin app
5. Doctor creates **MedicationRequest** (prescription) in admin app
6. Admin staff processes payment (triggers `process-payment` bot)
7. Pharmacy team creates MedicationDispense, tracks fulfillment + delivery
8. Patient sees medication status in Expo app
9. Admin staff books follow-up Appointment for patient
10. (Later) Patient can reschedule/cancel own appointments in Expo app

---

## Pharmacy / Medication Tracking

- Doctor creates **MedicationRequest** in admin app after consultation (`status: active`)
- Pharmacy staff creates **MedicationDispense** when fulfilling the prescription (`status: preparation → in-progress → completed`)
- Delivery tracked via extension on MedicationDispense (`deliveryStatus: pending → shipped → delivered`)
- When fully delivered, MedicationRequest updated to `status: completed`
- Patient app displays MedicationRequest + MedicationDispense status (read-only)
- Dispensing and delivery handled operationally by clinic staff via admin app — no automation needed initially

---

## Local Development

```
docker-compose up          # Postgres + Redis + Medplum server (:8103) + Admin app (:3000)
scripts/seed.sh            # Load AccessPolicies, Schedules, ClientApplications, Subscriptions
nx serve booking           # Next.js booking funnel on :3100
nx start patient-app       # Expo dev server
```

Seed bundles in `infra/seed/` ensure the local environment is ready with Schedules, Slots, AccessPolicies, and ClientApplications on first boot.

---

## Deployment (Fly.io)

| Service | What | Config |
|---|---|---|
| `trimly-medplum` | Medplum server (Docker) | `fly.toml` — shared-cpu-1x, Fly Postgres, Upstash Redis |
| `trimly-admin` | Medplum admin app (Docker) | `fly.admin.toml` — static SPA pointing at server |
| `trimly-booking` | Next.js booking funnel | `fly.booking.toml` — standalone Next.js output |
| Patient app | Expo → App Store / Google Play | EAS Build + EAS Submit (not on Fly) |

Bot deployment: `npx medplum bot deploy <bot-name>` from `packages/bots/`, scriptable in CI.

### CI Pipeline (GitHub Actions)

1. `nx affected --target=lint,test` — only lint/test changed projects
2. `nx affected --target=build` — only build affected apps
3. Deploy changed apps to Fly.io
4. If bots changed → run `deploy-bots.sh`

---

## Migration Phases

### Phase 1 — Scaffold & Core
- Set up Nx monorepo, docker-compose, seed data
- Get Medplum server running locally with Schedules, Slots, AccessPolicies
- Build `packages/shared` (FHIR helpers, validation schemas)
- Configure Stripe account + Daily.co account

### Phase 2 — Booking Funnel
- Build `apps/booking` — port existing funnel UI from plato-form
- Implement: quiz → auth (Medplum) → patient registration → Stripe card vault → slot selection → appointment creation
- Replaces: PlatoAPI.getAvailableOnlineSlots, createPatient, createAppointment

### Phase 3 — Bots & Notifications
- Build and deploy all bots
- Configure secrets (Twilio, SMTP, Discord, Daily.co, Stripe)
- End-to-end test: book appointment → video link created → confirmations sent

### Phase 4 — Patient App
- Build `apps/patient-app` in Expo
- Patient login via Medplum OAuth2 PKCE
- View appointments, medication status, weight tracking (Observation resources)
- (Later) Reschedule/cancel appointments

### Phase 5 — Data Migration & Cutover
- Migration script: pull patients from Plato API → create FHIR Patient resources in Medplum
- Migrate appointment history if needed
- **Payment cards cannot be migrated** from PlatoPay to Stripe — patients will need to re-enter card details on their next booking
- Deploy all services to Fly.io
- Run parallel operation period: both Plato and Medplum live, new bookings go to Medplum
- DNS cutover, decommission Plato

---

## Future Enhancements

- **Custom provider portal** (`apps/provider-portal`) — if the built-in admin app becomes limiting
- **In-app messaging** — FHIR Communication resources, real-time via Medplum WebSocket subscriptions
- **AWS migration** — use `@medplum/cdk` for ECS/Fargate + RDS + S3 deployment
- **Patient self-booking for follow-ups** — in Expo app
- **Progress tracking dashboards** — weight/BMI trends via FHIR Observation resources
