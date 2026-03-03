<p align="center">
  <img src="fello cover.png" alt="Fello — Live With Peers" width="600" />
</p>



<h1 align="center">Fello — Live With Peers</h1>

<p align="center">
  <strong>A roommate matching REST API built for interns.</strong><br/>
  Matches compatible housemates based on budget, schedule, lifestyle, and personality.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-archived-lightgrey" alt="Status: Archived" />
  <img src="https://img.shields.io/badge/java-21-orange" alt="Java 21" />
  <img src="https://img.shields.io/badge/spring%20boot-3.x-brightgreen" alt="Spring Boot" />
  <img src="https://img.shields.io/badge/postgresql-16-blue" alt="PostgreSQL 16" />
  <img src="https://img.shields.io/badge/branch%20coverage-~77.5%25-yellow" alt="Coverage" />
</p>

---

> **Note:** This project is archived and no longer under active development. It was built as a portfolio project to demonstrate backend API design, algorithm engineering, and security-first thinking with Spring Boot.

---

## The Problem

Every summer, thousands of interns face the same housing nightmare: overpriced solo apartments, sketchy Craigslist posts, or chaotic Facebook groups where you're messaging hundreds of strangers hoping someone is compatible. Traditional roommate finders assume year-long leases and don't account for the unique constraints of a 3-month internship like budget sensitivity, overlapping start/end dates, and office proximity.

## The Solution

Fello is a backend API that matches interns with compatible roommates using a **weighted 5-factor scoring algorithm** that evaluates what actually matters for short-term shared housing: budget alignment, internship timeline overlap, office proximity, lifestyle compatibility, and social interests. Users authenticate via Google OAuth, fill out a profile, and the API returns ranked matches they can accept, reject, or block.

---

## Architecture

```
┌─────────────────┐
│   Any Client    │
│  (HTTP + JSON)  │
└────────┬────────┘
         │
┌────────▼──────────────────────────────────────────┐
│              Spring Boot REST API                  │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │  Security Layer                               │ │
│  │  • Google OAuth2 login                        │ │
│  │  • JWT (HttpOnly cookies + Bearer header)     │ │
│  │  • IDOR prevention on all endpoints           │ │
│  │  • Stateless session management               │ │
│  └──────────────────┬───────────────────────────┘ │
│                     │                              │
│  ┌──────────────────▼───────────────────────────┐ │
│  │  Controller Layer                             │ │
│  │  • AccountController                          │ │
│  │  • MatchController                            │ │
│  │  • AuthController (logout)                    │ │
│  └──────────────────┬───────────────────────────┘ │
│                     │                              │
│  ┌──────────────────▼───────────────────────────┐ │
│  │  Service Layer                                │ │
│  │  • AccountService (CRUD, profile validation)  │ │
│  │  • MatchService (state machine, cooldowns)    │ │
│  │  • MatchEngine (scoring algorithm)            │ │
│  │  • JwtService (token generation/validation)   │ │
│  └──────────────────┬───────────────────────────┘ │
│                     │                              │
│  ┌──────────────────▼───────────────────────────┐ │
│  │  Repository Layer (Spring Data JPA)           │ │
│  │  • AccountRepository                          │ │
│  │  • MatchRepository                            │ │
│  └──────────────────┬───────────────────────────┘ │
└─────────────────────┼────────────────────────────-┘
                      │
              ┌───────▼───────┐
              │  PostgreSQL   │
              └───────────────┘
```

---

## Matching Algorithm

The core of Fello is a **weighted multi-factor scoring engine** that computes pairwise compatibility between users. The algorithm produces a score from 0.0 to ~1.2 (social traits provide additive bonuses beyond 1.0).

### Factor Weights

| Factor | Weight | Description |
|--------|--------|-------------|
| **Budget** | 35% | `1 - abs(A - B) / max(A, B)` — Closer budgets score higher |
| **Timeline** | 25% | `overlapping_days / max(term_A, term_B)` — Rewards date alignment |
| **Proximity** | 20% | `1 - haversine(A, B) / 35mi` — Haversine distance between offices |
| **Lifestyle** | 20% | Average of cleanliness, sleep cycle, and guest policy similarity |
| **Social** | Additive | Flat bonus per overlapping social trait (capped at lifestyle weight) |

### Dealbreakers (Automatic Disqualification)

The algorithm returns a score of **0.0** (incompatible) if any of the following conditions are met:

- **Gender preference mismatch**: If either user requests same-gender roommates and the other doesn't match
- **Office distance > 35 miles**: Offices too far apart for reasonable shared housing
- **Timeline overlap < 30 days**: Insufficient overlap to justify co-housing

### Scoring Functions

- **`calculateHaversineDistance()`** : Computes great-circle distance between two lat/lon coordinates using the Haversine formula (Earth radius = 3,958.8 mi)
- **`calculateBudgetScore()`** : Normalized difference with bounds checking ($0–$10,000 range, rejects negatives and overflow values)
- **`calculateTimelineScore()`** : Computes day-level overlap between two date ranges, requires minimum 30-day overlap
- **`calculateLifestyleScore()`** : Averages three ordinal-based similarity scores (cleanliness 1–5, sleep cycle enum, guest policy enum)
- **`calculateSocialScore()`** : Set intersection of social trait enums with a flat additive bonus per shared trait
- **`confirmGenderPreferences()`** : Boolean dealbreaker check before scoring begins

### Profile Fields

```
High Priority:     officeLatitude, officeLongitude, maxBudget, termStart, termEnd
Lifestyle:         cleanliness (1-5), sleepCycle (NIGHT_OWL|FLEXIBLE|EARLY_BIRD),
                   guestPolicy (QUIET_HOUSE|INDIFFERENT|SOCIAL_HOUSE)
Social:            socialTraitsSet (CITY_EXPLORER|OUTDOORSY|GAMER|GYMGOER|COOK|INTROVERT|EXTROVERT)
Dealbreakers:      gender (MALE|FEMALE|NON_BINARY), genderPreference (ANY|SAME)
```

Null fields contribute a score of 0 for their category : this incentivizes profile completion without requiring it.

---

## Match State Machine

Matches follow a defined state machine with safeguards:

```
           A requests B
               │
               ▼
           ┌────────┐
           │PENDING │◄──────────────────────────┐
           └───┬────┘                            │
               │                                 │
       ┌───────┼───────┐                         │
       ▼       ▼       ▼                         │
  ┌────────┐ ┌────────┐ ┌───────┐          (after 7-day
  │ACCEPTED│ │REJECTED│ │BLOCKED│           cooldown)
  └────────┘ └───┬────┘ └───────┘
                  │
                  │ 7-day cooldown
                  ▼
              Can re-match
```

- **PENDING → ACCEPTED**: Both users must independently request the match (mutual opt-in)
- **PENDING → REJECTED**: Either user can reject; triggers a 7-day cooldown before re-matching
- **PENDING → BLOCKED**: Either user can block; only the blocker can unblock
- **Duplicate prevention**: Cannot create a second PENDING match while one exists
- **Cooldown enforcement**: Rejected matches cannot be re-initiated for 7 days

---

## Security

### Authentication

- **Google OAuth2** login flow with custom `OauthSuccessHandler`
- **JWT tokens** delivered via `fello_token` HttpOnly/Secure cookie (browser path) or `Authorization: Bearer` header (API path)
- **Stateless sessions** : no server-side session storage (`SessionCreationPolicy.STATELESS`)
- **HMAC-SHA256** token signing via JJWT 0.13.0

### Authorization & IDOR Prevention

Every endpoint that operates on user data extracts the authenticated user's email from the JWT principal, not from request body parameters. This makes it architecturally impossible for User A to act as User B:

- `POST /api/match/two` : Verifies `authentication.getPrincipal() == request.email1()`
- `PUT /api/account/update` : Applies updates to the principal's account only
- `PATCH /api/match/block` : Same principal verification
- `GET /api/match` : Returns only the principal's matches

### Tested Attack Vectors

The security test suite (`SecurityIntegrityTests`) validates:

- Signature-invalid JWTs (wrong signing key) → 401
- Expired JWTs (via header and cookie) → 401
- Missing `Authorization` header → 401
- Malformed header (no `Bearer` prefix) → 401
- Garbage JWT strings → 401
- Cross-user match initiation (IDOR) → 4xx
- Cross-user blocking (IDOR) → 4xx
- Cross-user score viewing (IDOR) → 4xx
- Cross-user profile update isolation verified

---

## API Endpoints

### Account

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `GET` | `/api/account/login` | Public | Redirects to Google OAuth2 |
| `POST` | `/api/auth/logout` | Public | Clears `fello_token` cookie (`Max-Age=0`) |
| `GET` | `/api/account/me` | JWT | Returns authenticated user's profile |
| `PUT` | `/api/account/update` | JWT | Updates authenticated user's profile |

### Matching

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/api/match/two` | JWT | Initiate a match request (or accept if reciprocal) |
| `POST` | `/api/match/twoScore` | JWT | Get compatibility score between two users |
| `GET` | `/api/match` | JWT | List all matches for authenticated user |
| `PATCH` | `/api/match/reject` | JWT | Reject a pending match (starts 7-day cooldown) |
| `PATCH` | `/api/match/block` | JWT | Block a user |
| `PATCH` | `/api/match/unblock` | JWT | Unblock a user (only by original blocker) |

---

## Testing

### Test Coverage (~77.5% branch coverage)

The test suite is organized into four layers:

**Unit Tests (`MatchEngineTests`)** : Pure algorithm validation with no Spring context:
- Budget scoring: identical budgets, different budgets, zero/negative budgets, overflow protection
- Timeline scoring: perfect overlap, partial overlap, no overlap, sub-30-day overlap rejection
- Proximity scoring: same location, 25+ mile rejection, Haversine accuracy (NYC↔Philadelphia within 2mi tolerance)
- Lifestyle scoring: identical traits, opposite traits, mixed traits, null field handling
- Social scoring: full overlap, no overlap, partial overlap
- Gender dealbreaker logic: same-gender-only enforcement, null gender handling

**Controller Tests (`MatchControllerTests`, `AccountControllerTests`)** : MockMvc-based HTTP layer tests:
- Correct HTTP status codes for all success/error paths
- Request validation (missing fields, malformed emails)
- JWT principal mismatch detection
- 7-day cooldown enforcement
- Duplicate pending match prevention
- Block/unblock state transitions

**Integration Tests (`MatchIntegrationTests`)** : Full Spring context with real database:
- End-to-end match creation with real `MatchEngine` scoring
- Score variation based on profile differences
- Reject → cooldown enforcement at service + repository level
- Unique constraint enforcement at DB level
- Account deletion behavior (orphaned matches documented)

**Security Integrity Tests (`SecurityIntegrityTests`)** : Dedicated security boundary validation:
- All JWT attack vectors (expired, wrong key, garbage, missing)
- IDOR prevention across match, block, score, and profile update endpoints
- Cookie-based auth path alongside header-based auth

**Edge Case Tests (`MatchEdgeCaseTests`)** : Boundary condition handling:
- Fully null profiles (all scoring fields null) don't crash
- Empty/whitespace names handled gracefully

### Running Tests

```bash
# Run all tests
./mvnw test

# Run with coverage report
./mvnw test jacoco:report
# Report generated at: target/site/jacoco/index.html
```

---

## Local Development Setup

### Prerequisites
- Java 21+
- PostgreSQL 16+
- Maven 3.9+
- Google OAuth2 credentials (Google Cloud Console)

### Installation

```bash
# Clone repository
git clone https://github.com/packageIncoming/fello
cd fello

# Set environment variables
export FELLO_JWT_SECRET=your_jwt_secret_key
export SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/fello
export SPRING_DATASOURCE_USERNAME=postgres
export SPRING_DATASOURCE_PASSWORD=your_password
export SPRING_SECURITY_OAUTH2_CLIENT_REGISTRATION_GOOGLE_CLIENT_ID=your_google_client_id
export SPRING_SECURITY_OAUTH2_CLIENT_REGISTRATION_GOOGLE_CLIENT_SECRET=your_google_client_secret

# Start PostgreSQL
docker run -d \
  -e POSTGRES_DB=fello \
  -e POSTGRES_PASSWORD=your_password \
  -p 5432:5432 \
  postgres:16

# Build and run
./mvnw clean install
./mvnw spring-boot:run
```

API will be available at `http://localhost:8080`

---

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| **Language** | Java | 21 |
| **Framework** | Spring Boot | 3.x |
| **Security** | Spring Security 6 + JJWT | 0.13.0 |
| **Database** | PostgreSQL | 16 |
| **ORM** | Hibernate / Spring Data JPA | — |
| **Build** | Maven | 3.9+ |
| **Testing** | JUnit 5, Mockito, MockMvc, @SpringBootTest | — |
| **Other** | Lombok | — |

---

## Project Structure

```
fello/
├── src/main/java/com/fello/fello/
│   ├── Account/
│   │   ├── Account.java              # Entity with profile fields & validation
│   │   ├── AccountController.java     # REST endpoints for account CRUD
│   │   ├── AccountService.java        # Business logic
│   │   ├── AccountRepository.java     # Spring Data JPA
│   │   ├── DTO/                       # Request/response DTOs
│   │   ├── Enums/                     # Gender, SleepCycle, GuestPolicy, SocialTraits
│   │   ├── Exceptions/                # UserNotFoundException, etc.
│   │   └── Validators/                # @ValidDateRange custom annotation
│   ├── Auth/
│   │   ├── SecurityConfig.java        # Filter chain, endpoint permissions
│   │   ├── JwtAuthenticationFilter.java
│   │   ├── JwtService.java            # Token generation & validation
│   │   └── OauthSuccessHandler.java   # Post-OAuth JWT cookie writer
│   ├── Matching/
│   │   ├── Match.java                 # Entity (fromId, toId, score, status)
│   │   ├── MatchController.java       # REST endpoints for matching
│   │   ├── MatchService.java          # State machine, cooldowns, business rules
│   │   ├── MatchEngine.java           # 5-factor scoring algorithm
│   │   ├── MatchRepository.java       # Custom query: findMatchByPair
│   │   ├── DTO/                       # MatchTwoRequest, MatchTwoObjectDto, etc.
│   │   └── Exceptions/                # MatchException
│   └── Utils/
│       └── GlobalExceptionHandler.java
├── src/test/java/com/fello/fello/
│   ├── Account/
│   │   └── AccountControllerTests.java
│   ├── Auth/
│   │   └── SecurityIntegrityTests.java
│   ├── Matching/
│   │   ├── MatchControllerTests.java
│   │   ├── MatchServiceTests.java
│   │   ├── MatchIntegrationTests.java
│   │   ├── MatchEngineTests.java
│   │   └── MatchEdgeCaseTests.java
│   ├── E2E/
│   │   └── MatchE2ETests.java         # Stubs for future E2E tests
│   └── Utils/
│       └── TestHelper.java
├── pom.xml
└── README.md
```

---

## What Was Not Built (and Why)

This project was intentionally scoped as a **backend API** to demonstrate engineering depth over surface-level feature breadth:

- **1:1 Chat** : Planned but deprioritized. Would have required WebSocket infrastructure (Spring WebSocket + STOMP) and a Chat entity linked to ACCEPTED matches.
- **Batches (group housing)** : Designed conceptually (groups of 3–5 interns pooling budgets) but not implemented.
- **Email/university verification** : OAuth handles identity; .edu domain enforcement was a planned addition.
- **Production deployment** : The API runs locally; no cloud deployment was pursued for this project.
- **Frontend** : No frontend was built. This is a backend API project. Any client (web, mobile, CLI) could consume these endpoints.

---

## What I Learned

**Algorithm design under constraints**: Building a matching algorithm that handles null fields gracefully, enforces dealbreakers before scoring, and produces intuitive results required careful thought about edge cases. The biggest insight: null fields should penalize (score 0 for that factor), not crash or produce misleading high scores.

**Security as architecture, not a layer**: IDOR prevention isn't just input validation, it's designing endpoints so that the authenticated principal is the *only* source of user identity. Extracting the user from the JWT principal rather than trusting request body parameters makes an entire class of vulnerabilities structurally impossible.

**Test-driven confidence**: Writing security tests that explicitly attempt JWT forgery, IDOR attacks, and state machine violations gave me confidence the system actually works and not just that the happy path returns 200.

**State machine complexity**: The match lifecycle (PENDING → ACCEPTED/REJECTED/BLOCKED with cooldowns and mutual opt-in) was more complex than expected. Edge cases like "what if both users block each other simultaneously" or "what if a blocked user tries to re-match" required careful service-layer logic.

---

## Author

**Mert Isik**
- LinkedIn: [linkedin.com/in/mert-c-isik](https://linkedin.com/in/mert-c-isik)
- GitHub: [@packageIncoming](https://github.com/packageIncoming)
- Email: mertisik329@gmail.com

---

*Built to demonstrate backend API design, algorithm engineering, and security-first development with Spring Boot.*
