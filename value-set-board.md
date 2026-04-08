# Value Set Board

## One-Line Summary

A public grid where anyone can mark whether something has a value—and watch consensus emerge.

---

## Pitch

Most comparison tools are noisy: reviews are biased, rankings are opaque, discussions fragment into arguments. **Value Set Board** strips everything down to the simplest unit of shared meaning:

> Does this thing have this quality? Yes or No.

By aggregating thousands of these micro-decisions, the system reveals consensus, disagreement, and perception gaps. It becomes a kind of truth table for collective intuition.

**Examples:**

| | Fast | Simple | Safe | Productive |
|---|---|---|---|---|
| C | ✅ 94% | ❌ 12% | ❌ 8% | ❌ 31% |
| Rust | ✅ 88% | ❌ 19% | ✅ 96% | ⚡ 51% |
| Python | ❌ 23% | ✅ 91% | ❌ 44% | ✅ 87% |
| Go | ✅ 72% | ✅ 78% | ✅ 69% | ✅ 82% |

(⚡ = contested, ~50/50 split)

No accounts. No comments. Just signal.

---

## Core Concept

A **2D grid**:

- **Y-axis (Items):** things being evaluated — C, Rust, Python, React, PostgreSQL, Docker, etc.
- **X-axis (Values):** abstract qualities — Speed, Simplicity, Safety, Reliability, Expressiveness, Productivity
- **Cell:** a binary judgment — "Does this item have this value?"

Each cell stores `yes_count` and `no_count`. Users contribute by clicking. The aggregate is displayed as a percentage.

### What is a "Value"?

A **value** is a general, abstract quality that can be meaningfully applied across many items and judged without domain expertise.

**Valid:** Speed, Simplicity, Safety, Reliability, Flexibility, Expressiveness, Productivity, Predictability, Maturity, Community

**Invalid (too specific or mechanistic):**
- Memory Safety ← a sub-property of Safety
- Garbage Collection ← an implementation detail
- Async/Await ← a language feature
- Has a LSP ← a fact, not a perception

The line between "value" and "feature" is key. Values are perceptions; features are checkboxes. This distinction keeps the grid meaningful.

---

## Product Design

### Interaction Model

1. User visits `valuesetboard.io`
2. Sees the grid (default board: Programming Languages)
3. Clicks any cell → vote recorded instantly
4. Cell updates in real time to reflect new aggregate
5. Done. No account, no form, no friction.

For users who return: session cookie remembers their votes. They can change a vote at any time (no permanent commitment).

### Vote Display Modes

Each cell can display in multiple modes (toggle in settings):

- **Binary consensus:** ✅ or ❌ (majority wins, threshold 60%)
- **Percentage:** "78% yes"
- **Confidence bar:** horizontal fill proportional to yes%
- **Controversy indicator:** ⚡ when yes% is 40–60% (genuinely contested)
- **Heatmap:** background color from red (0%) → green (100%)

### Boards

The grid is organized into **boards** (curated collections of items sharing a category):

- Programming Languages *(seed board)*
- Web Frameworks
- Databases
- Cloud Providers
- Operating Systems
- Text Editors / IDEs
- Build Tools

Each board has its own set of relevant values (Databases board might include Scalability, ACID, Developer Ergonomics; Languages board won't need ACID).

---

## Data Model

```sql
-- Items: things being evaluated
CREATE TABLE items (
  id         UUID PRIMARY KEY,
  board_id   UUID NOT NULL REFERENCES boards(id),
  name       TEXT NOT NULL,
  slug       TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  is_active  BOOLEAN DEFAULT true,
  UNIQUE (board_id, slug)
);

-- Values: abstract qualities
CREATE TABLE values (
  id           UUID PRIMARY KEY,
  name         TEXT NOT NULL UNIQUE,
  description  TEXT,          -- shown on hover
  is_validated BOOLEAN DEFAULT false,
  created_at   TIMESTAMPTZ DEFAULT now()
);

-- Board-Value mapping: which values apply to which board
CREATE TABLE board_values (
  board_id  UUID REFERENCES boards(id),
  value_id  UUID REFERENCES values(id),
  position  INT,
  PRIMARY KEY (board_id, value_id)
);

-- Votes: the core data
CREATE TABLE votes (
  id         UUID PRIMARY KEY,
  item_id    UUID NOT NULL REFERENCES items(id),
  value_id   UUID NOT NULL REFERENCES values(id),
  session_id TEXT NOT NULL,   -- hashed fingerprint
  has_value  BOOLEAN NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  UNIQUE (item_id, value_id, session_id)  -- one vote per session per cell
);

-- Materialized aggregate (updated via trigger or cron)
CREATE MATERIALIZED VIEW cell_aggregates AS
SELECT
  item_id,
  value_id,
  COUNT(*) FILTER (WHERE has_value = true)  AS yes_count,
  COUNT(*) FILTER (WHERE has_value = false) AS no_count,
  COUNT(*) AS total
FROM votes
GROUP BY item_id, value_id;
```

**Aggregate refresh strategy:** Use `pg_notify` + a lightweight worker to refresh cell_aggregates on write. For high traffic, move aggregates to Redis counters (`INCR`/`DECR`) with async persistence.

---

## API Design

### Public REST API

```
GET /api/boards
→ list of boards with metadata

GET /api/boards/:slug
→ board metadata + ordered item/value lists

GET /api/boards/:slug/grid
→ full grid data: { cells: { [item_id]: { [value_id]: { yes, no, total } } } }

POST /api/vote
Body: { item_id, value_id, has_value: true|false }
Headers: X-Session-ID: <fingerprint>
→ { yes_count, no_count, total }

GET /api/items/:id
→ item detail + all its cell aggregates (value profile)
```

WebSocket (for live updates):
```
ws://valuesetboard.io/live/:board_slug
→ server pushes { item_id, value_id, yes_count, no_count } on each vote
```

---

## Anti-Abuse

### Session Identity

No login = no strong identity, but we can approximate:

```
session_id = SHA256(ip_subnet + user_agent + accept_language + canvas_fingerprint)
```

This is a **probabilistic** identity, not a perfect one. Good enough to:
- Prevent casual ballot stuffing
- Allow vote changes
- Not block legitimate users behind shared IPs (use /24 subnet, not full IP)

### Rate Limiting

- Max **30 votes per minute per session**
- Max **1 vote per cell per session** (enforced in DB via UNIQUE constraint)
- New items: max 5 new items per session per day

### Spam / Junk Items

- Items with < 10 total votes are hidden from the main grid
- New items enter a "pending" state, visible only to the submitter until they reach threshold
- Fuzzy deduplication on item names (trigram similarity via `pg_trgm`)

### Coordinated Manipulation

For cells showing sudden spikes (>3σ from rolling average):
- Temporarily suspend new votes on that cell
- Flag for manual review
- Display "vote integrity question" notice to users

---

## Tech Stack

### Backend

- **Language:** Go or Rust (performance matters for a stateless vote proxy)
- **Database:** PostgreSQL (aggregates via materialized view + Redis for hot counters)
- **Cache:** Redis for session rate limiting + real-time aggregates
- **API:** REST + WebSocket (gorilla/websocket or axum)
- **Auth (admin only):** simple JWT for board/value management

### Frontend

- **Framework:** SvelteKit or Next.js (SSR for SEO, fast interactivity)
- **UI:** custom grid component; virtualized for large boards (>100 items)
- **Real-time:** WebSocket for live cell updates
- **Styling:** minimal; the grid is the UI

### Infrastructure

- **Hosting:** Fly.io or Railway (simple scale-out for stateless API)
- **Database:** Managed PostgreSQL (Neon, Supabase, or RDS)
- **CDN:** Cloudflare (cache GET /grid responses for ~5 seconds)
- **Assets:** no heavy JS bundles; aim for < 100KB initial load

---

## Seed Data Plan

The board is only valuable with enough initial data to feel alive. Pre-seed:

**Programming Languages board:**
- Items: C, C++, Rust, Go, Python, JavaScript, TypeScript, Java, Kotlin, Swift, Haskell, Elixir, Ruby, PHP, Zig
- Values: Fast, Simple, Safe, Expressive, Productive, Mature, Predictable, Community-Driven

Run an internal voting session (team + friends) to seed ~500 votes before launch. Enough to display meaningful percentages without being manufactured consensus.

---

## Future Enhancements

### Controversy View

A dedicated view sorted by cells with ~50/50 splits. These are the interesting ones—where reasonable people genuinely disagree. Surfaces the ideological fault lines of a community.

### Item Profiles

Click any item → see its full value profile as a horizontal bar chart. "Python is 91% Simple, 87% Productive, 23% Fast." Instantly intuitive.

### Time Series

Track how perception of an item changes over time. "Is Rust simple?" was 8% in 2019; is it 25% now as the ecosystem matures? Longitudinal data becomes its own story.

### Comparison Mode

Select two items → see a side-by-side diff view. "Go vs Rust on every value." Powered by the same underlying data.

### Embed Widget

```html
<iframe src="https://valuesetboard.io/embed/languages?items=rust,go&values=fast,simple" />
```

Blog posts, documentation sites, and README files can embed live cells. Drives traffic and keeps data fresh.

### API for Researchers

Public read API (rate-limited) for academics studying collective perception of technology. Data exports available. Cite-able DOI for the dataset.

---

## Design Philosophy

- **Binary over complex** — forced choice produces cleaner signal than rating scales
- **Crowd over expert** — no official answers; the community *is* the answer
- **Structure over discussion** — no comments, no threads, no noise
- **Emergence over control** — don't editorialize; let the numbers speak
- **Zero friction** — if voting takes more than one click, fewer people vote, and the data gets worse

---

## What Makes This Different

Most comparison tools ask "which is better?" This asks "what does this *mean* to people?"

The result isn't a ranking. It's a map of collective perception—revealing how a community mentally models its own tools. That's genuinely new information, not just sorted reviews.
