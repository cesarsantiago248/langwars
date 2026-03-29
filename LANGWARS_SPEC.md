# LANGWARS — Full Technical Specification & MVP Plan

---

## PART 1: Expanded Technical Specification

### 1.1 Product Overview

**LangWars** is a real-time multiplayer translation game where players compete to translate words/sentences between English and Spanish. The core loop: read → translate → score → learn from mistakes.

**Target Users:**
- Language learners (beginner to intermediate)
- Students preparing for Spanish/English exams
- Casual gamers who enjoy word games
- Teachers using gamification in classrooms

### 1.2 Game Modes (Detailed)

#### Mode 1: Free-For-All (FFA) Translation
- 2-10 players per lobby
- Each round: 1 phrase shown, first to type correct translation wins points
- Round duration: 15-30 seconds (configurable by host)
- 10 rounds per game (configurable)
- Bidirectional: sometimes English→Spanish, sometimes Spanish→English

#### Mode 2: Team Mode Translation
- 2 teams (Red vs Blue), 2-5 players each
- Same round mechanics as FFA but team scores are combined
- Teammates can see each other's answers (no collision/cheating — it's cooperative)
- Bonus points if multiple team members answer correctly

#### Mode 3: FFA Fastest Mode
- Same as FFA but rounds are 5-8 seconds
- Emphasis on raw speed
- Smaller point gaps (accuracy still matters but speed is paramount)

#### Mode 4: Image Mode (FFA & Team)
- Instead of text phrase, show an image describing a concept
- Player types the word/phrase in the target language
- Example: Image of a "beach" → player types "playa"
- Requires image hosting and potentially image recognition for validation

#### Mode 5: Classroom Mode (Kahoot-style)
- 1 host (teacher) + unlimited students
- Host sees live dashboard of student responses
- Students compete individually but visible to host for tracking
- Host can pause, show leaderboard, control difficulty
- Great for exam prep or classroom activities

### 1.3 Core Game Mechanics

#### Answer Validation Engine
```
Input: player_answer (string)
Reference: correct_translation (string) + synonyms (string[])
Preprocessing: lowercase, trim, normalize unicode
Validation:
  1. Exact match (case-insensitive)
  2. Synonym match (fuzzy matching with Levenshtein distance ≤ 2)
  3. Common abbreviation mapping (e.g., "U" → "you", "d" → "de")
Scoring:
  - Base points: 100
  - Time bonus: +50 if answered in first 25% of round time
  - Accuracy multiplier: exact=1.0, synonym=0.8, abbreviation=0.7
```

#### Grammar Error Classification
Each phrase/word in DB tagged with:
- `grammatical_category`: pronoun | verb | noun | adjective | adverb | preposition | conjunction | article
- `conjugation_type`: present | past | future | conditional | imperative | subjunctive
- `difficulty_level`: 1-5 (algorithmically determined + user feedback)
- `context_tags`: sports | cars | science | games | food | travel | business | casual

#### End-Game Summary Stats
Per player per game:
- Accuracy %: correct / total questions
- Average response time
- Points earned
- Rank in lobby
- Most common error types
- Words answered wrong (and how many times across history)
- "Rare words" used correctly (words that <5% of players answer correctly)

#### User Profile & Study Path
Aggregate stats per user:
- Grammar categories with lowest accuracy → recommended courses
- Historical "weak words" list (words they repeatedly fail)
- Suggested study path based on error patterns
- Mastery level per grammar category (0-100%)

### 1.4 Phrase Database Schema (Content)

Sources:
- OpenLibrary API (public domain books, scraping sentences)
- Manual curation
- User-submitted phrases (with approval queue)

Each phrase entry:
```
phrase_id, original_text, language (en|es), difficulty_score
translations: [{text, is_primary, synonyms[], abbreviations[]}]
grammar_tags: [{category, conjugation}]
context_tags: [sports, cars, etc]
usage_frequency: rare | uncommon | common (based on corpus data)
```

### 1.5 User Account System

#### Authentication
- Primary: OAuth2 via Google, Apple, Facebook
- Secondary: Email/password (for users without social accounts)
- Session management: JWT tokens (access + refresh)

#### Account Data
- User profile (username, avatar, country, native_language)
- ELO rating per game mode
- Achievement badges
- Friend list
- Notification preferences
- Email preferences (marketing, reminders, friend requests)

#### Social Features
- Add friends by username or email
- Friend leaderboards per mode
- Invite friends to private lobby
- Referral system (bonus ELO or features for referring friends)

### 1.6 Achievements System
| Badge | Criteria |
|-------|----------|
| First Blood | Win your first FFA game |
| Speed Demon | Win 5 fastest-mode games |
| Polyglot | Play all game modes |
| Classroom Hero | Host 10 classroom sessions |
| 100 Games | Complete 100 games |
| Perfect Round | Get all questions correct in a game |
| Streak Master | Win 5 games in a row |
| Study Buddy | Complete your first grammar course |
| Wordsmith | Correctly translate 50 "rare" words |

### 1.7 Non-Functional Requirements

#### Performance
- WebSocket latency < 100ms for game events (real-time feel)
- API response time < 200ms (p95)
- Database queries < 50ms (p95)
- Support 1000 concurrent lobbies (10k concurrent players) at launch

#### Scalability
- Horizontal scaling via Redis pub/sub for WebSocket state
- Stateless FastAPI app servers behind load balancer
- Auto-scaling groups on cloud provider

#### Security
- HTTPS everywhere (Cloudflare SSL)
- Rate limiting (100 req/min per IP for API, 10/min for auth endpoints)
- Input sanitization (prevent XSS in usernames, answer fields)
- JWT validation on every protected endpoint
- CORS restricted to known origins
- DDoS protection via Cloudflare
- Database SQL injection prevention via ORM

#### Availability
- Target: 99.5% uptime
- Deploy in 2 regions if using managed DB (multi-region Postgres or just primary + read replica)

---

## PART 2: Database Schema Design (ERD)

### 2.1 Entity Relationship Overview

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    users     │     │   sessions   │     │   games      │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ id (PK)      │────<│ id (PK)      │     │ id (PK)      │
│ username     │     │ user_id (FK) │     │ lobby_id (FK)│
│ email        │     │ token        │     │ mode         │
│ password_hash│     │ created_at   │     │ started_at   │
│ avatar_url   │     │ expires_at   │     │ ended_at     │
│ country      │     └──────────────┘     │ host_id (FK) │
│ native_lang  │                           └──────┬───────┘
│ elo_ffa      │                                  │
│ elo_team     │          ┌───────────────────────┤
│ elo_fastest  │          │                       │
│ created_at   │     ┌─────┴─────┐          ┌──────┴───────┐
└──────┬───────┘     │  lobby    │          │ game_players │
       │            ├───────────┤          ├──────────────┤
       │            │ id (PK)   │          │ id (PK)      │
       │            │ code      │          │ game_id (FK) │
       │            │ host_id   │          │ user_id (FK) │
       │            │ mode      │          │ team         │
       │            │ settings  │          │ score        │
       │            │ status    │          │ accuracy     │
       │            │ created_at│          │ avg_speed_ms │
       └───────────>│ max_players     │     │ rank         │
                    └───────┬───────┘          └──────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
   ┌────┴────┐        ┌─────┴─────┐       ┌──────┴──────┐
   │ lobby_  │        │  lobby_   │       │  lobby_     │
   │ players │        │ messages  │       │ settings    │
   ├─────────┤        ├───────────┤       ├──────────────┤
   │ id (PK) │        │ id (PK)   │       │ lobby_id(FK)│
   │ lobby_id│        │ lobby_id  │       │ setting_key │
   │ user_id │        │ user_id   │       │ setting_val │
   │ team    │        │ message   │       └──────────────┘
   │ joined  │        │ created_at│
   └─────────┘        └───────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   phrases    │     │   phrase_    │     │ phrase_      │
├──────────────┤     │ translations │     │ grammar_tags │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ id (PK)      │────<│ id (PK)      │     │ id (PK)      │
│ original_en │     │ phrase_id(FK)│────<│ phrase_id(FK)│
│ original_es │     │ text         │     │ category     │
│ difficulty   │     │ is_primary   │     │ conjugation  │
│ context     │     │ synonyms     │     └──────────────┘
│ frequency   │     │ abbreviations│
└──────┬──────┘     └──────────────┘
       │
       │         ┌──────────────┐     ┌──────────────┐
       │         │ game_rounds  │     │  answers     │
       └────────>├──────────────┤     ├──────────────┤
                │ id (PK)      │     │ id (PK)      │
                │ game_id (FK) │     │ round_id (FK)│
                │ phrase_id(FK)│     │ user_id (FK) │
                │ round_number │     │ answer_text  │
                │ time_limit   │     │ is_correct   │
                │ winner_id    │     │ points_earned│
                └──────────────┘     │ response_ms  │
                                     │ answer_type  │
                                     └──────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   courses    │     │  chapters    │     │  questions   │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ id (PK)      │────<│ id (PK)      │────<│ id (PK)      │
│ title        │     │ course_id(FK)│     │ chapter_id(FK│
│ description  │     │ title        │     │ type         │
│ category     │     │ order        │     │ prompt_en    │
│ difficulty   │     │ description  │     │ prompt_es    │
│ icon_url     │     └──────────────┘     │ correct_ans  │
└──────┬──────┘                           │ synonyms    │
       │         ┌──────────────┐          └──────┬───────┘
       │         │user_progress│                  │
       └────────>├──────────────┤                  │
                 │ id (PK)      │                  │
                 │ user_id (FK) │                  │
                 │ course_id(FK)│                  │
                 │ chapter_id(FK)                  │
                 │ status       │                  │
                 │ score        │                  │
                 │ completed_at │                  │
                 └──────────────┘                  │

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ achievements  │     │user_achieve  │     │   friends    │
├──────────────┤     ├──────────────┤     ├──────────────┤
│ id (PK)      │────<│ id (PK)      │     │ id (PK)      │
│ name         │     │ user_id (FK) │     │ user_id (FK) │
│ description  │     │ achieve_id(FK│────>│ friend_id(FK)│
│ icon_url     │     │ earned_at    │     │ status       │
│ criteria     │     └──────────────┘     │ created_at   │
└──────────────┘                           └──────────────┘

┌──────────────┐
│ user_errors   │
├──────────────┤
│ id (PK)      │
│ user_id (FK) │
│ phrase_id(FK)│
│ wrong_count  │
│ last_attempt │
│ grammar_cat  │
│ updated_at   │
└──────────────┘
```

### 2.2 Indexes Required
- `users.username` (unique)
- `users.email` (unique)
- `lobby.code` (unique, for join-by-code)
- `game_players.game_id + user_id` (unique)
- `phrases.difficulty`
- `user_errors.user_id + grammar_cat`

### 2.3 Redis Keys Structure
```
lobby:{lobby_id}:state     → Hash (current game state)
lobby:{lobby_id}:players   → Set (player user_ids)
lobby:{lobby_id}:round     → String (current round number)
game:{game_id}:answers     → Sorted Set (user_id → response_time)
user:{user_id}:session     → String (WS connection info)
leaderboard:ffa:global    → Sorted Set (user_id → elo)
leaderboard:ffa:country:{cc} → Sorted Set
```

---

## PART 3: API Endpoint Specifications

### 3.1 Authentication
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/register` | Email + password registration |
| POST | `/auth/login` | Email + password login |
| POST | `/auth/oauth/google` | Google OAuth callback |
| POST | `/auth/oauth/facebook` | Facebook OAuth callback |
| POST | `/auth/refresh` | Refresh access token |
| POST | `/auth/logout` | Invalidate refresh token |
| GET | `/auth/me` | Get current user |

### 3.2 Users
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/users/{user_id}` | Get user profile (public) |
| PATCH | `/users/me` | Update own profile |
| GET | `/users/me/stats` | Get detailed personal stats |
| GET | `/users/me/weaknesses` | Get grammar categories to study |
| GET | `/users/{user_id}/achievements` | Get user's achievements |

### 3.3 Friends
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/friends` | List friends |
| POST | `/friends/{user_id}` | Send friend request |
| PATCH | `/friends/{friend_id}` | Accept/reject request |
| DELETE | `/friends/{friend_id}` | Remove friend |

### 3.4 Leaderboards
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/leaderboards/{mode}` | Global leaderboard (mode: ffa/team/fastest) |
| GET | `/leaderboards/{mode}/country/{cc}` | Country leaderboard |
| GET | `/leaderboards/me/{mode}` | User's rank in mode |

### 3.5 Lobbies (REST for setup)
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/lobbies` | Create new lobby |
| GET | `/lobbies/{lobby_id}` | Get lobby info |
| PATCH | `/lobbies/{lobby_id}` | Update lobby settings (host only) |
| POST | `/lobbies/join/{code}` | Join lobby by code |
| DELETE | `/lobbies/{lobby_id}` | Delete/close lobby (host only) |

### 3.6 Game (WebSocket Protocol)

**Connection**: `wss://api.langwars.io/ws/game/{lobby_id}?token={jwt}`

#### Client → Server Messages
```json
{ "type": "join", "user_id": 123 }
{ "type": "start_game" }
{ "type": "submit_answer", "answer": "playa", "round_id": 456 }
{ "type": "leave_lobby" }
{ "type": "chat", "message": "gg!" }
```

#### Server → Client Messages
```json
{ "type": "player_joined", "user": {...}, "player_count": 5 }
{ "type": "game_started", "settings": {...}, "total_rounds": 10 }
{ "type": "new_round", "round_id": 456, "phrase": "beach", "source_lang": "en", "time_limit": 20 }
{ "type": "player_answered", "user_id": 123, "correct": true }
{ "type": "round_ended", "correct_answer": "playa", "winner": { "user_id": 123, "points": 150 }, "leaderboard": [...] }
{ "type": "game_ended", "summary": { "rankings": [...], "player_stats": {...} } }
{ "type": "error", "code": "LOBBY_NOT_FOUND", "message": "..." }
```

### 3.7 Phrases (Admin/Content)
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/phrases` | List phrases (with filters) |
| POST | `/phrases` | Add new phrase |
| PATCH | `/phrases/{id}` | Update phrase |
| POST | `/phrases/import` | Bulk import from source |
| GET | `/phrases/suggest` | Get random phrases for game based on settings |

### 3.8 Courses
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/courses` | List all courses |
| GET | `/courses/{course_id}` | Get course with chapters |
| GET | `/courses/recommended` | Get recommended courses for user |
| POST | `/courses/{course_id}/chapters/{chapter_id}/complete` | Mark chapter complete |
| POST | `/courses/{course_id}/chapters/{chapter_id}/quiz` | Submit chapter quiz answers |

### 3.9 Achievements
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/achievements` | List all achievements |
| GET | `/achievements/me` | List user's earned achievements |

### 3.10 Admin
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/admin/phrases/import` | Trigger phrase import job |
| GET | `/admin/stats` | Platform stats |
| POST | `/admin/announcements` | Send push notification |

---

## PART 4: MVP Scope Definition

### 4.1 Phase 1: MVP Launch (Must-Have)

#### Core Experience
- [x] FFA Translation mode only (simplest game loop)
- [x] Text input mode only (no voice)
- [x] Basic lobby creation with shareable code
- [x] Real-time WebSocket gameplay (2-8 players)
- [x] Answer validation with synonym support
- [x] Points calculation (time + accuracy)
- [x] End-game summary (accuracy %, speed, score, rank)
- [x] Basic global leaderboard (no country yet)

#### Accounts & Auth
- [x] Google OAuth only (simplest)
- [x] Email/password fallback
- [x] Basic user profile (username, avatar, country)
- [x] JWT session management

#### Content
- [x] 500 pre-seeded phrases (mix of difficulties, no context filtering yet)
- [x] Manual phrase validation before adding to pool

#### Social
- [x] None (add post-launch)

#### Grammar/Study
- [x] None (add at 6-week post-launch)

### 4.2 Phase 2: First Major Release (Week 6-12)

#### Game Modes
- [ ] Team Mode
- [ ] Fastest Mode
- [ ] Context filtering (sports, science, etc.)

#### Features
- [ ] Country-based leaderboards
- [ ] Friend list + inviting friends to lobby
- [ ] ELO rating system
- [ ] Achievements (first 5 badges)
- [ ] Basic push notifications (game starting, friend online)

#### Content
- [ ] 2000 phrases
- [ ] Automatic phrase scraping from OpenLibrary
- [ ] Context tagging

### 4.3 Phase 3: Social & Engagement (Week 12-20)

#### Game Modes
- [ ] Image Mode
- [ ] Classroom Mode

#### Features
- [ ] Full achievements system
- [ ] Referral system
- [ ] Email notifications
- [ ] Grammar study section (basic courses)
- [ ] Weakness tracking + recommendations

#### Tech
- [ ] Native mobile apps (or PWA upgrade)

### 4.4 Phase 4: Scale (Week 20+)

#### Features
- [ ] Voice mode for answering
- [ ] Advanced grammar error analysis
- [ ] Adaptive difficulty
- [ ] Social leaderboards (friends-first)
- [ ] Tournaments

### 4.5 Features to CUT from MVP

| Feature | Reason to Cut |
|---------|---------------|
| Team Mode | Adds team balancing logic, UI complexity |
| Fastest Mode | Can be same lobby settings adjustment |
| Image Mode | Requires image hosting, CDN, image validation logic |
| Classroom Mode | Requires host dashboard, more complex WS state |
| Voice Input | STT integration complexity, cross-language issues |
| Country Leaderboards | Need geolocation + aggregation infra |
| Friend System | Social complexity, can be v2 |
| Grammar Courses | Content creation + course engine |
| Achievements | Gamification layer, not core loop |
| Push Notifications | Adds infrastructure (FCM/APNs) |
| Multiple OAuth providers | Google only is sufficient for MVP |
| Synonym scraping | Manual synonym curation for MVP pool |

### 4.6 MVP Tech Stack (Final Recommendation)

| Layer | Choice | Cost |
|-------|--------|------|
| Frontend | Next.js 14 + TanStack React + TailwindCSS | $0 (Vercel free) |
| Backend | FastAPI (Python 3.11) + SQLAlchemy (async) | $0-20 (Railway) |
| Database | PostgreSQL (Neon free tier) + Redis (Upstash free) | $0-15 |
| Auth | Supabase Auth | $0 (free tier) |
| Realtime | Native WebSockets + Redis Pub/Sub | Included above |
| Hosting | Railway (backend) + Vercel (frontend) | ~$0-20/mo |
| Domain | Cloudflare Registrar | $10/year |
| Email | Resend | $0 (500 emails/mo free) |
| CDN/DDoS | Cloudflare | $0 (free tier) |
| Image Storage | Cloudflare R2 | $0 (5GB/mo free) |

**Total MVP Cost: ~$0-20/month** (until you hit significant scale)

---

## Summary

| Task | Status |
|------|--------|
| 1. Expanded Technical Spec | ✅ Complete |
| 2. Database Schema (ERD) | ✅ Complete |
| 3. API Endpoints | ✅ Complete |
| 4. MVP Scope | ✅ Complete |
