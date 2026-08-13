# IT Training Management — Cloud Application

A cloud-based platform for managing IT training programs, built as a decoupled system: a **Vue.js SPA** on Firebase Hosting consuming a **Laravel REST API** on Google App Engine.

Trainees register, apply to training programs, upload materials, and book meetings with advisors. Advisors manage their assigned trainees and approve meeting requests. Managers approve registrations, issue trainee IDs, and oversee the whole system.

**Live demo:** https://ittrainingsystem.web.app/login

---

## Contributors

Academic team project — *Advanced Software Engineering (SDEV 4304)*, Faculty of Information Technology, Islamic University of Gaza.
Instructor: Dr. Rebhi S. Baraka.

| Contributor | Area |
|---|---|
| **Ahmed Hammad** (this repo) | **Backend API (46 of 49 commits), cloud deployment, database design** |
| Mohamed Sokar | Frontend (Vue.js SPA) |
| Zakaria Harara | Frontend, API integration |
| Mohammed Akram | Mail notifications |

---

## Demo Accounts

The demo runs in **test mode** — no real payments are processed.

| Role | Login URL | Credentials |
|---|---|---|
| Trainee | `/login/trainee` | ID `7137660` · password `12345678` |
| Advisor | `/login/supervisor` | `rbaraka@ittraining.com` · `12345678` |
| Manager | `/login/manager` | `admin@admin.com` · `12345678` |

Stripe test card: `4242 4242 4242 4242`, any future expiry, any 3-digit CVC.

---

## Screenshots

> _Add 4–5 images to a `docs/` folder and link them here._

| Login | Programs |
|---|---|
| ![Login](docs/login.png) | ![Programs](docs/programs.png) |

| Meetings | Payment |
|---|---|
| ![Meetings](docs/meetings.png) | ![Payment](docs/payment.png) |

---

## Architecture

```
┌─────────────────────────┐         ┌──────────────────────────┐
│   Vue.js SPA            │  HTTPS  │   Laravel REST API       │
│   Firebase Hosting      │ ──────► │   Google App Engine      │
│   Firebase Auth + SDK   │  JSON   │   Sanctum token auth     │
└─────────────────────────┘         └────────────┬─────────────┘
                                                 │
                          ┌──────────────────────┼──────────────────────┐
                          ▼                      ▼                      ▼
                 ┌─────────────────┐   ┌──────────────────┐   ┌─────────────────┐
                 │  Cloud SQL      │   │  Cloud Storage   │   │  Mail service   │
                 │  (MySQL)        │   │  files & logos   │   │  notifications  │
                 └─────────────────┘   └──────────────────┘   └─────────────────┘
```

**Repository layout**

```
BackendTrainingSystem/     Laravel REST API — deployed to Google App Engine
frontend-training-system/  Vue.js SPA — deployed to Firebase Hosting
cors.json                  Cloud Storage CORS configuration
```

---

## Tech Stack

**Backend** — PHP, Laravel, Laravel Sanctum, Eloquent ORM
**Frontend** — Vue.js, JavaScript, HTML5, CSS3, Firebase SDK
**Database** — Cloud SQL (MySQL)
**Cloud** — Google App Engine, Firebase Hosting, Google Cloud Storage, Firebase Analytics
**Payments** — Stripe
**Tools** — Git, Postman, PhpStorm, VS Code

---

## Backend Overview

The API exposes **40+ endpoints** across **13 controllers** and **11 Eloquent models**.

### Authentication

Token-based authentication with **Laravel Sanctum**, using separate login flows per role so each user type authenticates against its own credentials:

```
POST /api/trainee/login      POST /api/advisor/register
POST /api/advisor/login      POST /api/manager/register
POST /api/manager/login      POST /api/logout/{id}
```

All protected routes are gated server-side with the `auth:sanctum` middleware — direct URL access without a valid token is rejected regardless of what the frontend allows.

### Meeting Scheduling with Conflict Detection

Meeting requests are validated against an advisor's existing bookings before being persisted. The overlap query catches all three collision cases — a new meeting starting inside an existing slot, ending inside one, or fully containing one:

```php
$conflictingMeetings = Meeting::where('advisor_id', $validatedData['advisor_id'])
    ->where(function ($query) use ($validatedData) {
        $query->whereBetween('start_time', [$start, $end])
            ->orWhereBetween('end_time', [$start, $end])
            ->orWhere(function ($query) use ($validatedData) {
                $query->where('start_time', '<=', $start)
                      ->where('end_time',   '>=', $end);
            });
    })
    ->count();

if ($conflictingMeetings > 0) {
    return response()->json(
        ['error' => 'There is a scheduling conflict for the requested meeting'], 409
    );
}
```

On success the meeting is created and a notification is dispatched to the advisor.

### Core Endpoints

| Resource | Endpoints |
|---|---|
| Trainees | `apiResource /trainees`, `/trainee/profile`, `/trainees/payment` |
| Advisors | `apiResource /advisors`, `/advisor/get-meetings/{id}`, `/advisor/acceptMeeting/{id}`, `/advisor/list-trainees` |
| Managers | `apiResource /managers`, `/manager/accept-trainee/{id}`, `/manager/show-training-requests` |
| Programs | `apiResource /programs`, `/programs/get-logo/{id}` |
| Meetings | `/trainees/create-meeting`, `/trainee/list-meetings-requests`, `/manager/list-meetings-requests` |
| Files | `apiResource /files`, `/trainees/get-files/{id}` |
| Requests | `/trainee/apply-program`, `/trainees/show-training-requests/{id}` |
| Attendance | `/trainees/attendance` |
| Notifications | `apiResource /notifications` |

### Data Model

11 models with foreign-key relationships: `User`, `Trainee`, `Advisor`, `Manager`, `Program`, `Discipline`, `Meeting`, `TrainingRequest`, `TraineeAttendance`, `StoredFile`, `Notification`.

`User` holds authentication and role, with `Trainee` / `Advisor` / `Manager` as role-specific profiles. Advisors are classified by `Discipline`; trainees link to programs through a `program_trainee` pivot table.

---

## Running Locally

**Requirements:** PHP 8.1+, Composer, MySQL 8, Node.js 16+

### Backend

```bash
cd BackendTrainingSystem
composer install
cp .env.example .env
php artisan key:generate
```

Set your database credentials in `.env`, then:

```bash
php artisan migrate
php artisan serve
```

API available at `http://localhost:8000/api`.

### Frontend

```bash
cd frontend-training-system
npm install
npm run dev
```

Point the frontend at your local API base URL before running.

> **Note:** `.env` is git-ignored and contains credentials — never commit it.

---

## Deployment

**Backend → Google App Engine**

```bash
gcloud app deploy app.yaml
```

Runtime and resources are declared in `app.yaml`; Cloud SQL is attached as the managed MySQL instance and Cloud Storage handles uploaded files and program logos.

**Frontend → Firebase Hosting**

```bash
npm run build
firebase deploy --only hosting
```

CORS for the storage bucket is configured via `cors.json`.

---

## Features

- Trainee registration with document upload and manager approval workflow
- Auto-generated unique trainee IDs, emailed on approval
- Training program browsing, application, and qualification submission
- Advisor assignment by discipline
- Meeting requests with server-side conflict resolution
- File uploads (videos, images, code, documents) reviewable by advisors
- Attendance forms
- Email and in-app notifications
- Stripe payment gateway for account activation
- Manager dashboard for accounts, requests, and system oversight
- Firebase Analytics event tracking

---

## License

Academic project — released for educational and portfolio purposes.