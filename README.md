<img width="7680" height="4320" alt="Fello.sh cover" src="https://github.com/packageIncoming/fello-design-doc/blob/main/cover.png" />

# Fello.sh: The Internship Roommate Finder
### Designed by Mert Isik

# Background

- Many college students are given internship offers in locations that they do not live in. These interns need a place to live. They usually have the following options:
	1. The company provides housing assistance either through a stipend or homes to interns. This is very rare and often only for competitive companies.
	2. Interns can rent an entire apartment/temporary living themselves. This is very expensive and hard to find since internships are typically only 3 months. Alternatives like Airbnb are also very expensive
	3. Interns can rent with roommates which is the most common and cost-effective approach.
- However, to rent with roommates, a person needs to have roommates. While there are existing apps that match potential roommates together, there’s no specific app/platform that seems to target the niche of interns who want to rent living spaces with other interns. 

## What does Fello do? What can students use it for?

- **In short, Fello is a platform for interns to find fellow interns to rent temporary living with. Fello focuses on connecting interns based on location, interests, duration of stay, and more.** 
- Fello provides security by only allowing Verified Users to message and see roommate profiles. Fello verifies students through their email (**OAuth2**) 
- Fello lets interns create a profile page with **Traits**. These Traits are used to match users. Further details below on the MatchEngine.
- Fello allows interns to look for other interns in the area they designate through a **search** AND **match** feature (think LinkedIn  + Tinder). 
- Fello also allows users to **message** each other, either 1:1 or as a group. 
- Fello allows users to create groups called **Batches**. Batches come with a group chat and the following:
	- **Democratic invitation**: When a member proposes adding another person to a Batch, ALL current members MUST accept the proposal.
	- **Silent veto**: Any member of a batch can silently veto a proposal. This veto is anonymous and immediately cancels the invite. 
	- **Democratic kick**: Members can anonymously vote to remove another member from the Batch. 
## Core Features for MVP:

The core functionality needed for the MVP of Fello is as follows:
1. Users can create an account. Accounts must be created and verified with a .edu email before the full functionality of Fello is opened.
2. Users can customize their profile with personal characteristics to find the best match roommates.
3. Users can find potential roommates through the **FelloMatch** feature or by **searching in their area**
4. Users can **message each other 1:1 when finding matching roommates**
5. Users can create **Batches**, groups of multiple users. Batches have the following functionality:
	- Create, Update, Delete batch
	- **Vote-in members**: All current members of the Batch must approve of the invite. 
	- **Veto member proposals**: Any user can veto the invitation anonymously, immediately shutting down the proposal
	- **Anonymous kick**: Users can anonymously vote to kick out another member
	- **Group chats**: Members within a batch can text each other in a shared messaging chat.

---

# Account: User Traits

## User Traits
Users can set traits for their profile. These traits are used by the **MatchEngine** to turn users into vectors to compare and find highest similarity (least difference/best fit)

### 1. Core Logistics (High Weight Traits)
Users can set important information about their internship and budget. These are the highest priority aspects of their profile.

- **Office Location: (Lat/Long)** - Used to calculate the "Batch Commute Center" or distance between two work locations (important for matching engine). 
- **Budget Max (Per Person): ($)** - What the individual is willing to pay for their portion of the rent. Used to calculate total "Batch Purchasing Power" and match users by budgets.
- **Timeline: (Start Date - End Date)** - Crucial for aligning leases and internship durations.
### 2. Lifestyle Heuristics (Medium Weight)
Users can either select from a range (cleanliness) or a category that best fits their lifestyle

- **Cleanliness:** (Scale 1-5) 1: "Lived in" | 5: "Surgical/Spotless".
- **Sleep Cycle:** (Early Bird | Night Owl | Irregular).
- **WFH Frequency:** (Office-bound | Hybrid | Remote).
- **Guest Policy:** (Quiet House | Social House).

### 3. Social & Friendship (Low Weight/Ties)
Users can set social/friendship characteristics which can help break ties or find better fit. Not weighed as heavily as #1 and #2 but can assist in nuanced matching.

- **The "Weekend Goal"**: (City Explorer | Hiker/Outdoors | Gamer/Homebody).
- **Academic Major**: (SWE, UX, PM, Finance, etc.).
- **University Name**: (Used to find "School Alums" in the same city).
### 4. Safety & Privacy (Max-Priority/Deal-breakers)
Users can set Max-Priority characteristics/preferences which will either allow/disallow a match.

- **Gender Identity:** (User Defined).
- **Gender Preference**: (Same-gender only | Open to all).

---
# Technical Details: Fello MatchEngine v1.0

## 1. The Core Equation

The **Total Match Score (**$S_{total}$**)** is the weighted sum of normalized sub-scores, multiplied by hard constraints (Filters).

$$S_{total} = (\sum w_i \cdot s_i) \times \prod C_j$$

Where:

- $w_i$: Weight of the category.
- $s_i$: Normalized score [0, 1].
- $C_j$: Hard constraint [0 or 1].
## 2. Sub-Score Vectorization ($s_i$)

### 2.1 Commute Vector ($s_{commute}$)

- **Logic:** Distance between Office A and Office B.
- **Normalization:** $1 - \min(1, \frac{dist(A, B)}{max\_dist})$
- **Target:** People working in the same neighborhood (e.g., SoMa) should score ~0.9.
### 2.2 Budget Alignment ($s_{budget}$)

- **Logic:** High-budget users shouldn't be matched with low-budget users to avoid friction during house hunting.
- **Calculation:** $1 - \frac{|Budget_A - Budget_B|}{\max(Budget_A, Budget_B)}$
### 2.3 Timeline Overlap ($s_{time}$)

- **Logic:** If Person A is there May–July and Person B is there June–August, they only have 50% overlap.
- **Calculation:** $\frac{Days\_Overlap}{Max(Days\_A, Days\_B)}$
- _Note: If Overlap < 30 days,_ $C_{timeline} = 0$ _(Hard fail)._

### 2.4 Lifestyle Distance ($s_{lifestyle}$)

- **Logic:** Integer difference of Enum values.
- **Features:** Cleanliness [1-5], Sleep [1-3], Guest Policy [1-3].
- **Calculation:** For each feature $f$: $score_f = 1 - \frac{|f_A - f_B|}{f_{max} - f_{min}}$.
- **Average:** $s_{lifestyle}$ is the average of these sub-scores.

## 3. Bonuses & Hard Constraints ($C_j$)

### 3.1 Hard Constraints (Multipliers)

- **Gender Match (**$C_{gender}$**):** If User A requires "Same-Gender Only" and User B is a different gender, the multiplier is **0**. Otherwise, **1**.
- **Verification (**$C_{verif}$**):** Optional filter. If "Verified Only" is checked, unverified users get a **0**.

### 3.2 The "Alumni & Major" Bonus (Additive)

- **University Bonus:** If `Univ_A == Univ_B`, add **+0.05** to total score.
- **Major Bonus:** * Create a Map: `{"CS": "TECH", "EE": "TECH", "FIN": "BUS"}`.
    - If `Cluster_A == Cluster_B`, add **+0.03**.

## 4. MVP Weights ($w_i$)

For an MVP, the following will be the weights for each category:

|               |                  |                                                 |
| ------------- | ---------------- | ----------------------------------------------- |
| **Category**  | **Weight (wi​)** | **Rationale**                                   |
| **Budget**    | 0.35             | The #1 reason groups fail.                      |
| **Timeline**  | 0.25             | Logistics must align first.                     |
| **Commute**   | 0.20             | Proximity to office drives neighborhood choice. |
| **Lifestyle** | 0.20             | Daily habits (Cleanliness/Sleep).               |

For a future full release, it is planned to allow users to customize the Importance (weight) of each category.
-  Ex. After account creation, user is prompted "How important is a matching budget to you" and the user selects a number between 1 and 5. This is repeated for all 4 categories, the entered scores are summed and used to calculate the weight for each category as (Score / Total Score).

---
# Finding Potential Matches using MatchEngine (Fello Find API)

## The Goal
We want users to be able to sift through potential matches. For the MVP, this will be done by getting an initial group of at most ~200 high-quality matches.

## The Flow
1. **User sends a request to an API endpoint**
2. **The service layer first begins a 'coarse filter' over other users: Only other users whose location, budget, and gender fit the original user's preferences pass the coarse filter. This is limited to the first 200 users.**
	1. This prevents needless checks like comparing a user in Austin to a user in San Francisco.
3. **The MatchEngine is used to calculate the match between the original user and each potential candidate. The top 20 from this search are returned to the user, sorted by match score.**

## Deeper Details

### 1. The Performance Problem

Doing a full vector similarity check on every request is $O(N)$. With 10,000 users, that's 10,000 complex math operations per "Refresh." This causes high CPU load and slow API response times.

### 2. The Solution: Tiered Pruning

We divide the matching process into three stages, moving from "Cheap & Fast" to "Expensive & Precise."

#### Stage 1: SQL Pruning 

We use the database to exclude users who could _never_ be a match. This reduces $N$ from thousands to the target (200).

**SQL Filters to apply in the `WHERE` clause:**

- **City ID:** Must be in the same city (Hard requirement).
- **Gender Preference:** Filter out anyone who doesn't meet the user's `Same-Gender` constraint.
- **Budget Floor/Ceiling:** Only pull users within $\pm 30\%$ of the seeker's budget.
- **Timeline Overlap:** Only pull users whose stay overlaps by at least 30 days.
    

```
-- Example Pruning Query
SELECT * FROM user_profiles 
WHERE city_id = :myCity
  AND budget BETWEEN :minBudget AND :maxBudget
  AND id != :myId
  AND id NOT IN (SELECT target_id FROM matches WHERE initiator_id = :myId) -- Skip already matched/rejected
LIMIT 200;
```

#### Stage 2: Java Ranking

Once the DB returns the 200 "closest" candidates, we pass them into the `MatchEngine` in Java.

1. Vectorize the 200 candidates.
2. Calculate the $S_{total}$ using the weighted formula.
3. Sort the list in memory.
4. Return the top 20 to the user.

#### Stage 3: Result Caching

We don't need to re-calculate matches every time the user refreshes the page.

- **Strategy:** Store the IDs of the Top 20 matches in **Redis** with a Time-to-Live (TTL) of 30 minutes.
- **Invalidation:** If the user updates their own profile (e.g., changes budget), we clear the Redis cache.

### 3. The API Design

#### Endpoint: `GET /api/v1/matches/discover`

**Request Headers:** `Authorization: Bearer <token>` **Query Params:** `page=0`, `size=20`

**Backend Logic:**

1. **Auth:** Identify `current_user_id` from token.
2. **Cache Check:** Check Redis for `matches:{user_id}:page_0`.
3. **Fetch & Prune:** If cache miss, run the SQL Pruning Query.
4. **Rank:** Pass results through `MatchEngine.calculateScores()`.
5. **Store:** Save results to Redis.
6. **Response:** Return the serialized `UserMatchDTO` list.

#### 5. Summary of Workflow

1. **SQL:** Filter 10,000 users down to 200 based on City/Budget/Timeline.
2. **Java:** Score and Sort those 200 users using the full `MatchEngine` logic.
3. **JSON:** Return the top 20 to the UI.

---
# Technical Details: Matching

Beyond finding matches, we also want to allow users to match with each other. This is handled through the Match entity which is discussed below. This Match entity keeps track of match attempts, match statuses, and handles edge-cases.

## The Flow:

1. **User A finds User B's profile**  through the **Fello Find API** (above)
2. **User A sends a Match Request to User B which creates a Match entity**
3. **If User B declines, the Match is kept and its status is set to REJECTED. User A cannot send another request for 1 week, the Match object is kept in the database as an archive of the attempt.**

### 1. The Match Entity 
- **Table:** `matches`
- **Columns:**
    - `id`: UUID
    - `initiator_id`: User UUID (Who sent the request)
    - `target_id`: User UUID (Who received it)
    - `status`: ENUM (`PENDING`, `ACCEPTED`, `REJECTED`, `UNMATCHED`, `BLOCKED`, `EXPIRED`)
    - `created_at`: Timestamp
    - `updated_at`: Timestamp (Crucial for cooldown logic)
    - `match_score`: Float (Snapshot of the score at time of match)
- **Constraints:**
    - **Composite Unique Index:** `UNIQUE(initiator_id, target_id)` (Ensure application logic sorts IDs or checks both directions to prevent A->B and B->A duplicates).
### 2. The Chat Entity 
- **Table:** `chats`
- **Columns:**
    - `id`: UUID
    - `match_id`: FK to `matches` (One-to-One relationship)
    - `is_active`: Boolean (True by default; False if a user unmatches/leaves)
    - `archived_at`: Timestamp (Nullable)
### 3. The Logic Flow
1. **Discovery & Filtering:** * User A sees User B on the feed _UNLESS_ a `matches` row exists where `status = BLOCKED`.
2. **Action:** User A clicks "Connect."
    - _Check 1 (Block/Cooldown):_ * If `status == BLOCKED`: Return Error (Invisible to user).
        - If `status == REJECTED` AND `updated_at > NOW() - 7 DAYS`: Return Error ("Cooldown active").
    - _Check 2 (Cap Limit):_ Does User A have > 20 outbound `PENDING` requests?
        - _Yes:_ Return Error ("Too many pending requests. Cancel some to proceed.")
    - _Check 3 (Existing Request / Cross-Connect):_ Does a `PENDING` match exist where `initiator = B` and `target = A`?
        - _Yes:_ **Treat as Accept.** Update status to `ACCEPTED`, create Chat, notify both. (Skip to Step 3).
    - _No existing match:_ Create new `Match` row (A->B) with status `PENDING`.
    - _Notification:_ User B gets "New Request from User A".
3. **Decision:**
    - _If User B clicks Accept:_ Update `Match` status to `ACCEPTED`. **Create `Chat` row.** Trigger WebSocket event to open chat window for both.
    - _If User B clicks Reject:_ Update `Match` status to `REJECTED`. Set `updated_at` to NOW. Archive it.
    - _If User B clicks Block:_ Update `Match` status to `BLOCKED`. User A is removed from B's feed forever.
4. **Maintenance (System Jobs):**
    - **Nightly Cron Job runs: `UPDATE matches SET status = 'EXPIRED' WHERE status = 'PENDING' AND created_at < NOW() - INTERVAL '14 DAYS'`.
    - **Termination:** If User A unmatches: Update `Match` status to `UNMATCHED`.

---
# Technical Details: Chat Architecture
## Fello 1:1 Chat System

While Fello will have 2 different modes of communication (1:1 and Batch), 1:1 is a simpler version that is easy to implement for a MVP and provides the method by which users can message each other after matching but before entering a batch together. 

### 2. Data Models (PostgreSQL)

#### 2.1 The `Conversation` Table

Acts as the container for the chat.

- `id`: UUID (Primary Key)
- `match_id`: UUID (Foreign Key to `matches.id`) - _Critical for 1:1 logic._
- `type`: Enum (`DIRECT`, `BATCH`) 
- `created_at`: Timestamp

#### 2.2 The `Message` Table

Stores the actual content.

- `id`: UUID (Primary Key)
- `conversation_id`: UUID (Indexed FK)
- `sender_id`: UUID (FK to `users.id`)
- `content`: Text (The message body)
- `timestamp`: Timestamp (Default: `now()`)
- `is_read`: Boolean (Default: `false`)

### 3. Real-time Infrastructure (WebSockets)

#### 3.1 Connection Protocol

- **Standard:** STOMP over WebSockets.
- **Handshake Endpoint:** `/ws-fello`
- **Inbound Prefix (`/app`):** Used for actions requiring backend logic (sending messages).
- **Outbound Prefix (`/topic`):** Used for broadcasting messages to subscribers.

#### 3.2 The Topic Hierarchy

For 1:1 chats, the destination is unique to the `conversationId`.

- **Subscription Path:** `/topic/chat/{conversationId}`
- **Message Destination:** `/app/chat.send/{conversationId}`

### 4. Message Lifecycle

1. **Transmission:** User A sends a STOMP frame to `/app/chat.send/{conversationId}`.
2. **Interception (Security):** The `ChannelInterceptor` verifies:
    - Is User A authenticated?
    - Does User A belong to `conversationId`?
3. **Persistence:** The `ChatService` saves the message to the `Message` table.
4. **Broadcast:** The `SimpMessagingTemplate` pushes the message to `/topic/chat/{conversationId}`.
5. **Reception:** User B, who is subscribed to that topic, receives the JSON payload and updates their UI

### 5. REST API Requirements

WebSockets are for _live_ data. We need standard REST endpoints to load the history when a user first opens a chat window.

|Endpoint|Method|Description|
|---|---|---|
|`/api/v1/chats`|`GET`|Returns list of all active conversations for the user.|
|`/api/v1/chats/{id}/messages`|`GET`|Returns paginated history (e.g., last 50 messages).|
|`/api/v1/messages/{id}/read`|`PATCH`|Marks a specific message (and all prior) as read.|

### 6. Security & Guardrails

##### 6.1 Authentication

- **Handshake:** JWT must be passed in the headers of the `CONNECT` frame.
- **Authorization:** Every `SUBSCRIBE` attempt to a `/topic/chat/{id}` must be checked against the `conversation_members` table. If the user isn't a member, the subscription is denied.

##### 6.2 Rate Limiting

- To prevent "Chat Spam," users are limited to 1 message per 500ms.
- Maximum message length: 2,000 characters.

#### 7. MVP Frontend Requirements

- **Auto-Reconnect:** Implement exponential backoff if the socket drops.
- **State Management:** Append new messages to the existing list without re-fetching the entire history.
- **Unread Indicators:** Show a badge on the sidebar if `is_read` is false for the latest message in a conversation.

---

## Technical Details: Batch System


The Batch system is designed to transform Fello from a 1:1 matching tool into a collaborative squad-building platform. By allowing users to aggregate into "Batches" (squads) of 3-5 people, interns can transition from individual searchers to a collective unit. This group dynamic provides significant advantages, such as pooling budgets to afford high-end luxury apartments or gaining market leverage when negotiating leases. The system provides the necessary infrastructure for private discussion, resource sharing, and democratic group management.

### 2. The Batch Lifecycle (Simplified MVP)

#### 2.1 Creation

Any user with a verified identity (e.g., University or Corporate email verification) can initiate a Batch.

- **State: `ACTIVE`** – The Batch is live and visible to its members. It remains in this state until it is either `LOCKED` (full and ready to sign) or `INACTIVE` (disbanded).
- **Role: The "Leader"** – The creator is designated as the primary administrator. While the squad is democratic, the Leader holds the specific power to edit the Batch metadata (title, target neighborhood) and is the primary agent for initiating external invitations to potential members.

#### 2.2 The "Social-First" Invite Flow

1. **Initiation:** An existing member identifies a compatible intern and proposes them for the Batch. This is a "soft" proposal that does not notify the candidate yet.
2. **Calculation:** The server immediately executes a high-performance similarity check, calculating the **Average Match Score** between User X and all current `LOCKED_MEMBER` participants. This provides a group-wide compatibility metric based on the aggregate of all member vectors.
3. **Review:** A WebSocket event (`INVITE_PROPOSED`) pushes an interactive card directly into the Batch's real-time chat.
    - **UI Display:** The card serves as a visual "resume" for the candidate, prominently displaying "Jordan K. (Average 88% Squad Match)" to give members instant context.
4. **Deep Link:** Members can click "Check out profile" to launch a modal containing individual breakdown scores (e.g., "Matches 90% with you, but 70% with Sam"). This allows members to assess how the candidate's lifestyle—such as sleep schedules or cleanliness preferences—affects the group's overall harmony.
5. **Anonymous Veto:** A 24-hour "grace period" window opens. To prevent social friction and awkward confrontations, any member can  **Veto**.
    - **Implication:** If a veto is cast, the proposal is killed silently. The candidate is never informed they were under consideration, and the members do not know who among them cast the veto, preserving the internal squad culture.
6. **Finalization:** If the 24-hour window expires with zero vetoes, the candidate is officially invited. Once they accept, they are promoted to a `LOCKED_MEMBER` and granted full access to the conversation.

#### 2.3 The Kick Logic (Majority Rule)

- **Motion:** To address "ghosting" or toxic behavior, any member can initiate a "Motion to Remove."
- **Voting:** The motion requires a **Simple Majority** (e.g., 2/3 or 3/4) to pass. This democratic threshold ensures that one person cannot disrupt the squad without group consensus.
- **Execution:** Upon a successful vote, the server immediately revokes the user's WebSocket subscription access and deletes their membership record, effectively removing them from the squad's private digital space.

### 3. Database Schema

### 3.1 Table: `batches`

| Column       | Type      | Description                                                                       |
| ------------ | --------- | --------------------------------------------------------------------------------- |
| `id`         | UUID (PK) | Unique identifier for the squad, used in all WebSocket routing.                   |
| `title`      | VARCHAR   | User-defined name (e.g., "The Stripe Squad" or "SF Summer 2026").                 |
| `creator_id` | UUID (FK) | Reference to the user who founded the batch; used for administrative permissions. |
| `status`     | ENUM      | Tracks the lifecycle: `OPEN` (accepting), `LOCKED` (full), `INACTIVE` (deleted).  |
| `created_at` | TIMESTAMP | Audit trail for tracking the longevity of the squad.                              |

#### 3.2 Table: `batch_members`

|Column|Type|Description|
|---|---|---|
|`batch_id`|UUID (FK)|The specific Batch the member is associated with.|
|`user_id`|UUID (FK)|Reference to the individual intern's profile.|
|`status`|ENUM|`PENDING_VETO` (under review) or `LOCKED_MEMBER` (active participant).|
|`joined_at`|TIMESTAMP|Records the moment the user officially cleared the veto window.|

#### 3.3 Table: `batch_votes` (Anonymous Storage)

|Column|Type|Description|
|---|---|---|
|`id`|UUID (PK)|Unique identifier for the voting transaction.|
|`batch_id`|UUID (FK)|Links the vote to the specific squad context.|
|`voter_id`|UUID (FK)|Crucial for preventing double-voting; this column is **never** exposed to the API.|
|`target_id`|UUID (FK)|The candidate or member currently being voted upon.|
|`type`|ENUM|Distinguishes between a join-phase `VETO` or a removal-phase `KICK`.|

### 4. WebSocket Event Architecture

| Event Type        | Destination         | UI Treatment                                                                                    |
| ----------------- | ------------------- | ----------------------------------------------------------------------------------------------- |
| `INVITE_PROPOSED` | `/topic/batch.{id}` | Rich card featuring the candidate’s avatar, bio snippet, and **Average Match Score**.           |
| `VETO_EXECUTED`   | `/topic/batch.{id}` | Neutral system notification: "Proposal for<br><br>$$Name$$<br><br>declined by group consensus." |
| `MEMBER_JOINED`   | `/topic/batch.{id}` | Celebratory banner: "Welcome<br><br>$$Name$$<br><br>! The squad is now at $X/5$ members."       |
| `KICK_INITIATED`  | `/topic/batch.{id}` | Urgent progress bar: "A vote has started to remove<br><br>$$Name$$<br><br>. $X$ votes needed."  |

### 5. Security & Performance

- **The "Batch Average" Calculation:** Performance is optimized by calculating the average similarity score at the moment of invitation. The server fetches the latent vectors for all $N$ current members and the 1 candidate, performing $N$ similarity checks (dot products). For an MVP squad where $N \le 5$, this operation is mathematically trivial ($O(N)$) and does not require complex background processing.
- **Authorization Interceptors:** To prevent unauthorized eavesdropping, every `SUBSCRIBE` request to a `/topic/batch.{id}` destination is intercepted. The backend validates the user's JWT and checks the `batch_members` table to ensure they are a `LOCKED_MEMBER`. Unauthorized requests result in an immediate subscription denial.
- **Anti-Spam Rate Limits:** To maintain a high-quality ecosystem, users are limited to creating one active Batch at a time and initiating a maximum of two invites per day. This prevents "shotgun" invitation tactics and ensures that each proposal is meaningful.


---

## Technical Specification: Fello Auth & Security (Stateless JWT)


The objective is to implement a high-trust, stateless authentication ecosystem tailored for the Fello.sh MVP. By utilizing Google OAuth2 as the foundational Identity Provider, the system establishes an immediate "Walled Garden" where only users with verified academic credentials can participate. This architecture is further enhanced by an asynchronous "Account Linking" layer for LinkedIn, which serves as a second factor of professional verification, ensuring that the community remains exclusive to active interns and students without the need for handling sensitive, confidential documents.

### 2. Layer 1: Primary Authentication (Google .edu verification)

This layer serves as the entry point for all users, creating the initial user session and establishing the baseline identity.

1. **The Flow:** We utilize the standard **OAuth2 Authorization Code flow** with OpenID Connect (OIDC). The SPA initiates the process, but the critical code-to-token exchange happens entirely on the Spring Boot backend to keep the `client_secret` hidden from the browser.
2. **The Logic:**
    - **Extraction:** Upon a successful handshake, the backend extracts the `email` and `email_verified` claims from the OIDC ID Token.
    - **Validation:** The system performs a suffix check. If the `email.endsWith(".edu")`, the user is programmatically granted the **Blue Badge** (Student Identity).
    - **JWT Minting:** A custom **Fello JWT** is generated. This token includes claims such as `userId`, `roles`, and the `isEdu` Boolean. This effectively "promotes" the Google identity into a Fello-specific identity.
3. **The Secure Hand-off:** To transition the user back to the SPA, the server performs a 302 Redirect. The JWT is passed as a URL fragment or query parameter.
    - _Security Note:_ The app is designed to immediately extract this token into `localStorage` and call `window.history.replaceState()` to scrub the token from the browser's address bar, preventing it from leaking into shoulder-surfers' views or server logs.

### 3. Layer 2: Account Linking (LinkedIn Verification)

Once a user has an active session, they can choose to "Verify Professional Identity." This is treated as a separate, distinct OAuth2 flow that enriches the existing user profile.

#### 3.1 The Linking Flow
1. **Initiation:** The user triggers a "Connect LinkedIn" action within their profile settings.
2. **Contextual Handshake:** SPA redirects the user to the backend at `/oauth2/authorization/linkedin`.
    - _Implication:_ The backend must maintain the current user's context during this redirect. This is achieved by storing the user's `userId` in the OAuth2 `state` parameter or a temporary encrypted cookie.
3. **Authorization:** The user approves Fello.sh to access their LinkedIn "Lite Profile" (name, photo, and public profile URL).
4. **The Permanent Link:**
    - The backend receives the LinkedIn callback, extracts the unique LinkedIn member ID (`sub`), and the public `profile_url`.
    - The backend performs a database update to bridge the two identities:
        ```
        UPDATE users 
        SET linkedin_id = :liId, 
            linkedin_url = :liUrl, 
            is_profile_verified = true 
        WHERE id = :currentUserId;
        ```
5. **The Reward:** The user's JWT is refreshed (or the frontend state is updated) to reflect the **Tier 2: Profile Verified** status, signaling to potential roommates that this user is a "vetted" professional.
### 4. Security Layers

#### 4.1 REST Security (`OncePerRequestFilter`)

A custom `JwtAuthenticationFilter` is placed at the front of the Spring Security filter chain. For every request:

- **Signature Verification:** The filter extracts the "Bearer" token from the `Authorization` header and validates the cryptographic signature using the server's private secret key.
- **Expiry Check:** It ensures the `exp` claim has not passed.
- **Context Population:** If valid, the filter creates an `Authentication` object and injects it into the `SecurityContextHolder`. This allows downstream services (like the MatchEngine) to access the `userId` via a simple `@AuthenticationPrincipal` annotation.
    

#### 4.2 WebSocket Security (`ChannelInterceptor`)

Standard HTTP security filters do not protect WebSocket traffic once the initial handshake is complete. Therefore, we implement a **STOMP ChannelInterceptor**.

- **The CONNECT Gate:** When the client sends the initial `CONNECT` frame, the interceptor extracts the JWT from the `nativeHeaders`. If the token is invalid or missing, the connection is terminated before any data can be exchanged.
- **Subscription Guardrails:** When a user attempts to `SUBSCRIBE` to a specific destination (e.g., `/topic/chat/{matchId}`), the interceptor performs a "Relationship Check" against PostgreSQL. It verifies that the `userId` extracted from the JWT is actually a participant in that specific match or batch. This prevents "ID guessing" where a malicious user tries to listen to a private conversation they don't belong to.
    

### 5. User Verification Tiers (MVP Scope)

Visual trust cues are essential for a community-driven app.

|Tier|Badge|Verification Method|Impact on UX|
|---|---|---|---|
|**Tier 1**|**Blue Badge**|**Primary Auth:** Verified via Google `.edu` domain check during initial login.|Allows the user to browse matches and initiate messages.|
|**Tier 2**|**Profile Verified**|**Account Linking:** Verified via a secondary LinkedIn OAuth2 handshake.|Adds a "Vetted" checkmark to the profile, significantly increasing match acceptance rates.|

### 6. Technical Stack & Security Guardrails

- **Spring Security:** Configured to handle multiple `ClientRegistration` objects for Google and LinkedIn.
- **Statelessness:** No server-side sessions (`HttpSession`) are used. This allows the API to scale horizontally across multiple containers (e.g., on Railway or AWS) without requiring sticky sessions.
- **CSRF Protection:** Since we use JWTs and not cookies for API calls, CSRF is disabled for REST endpoints, significantly simplifying the frontend-to-backend communication.
- **Data Minimization:** Fello.sh never stores LinkedIn access tokens. Once the `linkedin_id` is retrieved and verified, the tokens are discarded. We only store the "Fact of Verification."

---
### Algorithmic Complexity

**Query Complexity:**
- SQL Pruning: O(log N) with B-tree indexes on city_id, budget
- Vector Scoring: O(K) where K = candidates returned (200)
- In-memory Sort: O(K log K) ≈ O(1600) operations
- **Total: O(log N + K log K)**

**With 10,000 users:**
- Without pruning: O(10,000) = 10k operations per match
- With pruning: O(log 10k + 200 log 200) ≈ O(13 + 1600) = ~1600 operations
- **83% reduction in compute**

**Space Complexity:**
- User vectors: O(K * D) where D = 8 features ≈ 1.6KB
- Redis cache: O(U * 20 * 100B) = 2MB per 1000 active users

---
### Testing Approach

**Unit Tests:**
- MatchEngine.calculateBudgetScore() with edge cases
- Timeline overlap with partial/full/zero overlap
- Hard constraint multipliers (0 or 1)

**Integration Tests:**
- Full matching flow: Auth → Profile → Query → Score
- WebSocket message delivery
- Batch invitation acceptance/veto

**Load Tests:**
- 100 concurrent match requests
- 1000 users in database
- Target: <2s response time for match queries

---

### Key Metrics

**Application:**
- Match query latency (p50, p95, p99)
- Cache hit rate (target: >80%)
- WebSocket connection count
- Messages sent per second

**Business:**
- Match acceptance rate
- Average match score of accepted matches
- Time to first message
- Batch formation rate

**Alerts:**
- Match query >3s (investigate SQL indexes)
- Cache hit rate <60% (review TTL strategy)
- WebSocket connections >1000 (scale horizontally)

---
