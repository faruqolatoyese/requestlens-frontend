# RequestLens: Real-Time API Observability & Telemetry Dashboard (Frontend)

**RequestLens** is a real-time API monitoring, telemetry ingestion, and automated alerting platform built for modern engineering teams. It enables developers to track endpoint latency, visualize traffic and status code distributions, inspect live request streams, and receive automated incident alerts when `5xx` server errors spike.

> **Multi-Repository Architecture**
> - **Frontend Repository (This Repo):** [faruqolatoyese/requestlens-frontend](https://github.com/faruqolatoyese/requestlens-frontend)
> - **Backend Repository:** [faruqolatoyese/requestlens-backend](https://github.com/faruqolatoyese/requestlens-backend)

---

## Live Links & Documentation

| Resource | URL | Description |
| :--- | :--- | :--- |
| **Live Web Application** | [requestlens-frontend.vercel.app](https://requestlens-frontend.vercel.app) | Production Next.js Frontend Dashboard |
| **Production API Server** | [requestlens-backend.onrender.com](https://requestlens-backend.onrender.com) | Express + Socket.IO Telemetry & REST API |
| **Interactive API Docs** | [requestlens-backend.onrender.com/api-docs](https://requestlens-backend.onrender.com/api-docs) | OpenAPI 3.0 / Swagger UI Documentation |
| **Backend Repository** | [github.com/faruqolatoyese/requestlens-backend](https://github.com/faruqolatoyese/requestlens-backend) | Express, Prisma, PostgreSQL, Redis, BullMQ & Socket.IO Backend |

---

## The Problem We Are Solving

Engineering teams frequently face silent API degradations such as creeping endpoint latency, unexpected traffic spikes, or sudden `5xx` error surges that go unnoticed until end-users report broken workflows. Traditional Application Performance Monitoring (APM) suites are often heavyweight, expensive, and require complex infrastructure agents.

**RequestLens** solves this with a streamlined, developer-first observability workflow:
1. **Zero-Friction Integration:** Create a project, generate an API key, register your routes, and drop a 10-line middleware snippet (Express, FastAPI, or cURL) into your backend in under 5 minutes.
2. **Instant Real-Time Visibility:** Instead of polling or waiting for delayed log indexing, RequestLens streams ingested telemetry logs directly to your browser over authenticated **WebSockets (Socket.IO)**, updating charts and log tables the millisecond a request finishes.
3. **Built-In Traffic Simulation:** Test your observability pipeline and alert thresholds immediately from the dashboard using the interactive **Log Ingestor Simulator** no external traffic generator required.
4. **Noise-Free Automated Alerting:** Configure granular `5xx` error rate rules with sliding time windows, minimum request volume guards, and cooldown periods so your team receives actionable incident emails without inbox spam.

---

## System Architecture & End-to-End Data Flow

RequestLens is split into two decoupled services (**`requestlens-frontend`** and **`requestlens-backend`**) that communicate via REST APIs and persistent WebSocket channels.

### How the Frontend & Backend Work Together

1. **Dual Authentication Architecture:**
   - **Developer Dashboard Auth (`Bearer <JWT>`):** Users sign in via Email/Password or **Google OAuth 2.0** (`GoogleButton` using Google Identity Services). The backend verifies credentials, issues a signed JWT, and the frontend attaches it to all `/api/projects/*` REST requests and the Socket.IO handshake.
   - **Telemetry Ingestion Auth (`Bearer <API_KEY>`):** Each project is provisioned a unique API key (`ApiKey` model). When a monitored server or the frontend's **Log Ingestor Simulator** sends telemetry to `POST /api/logs`, the backend's `apiKeyMiddleware` validates the key and resolves the target `projectId`.

2. **Real-Time Telemetry Synchronization:**
   - When the user opens a project dashboard (`/dashboard/[projectId]`) or logs view (`/dashboard/[projectId]/logs`), the custom `useSocket` hook establishes an authenticated Socket.IO connection and emits `join_project` with the `projectId`.
   - The backend verifies that the authenticated user owns the project before joining the socket to the isolated `projectId` room.
   - Whenever a new log is ingested at `POST /api/logs`, the backend persists the record in PostgreSQL and emits `new_log` to the `projectId` room.
   - The frontend receives `new_log` and triggers `queryClient.invalidateQueries()` on `["projectStats", projectId]` and `["projectLogs", projectId]`, seamlessly refreshing KPI metrics, traffic charts, and paginated logs in real time.

3. **Asynchronous `5xx` Alerting Pipeline:**
   - Users configure `AlertRule` thresholds in `/dashboard/[projectId]/settings` (e.g., *"Trigger if `5xx` error rate `>= 5%` over a `5-minute` window with at least `10` requests, then enter a `30-minute` cooldown"*).
   - On every log ingestion, the backend evaluates active rules asynchronously so ingestion latency is never blocked.
   - If a rule's threshold is breached and the rule is not in cooldown, an `AlertLog` record is created and a job is pushed to **BullMQ (`alert-emails`)** backed by **Redis**, which dispatches a detailed HTML breakdown of failing endpoints via **Resend**.

---

## Key Features (Frontend)

- **Real-Time Observability Overview (`/dashboard/[projectId]`):**
  - **KPI Summary Cards:** Track Total API Requests, Failed Requests (`4xx`/`5xx`), and overall Success Rate (`%`) with health indicators.
  - **7-Day Traffic Composed Chart:** Stacked bar visualization comparing daily successful (`2xx`) vs. failed (`4xx`/`5xx`) request volumes using **Recharts**.
  - **Status Code Distribution:** Interactive donut chart breaking down `2xx Success` vs. `4xx/5xx Errors`.
  - **Per-Endpoint Performance:** Monitor individual route request counts, average response latency (`ms`), and route-specific success rates.
- **Dynamic SDK & Middleware Snippets:**
  - Copy-ready integration snippets dynamically populated with your project's active **API Key** and selected **Endpoint ID** for **cURL**, **Express.js (Node.js)**, and **FastAPI (Python)**.
- **Interactive Log Ingestor Simulator:**
  - Built-in control panel to fire synthetic telemetry requests with custom HTTP status codes (`200`, `201`, `400`, `401`, `403`, `404`, `500`) and response latencies (`45ms` to `1800ms`) to verify live WebSocket updates and test alert rules.
- **Endpoint Registry (`/dashboard/[projectId]/endpoints`):**
  - Register, inspect, and remove monitored API routes (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`) with method-specific color badges.
- **Live API Logs Explorer (`/dashboard/[projectId]/logs`):**
  - Real-time paginated log stream displaying millisecond-precision timestamps, HTTP methods, endpoint paths, status codes, and response times (with visual warnings for slow requests `> 500ms`), plus instant search filtering.
- **Alerting Rules & Credential Management (`/dashboard/[projectId]/settings`):**
  - Reveal/copy project API keys, create/toggle/delete `5xx` threshold alert rules, and manage project lifecycle.

---

## Tech Stack

### Frontend (`requestlens-frontend`)
- **Framework:** [Next.js 16](https://nextjs.org/) (App Router, React 19, TypeScript)
- **State & Data Fetching:** [TanStack React Query v5](https://tanstack.com/query/latest)
- **Real-Time Client:** [Socket.IO Client](https://socket.io/docs/v4/client-api/)
- **Data Visualization:** [Recharts](https://recharts.org/)
- **Styling & UI Components:** [Tailwind CSS v4](https://tailwindcss.com/), [Radix UI](https://www.radix-ui.com/), [shadcn/ui](https://ui.shadcn.com/), [Framer Motion](https://www.framer.com/motion/), [Lucide Icons](https://lucide.dev/)
- **Forms & Validation:** [React Hook Form](https://react-hook-form.com/) + [Zod](https://zod.dev/)
- **Authentication:** JWT Bearer Auth + Google Identity Services (`accounts.google.com/gsi/client`)

### Backend (`requestlens-backend`)
- **Runtime & Framework:** Node.js, Express 5, TypeScript
- **Database & ORM:** PostgreSQL + Prisma ORM
- **Real-Time Server:** Socket.IO
- **Queue & Background Jobs:** Redis (`ioredis`) + BullMQ
- **Transactional Emails:** Resend
- **API Documentation:** OpenAPI 3.0 / Swagger UI (`swagger-ui-express`)

---

## Project Structure

```text
requestlens-frontend/
├── app/
│   ├── (auth)/
│   │   ├── login/page.tsx                # Email/Password & Google Sign-In page
│   │   ├── register/page.tsx             # Account registration page
│   │   └── layout.tsx                    # Auth layout wrapper
│   ├── dashboard/
│   │   ├── [projectId]/
│   │   │   ├── endpoints/page.tsx        # Register & manage monitored HTTP endpoints
│   │   │   ├── logs/page.tsx             # Live-updating paginated request logs & search
│   │   │   ├── settings/page.tsx         # API credentials, 5xx Alert Rules & Danger Zone
│   │   │   └── page.tsx                  # Project Overview: charts, snippets & Log Simulator
│   │   ├── layout.tsx                    # Authenticated dashboard shell & navigation
│   │   └── page.tsx                      # Projects workspace & API Key generation modal
│   ├── globals.css                       # Global Tailwind v4 styles & theme variables
│   ├── layout.tsx                        # Root layout with React Query Providers
│   └── page.tsx                          # Marketing landing page with interactive preview
├── components/
│   ├── ui/                               # Reusable UI primitives (Button, Card, Table, Modal, etc.)
│   ├── GoogleButton.tsx                  # Google OAuth 2.0 GSI button integration
│   ├── footer.tsx                        # Application footer
│   └── providers.tsx                     # TanStack QueryClientProvider wrapper
├── hooks/
│   ├── useAlerts.ts                      # React Query hooks for CRUD & toggling Alert Rules
│   ├── useAnalytics.ts                   # Hooks for project stats, paginated logs & simulator
│   ├── useAuth.ts                        # Hooks for login, register, Google OAuth & logout
│   ├── useEndpoints.ts                   # Hooks for listing, creating & deleting endpoints
│   ├── useProjects.ts                    # Hooks for project workspace management
│   └── useSocket.ts                      # Socket.IO lifecycle & room subscription hook
└── lib/
    ├── api.ts                            # Typed REST API client & TypeScript interfaces
    └── utils.ts                          # Classname merging utilities (clsx + tailwind-merge)
```

---

## Getting Started (Local Development)

### Prerequisites
- **Node.js** `v20+` and `npm`
- A running instance of **[requestlens-backend](https://github.com/faruqolatoyese/requestlens-backend)** (defaults to `http://localhost:5000`) or connection to the production backend API.

### 1. Clone the Repository
```bash
git clone https://github.com/faruqolatoyese/requestlens-frontend.git
cd requestlens-frontend
```

### 2. Install Dependencies
```bash
npm install
```

### 3. Configure Runtime Environment Variables
Configure the following environment variables in your development environment before starting the server:

| Variable Name | Required | Description | Example Value |
| :--- | :---: | :--- | :--- |
| `NEXT_PUBLIC_API_URL` | Yes | Base URL of the RequestLens Backend REST API (also used by `useSocket` to derive the WebSocket host) | `http://localhost:5000/api` |
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | Yes | Google OAuth 2.0 Web Client ID for "Continue with Google" authentication | `your-google-client-id.apps.googleusercontent.com` |

### 4. Start the Development Server
```bash
npm run dev
```
Open [http://localhost:3000](http://localhost:3000) in your browser.

### 5. Build for Production
```bash
npm run build
npm run start
```

---

## Quick Usage Walkthrough

1. **Sign Up / Log In:** Create an account at `/register` or sign in with Google.
2. **Create a Project:** On `/dashboard`, click **Create Project** and copy your generated **Project API Key**.
3. **Register an Endpoint:** Navigate to your project's **Endpoints** tab (`/dashboard/[projectId]/endpoints`) and register a route (e.g., `POST /api/v1/checkout`).
4. **Simulate or Send Live Traffic:**
   - **Option A (In-App Simulator):** On the **Overview** tab, use the **Log Ingestor Simulator** to select your endpoint, pick a status code (`200` or `500`) and latency, and click **Fire Test Request**. Watch the charts and logs update live via WebSockets.
   - **Option B (Backend Middleware):** Copy the **cURL**, **Express**, or **FastAPI** snippet from the **SDK & Integration** panel on the Overview tab and drop it into your application.
5. **Configure Alert Rules:** Open the **Settings** tab (`/dashboard/[projectId]/settings`), add a **5xx Error Alert Rule**, and test it by firing `500 Server Error` requests in the simulator.

---

## License

This project is licensed under the MIT / ISC License.
