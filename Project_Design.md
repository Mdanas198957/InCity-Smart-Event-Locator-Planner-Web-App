# InCity – Smart Event Locator & Planner (India Edition)

## 1. Folder Structure (Monorepo)

```text
incity-smart-event-locator/
├── client/                     # Frontend (React + Vite/Next.js)
│   ├── public/                 # Static assets, map markers, favicons
│   ├── src/
│   │   ├── assets/             # Images, SVGs, animation files
│   │   ├── components/         # Reusable UI components
│   │   │   ├── common/         # Buttons, Inputs, Modals
│   │   │   ├── layout/         # Navbar, Sidebar, Footer
│   │   │   ├── map/            # Mapbox integration, Heatmap layers
│   │   │   └── events/         # Event cards, Stack conflict resolver UI
│   │   ├── hooks/              # Custom React hooks (useMap, useCrowdData)
│   │   ├── pages/              # Route components (Home, Dashboard, EventDetail)
│   │   ├── services/           # API hooks (React Query setup)
│   │   ├── store/              # Zustand/Redux state slices
│   │   ├── styles/             # Tailwind config, global CSS
│   │   ├── types/              # TypeScript interfaces
│   │   └── utils/              # Helper functions, formatters
│   ├── package.json
│   └── tailwind.config.js
├── server/                     # Backend (Node.js + Express)
│   ├── prisma/                 # Prisma schema & migrations
│   ├── src/
│   │   ├── config/             # DB, Redis, Env setups
│   │   ├── controllers/        # Request handlers
│   │   ├── middlewares/        # Auth, Error handling, Rate limiting
│   │   ├── routes/             # Express routes
│   │   ├── services/           # Business logic (event booking, conflicts)
│   │   ├── sockets/            # WebSockets for live crowd updates
│   │   ├── utils/              # JWT, hashing, geohashes
│   │   └── index.ts            # Entry point
│   └── package.json
├── ai-engine/                  # Python Microservice (FastAPI)
│   ├── app/
│   │   ├── api/                # FastAPI routes
│   │   ├── models/             # ML Models (Crowd prediction)
│   │   ├── services/           # Inference logic, conflict resolution algorithms
│   │   └── main.py             # FastAPI entry
│   ├── requirements.txt
│   └── Dockerfile
├── docker-compose.yml          # Local development stack (Postgres, Redis, Services)
└── README.md
```

## 2. Frontend Components

- **`HeroSection`**: Animated tagline "Plan Smart. Avoid Crowds. Experience Better." using Framer Motion. Features a dark-glass search bar (Event, City, District, Date).
- **`InteractiveMap`**: The core map dashboard using Mapbox GL JS. Handles layers for districts, pins for events, and a heatmap layer for crowd density.
- **`EventStackResolver`**: A specialized UI component that slides up when overlapping events are detected. Displays conflicting events as a "stack of glass cards", letting the user swipe/scroll to see Priority Ranking, crowd density, and alternative time slots.
- **`DensityMeter`**: A live meter (Green, Orange, Red) showing current or predicted crowd levels at an event venue.
- **`EventCard`**: Premium card component using glassmorphism. Shows event image, title, organizer verified badge, rating, and quick "Book/Save" buttons.
- **`BookingModal`**: Secure modal for ticket generation and payment with privacy toggles (e.g., "Join Anonymously").
- **`FilterSidebar`**: Collapsible sidebar for toggling Free/Paid, Public/Private, and Event Categories.

## 3. Backend API Routes

**Auth & Users**
- `POST /api/auth/register` (User/Organizer signup)
- `POST /api/auth/login` (JWT generation)
- `GET /api/users/profile` (Get user preferences & history)

**Events**
- `GET /api/events` (List events with geo-filters & category filters)
- `POST /api/events` (Create new event - Organizer only)
- `GET /api/events/:id` (Event details)
- `GET /api/events/conflicts` (Detect overlapping events in a specific radius & time)

**Bookings**
- `POST /api/bookings` (Book an event, generate QR)
- `GET /api/bookings/my-tickets`

**Maps & Crowd (WebSockets & REST)**
- `GET /api/crowd/density?lat=x&lng=y` (Get current/predicted density from Redis/AI service)
- `WS /ws/crowd-updates` (Real-time pushing of density changes to clients looking at the map)

**AI Service (FastAPI)**
- `POST /ai/predict-crowd` (Takes location, time, historical data, returns density score)
- `POST /ai/resolve-conflict` (Takes competing events and user profile, returns ranked recommendations)

## 4. Database Schema (Prisma / PostgreSQL)

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id             String    @id @default(uuid())
  name           String
  email          String    @unique
  passwordHash   String
  role           Role      @default(USER) // USER, ORGANIZER, ADMIN
  preferences    Json?     // Interests, preferred crowd levels
  privacyMode    Boolean   @default(false)
  bookings       Booking[]
  reviews        Review[]
  createdAt      DateTime  @default(now())
}

model Event {
  id             String    @id @default(uuid())
  title          String
  description    String
  latitude       Float
  longitude      Float
  address        String
  city           String
  district       String
  state          String
  startTime      DateTime
  endTime        DateTime
  isPrivate      Boolean   @default(false)
  ticketPrice    Float     @default(0.0)
  maxCapacity    Int
  organizerId    String
  organizer      User      @relation(fields: [organizerId], references: [id])
  bookings       Booking[]
  reviews        Review[]
}

model Booking {
  id             String    @id @default(uuid())
  userId         String
  eventId        String
  qrData         String    @unique
  status         String    // CONFIRMED, CANCELLED
  user           User      @relation(fields: [userId], references: [id])
  event          Event     @relation(fields: [eventId], references: [id])
  createdAt      DateTime  @default(now())
}

model Review {
  id             String    @id @default(uuid())
  rating         Int       // 1-5
  comment        String?
  userId         String
  eventId        String
  user           User      @relation(fields: [userId], references: [id])
  event          Event     @relation(fields: [eventId], references: [id])
}

model CrowdAnalytics {
  id             String    @id @default(uuid())
  locationGeoHash String   
  timestamp      DateTime
  densityScore   Float     // 0.0 to 1.0 (Green to Red)
  source         String    // Historical, API, BookingCount
}

enum Role {
  USER
  ORGANIZER
  ADMIN
}
```

## 5. Crowd Density Algorithm Logic

The Crowd Density Algorithm combines deterministic data (bookings) and probabilistic data (historical/API) to generate a dynamic Heatmap score `D(t)`.

**Inputs:**
1. **$E_{bookings}$**: Active ticket bookings/registrations for the location API.
2. **$H_{historical}$**: Historical crowd data for that district/pin-code at that time/day.
3. **$T_{traffic}$**: Live traffic multiplier from Google Maps Traffic API or Mapbox.
4. **$V_{capacity}$**: Max capacity of the venue.

**Step 1: Base Density Calculation**
`Base_Score = (E_bookings / V_capacity) * 100`

**Step 2: Time & Traffic Multipliers**
- If peak hours (e.g., 6 PM - 9 PM) or high traffic index: `Traffic_Multiplier = 1.3`
- If festive season or weekend: `Date_Multiplier = 1.2`

**Step 3: AI Prediction (FastAPI)**
The Python service runs a predictive model (e.g., Random Forest or lightweight XGBoost) trained on local Indian conflict scenarios (e.g., local market days, political rallies).
`Predicted_Surge = AI_Model.predict(location, time, type_of_event)`

**Step 4: Final Density Score Normalization**
`D(t) = (Base_Score * Traffic_Multiplier * Date_Multiplier) + Predicted_Surge`
Normalize `D(t)` to a 0-100 scale:
- **0 - 40**: Green (Safe, fluid movement)
- **41 - 75**: Orange (Moderate crowd, slight delays)
- **76 - 100**: Red (High crowd, recommend alternatives)

## 6. UI Mockup Description

**Theme:** 
- **Colors:** Royal Navy (`#0A192F`) & Deep Black backgrounds, with Metallic Gold (`#FFD700`) and Neon Cyan accents for interactable elements.
- **Font:** 'Inter' for UI legibility, 'Outfit' for premium headings.
- **Style:** Glassmorphism with deep blurs and soft drop shadows for elevated cards.

**Landing Page:**
- **Header:** Transparent glass navbar with "InCity" gold logo, "Explore", "Host", "Login".
- **Hero:** A high-quality dark-themed ambient video background of an Indian city skyline, overlaid with a frosted glass search panel.
- **Below Hero:** "Live India Map" section showing glowing dots where events are happening.

**Interactive Map Dashboard:**
- Map takes up 100% of the screen height (viewport).
- Floating dark-glass panels on the left over the map.
- Distinct heatmap colors on the map: Soft green, pulsing orange, and glowing red for severe congestion zones.
- Clicking a red zone ripples outwards and a sidebar slides in from the right: *"Warning: 2 events conflicting in Indiranagar. View Alternatives."*

**Event Stack UI (Conflict Resolver):**
- When 3 events overlap on Saturday at 6 PM in the same district, the UI shows a "stacked cards" 3D animation.
- **Card 1 (Top):** "Tech Meetup" (90% Match to user, Density: Orange)
- **Card 2 (Middle):** "Local Food Fest" (Density: Red zone, 2km away)
- Action buttons glow: "Reschedule for 8 PM" or "Book Best Match".

## 7. Deployment Steps

**Phase 1: CI/CD Pipeline Setup**
- Set up GitHub Actions for automated linting (ESLint), testing, and formatting.

**Phase 2: Database & Backend (Railway/Render)**
- Provision a PostgreSQL database with PostGIS enabled on Railway/Render.
- Provision a Redis instance for caching.
- Deploy the Node.js Express Server. Add Environment variables (`DATABASE_URL`, `JWT_SECRET`, `MAPBOX_KEY`).
- Run Prisma migrations (`npx prisma migrate deploy`).

**Phase 3: AI Microservice (Render/AWS)**
- Dockerize the FastAPI Python app.
- Deploy the Docker container internally connected to the Node.js backend.

**Phase 4: Frontend (Vercel)**
- Connect the GitHub repo to Vercel for the `client/` directory.
- Configure build command (`npm run build`) and output directory (`dist`).
- Add Environment variables (`VITE_API_BASE_URL`, `VITE_MAPBOX_TOKEN`).
- Deploy. Vercel automatically deploys to its global Edge Network.

**Phase 5: Performance & CDN (Cloudflare)**
- Link custom domain (e.g., `incity.in`).
- Route traffic through Cloudflare for DDoS protection and asset caching—crucial for handling traffic spikes during major Indian festivals or events.
