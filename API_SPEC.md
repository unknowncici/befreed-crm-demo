# Befreed CRM — API Specification

**Version:** 1.0
**Stack:** Node.js (Express) or Python (FastAPI) — developer's choice
**Database:** Supabase (Postgres)
**Auth:** Simple API key header for v1 (`X-API-Key`) — internal tool only
**Base URL:** `https://api.befreed.internal/v1` (or localhost:3000 in dev)

---

## Database Schema

```sql
-- Creators
CREATE TABLE creators (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name                TEXT NOT NULL,
  phone               TEXT NOT NULL UNIQUE,
  type                TEXT NOT NULL CHECK (type IN ('boosted', 'regular')),
  color               TEXT,                    -- hex, for UI avatar
  avatar_initials     TEXT,
  contract_signed_at  TIMESTAMPTZ,
  handle_requested_at TIMESTAMPTZ,
  handle_confirmed_at TIMESTAMPTZ,
  warmup_started_at   TIMESTAMPTZ,
  warmup_approved_at  TIMESTAMPTZ,
  active_since        TIMESTAMPTZ,
  posting_target      INT DEFAULT 6,           -- videos per week
  created_at          TIMESTAMPTZ DEFAULT NOW()
);

-- Content submissions (from Sideshift)
CREATE TABLE content_submissions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id    UUID REFERENCES creators(id) ON DELETE CASCADE,
  title         TEXT,
  submitted_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  views         INT,
  status        TEXT NOT NULL CHECK (status IN ('pending_review', 'approved', 'rejected')),
  reviewed_at   TIMESTAMPTZ,
  reviewed_by   TEXT,                          -- 'Nathan' etc.
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Group chats (one per creator)
CREATE TABLE group_chats (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id          UUID REFERENCES creators(id) ON DELETE CASCADE UNIQUE,
  imessage_chat_guid  TEXT UNIQUE,             -- BlueBubbles chat identifier
  participants        TEXT[],                  -- display names in the chat
  created_at          TIMESTAMPTZ DEFAULT NOW(),
  last_team_reply_at  TIMESTAMPTZ,
  last_checkin_at     TIMESTAMPTZ
);

-- Messages (mirrored from iMessage)
CREATE TABLE messages (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  chat_id       UUID REFERENCES group_chats(id) ON DELETE CASCADE,
  imessage_guid TEXT UNIQUE,                   -- dedup from BlueBubbles
  role          TEXT NOT NULL CHECK (role IN ('team', 'creator', 'system')),
  sender_name   TEXT NOT NULL,
  body          TEXT NOT NULL,
  sent_at       TIMESTAMPTZ NOT NULL,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Alerts
CREATE TABLE alerts (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id    UUID REFERENCES creators(id) ON DELETE CASCADE,
  type          TEXT NOT NULL,                 -- posting_gap, reply_gap, deadline_urgent, etc.
  severity      TEXT NOT NULL CHECK (severity IN ('flash', 'red', 'amber')),
  title         TEXT NOT NULL,
  detail        TEXT,
  resolved      BOOLEAN DEFAULT FALSE,
  resolved_at   TIMESTAMPTZ,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- Feedback log (Nathan's notes per creator)
CREATE TABLE feedback_log (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id  UUID REFERENCES creators(id) ON DELETE CASCADE,
  from_name   TEXT NOT NULL DEFAULT 'Nathan',
  note        TEXT NOT NULL,
  logged_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Onboarding checklist (one row per creator per item)
CREATE TABLE onboarding_checklist (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id  UUID REFERENCES creators(id) ON DELETE CASCADE,
  label       TEXT NOT NULL,
  done        BOOLEAN DEFAULT FALSE,
  done_at     TIMESTAMPTZ
);

-- Deadlines
CREATE TABLE deadlines (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id  UUID REFERENCES creators(id) ON DELETE CASCADE,
  label       TEXT NOT NULL,
  type        TEXT NOT NULL,                   -- 'handle', 'warmup', etc.
  target_at   TIMESTAMPTZ NOT NULL,
  missed      BOOLEAN DEFAULT FALSE,
  resolved    BOOLEAN DEFAULT FALSE
);
```

---

## Computed Fields (not stored — derived at query time)

```
creator.stage →  derived from contract_signed_at, handle_confirmed_at,
                 warmup_approved_at, active_since
creator.weekly_posts.count → COUNT of content_submissions WHERE
                 submitted_at >= Monday 00:00 of current week
                 AND status != 'rejected'
```

---

## Endpoints

---

### Creators

#### `GET /creators`
Returns all creators with computed stage and weekly posting stats.

**Query params:**
| Param | Type | Description |
|---|---|---|
| `filter` | string | `all` \| `active` \| `boosted` \| `regular` \| `has_alert` |
| `sort` | string | `name` \| `stage` \| `posts` \| `last_post` \| `views` \| `checkin` |
| `dir` | string | `asc` \| `desc` |

**Response:**
```json
{
  "creators": [
    {
      "id": "uuid",
      "name": "Sofia Chen",
      "phone": "+1-415-555-0101",
      "type": "boosted",
      "color": "#7c3aed",
      "avatar_initials": "SC",
      "stage": "active",              // computed
      "contract_signed_at": "2024-11-03T00:00:00Z",
      "handle_confirmed_at": "2024-11-05T00:00:00Z",
      "warmup_approved_at": "2024-11-20T00:00:00Z",
      "active_since": "2024-12-01T00:00:00Z",
      "weekly_posts": {               // computed
        "count": 4,
        "target": 6,
        "last_submitted_at": "2024-12-19T10:00:00Z"
      },
      "total_views": 424000,          // computed: sum of approved content views
      "pending_review_count": 0,      // computed
      "feedback_count": 2,            // computed
      "active_alert_count": 1,        // computed
      "last_checkin_at": "2024-12-20T18:00:00Z",
      "created_at": "2024-11-01T00:00:00Z"
    }
  ]
}
```

---

#### `GET /creators/:id`
Full creator profile for the detail page.

**Response:**
```json
{
  "id": "uuid",
  "name": "Sofia Chen",
  "phone": "+1-415-555-0101",
  "type": "boosted",
  "color": "#7c3aed",
  "avatar_initials": "SC",
  "stage": "active",
  "posting_target": 6,
  "weekly_posts": {
    "count": 4,
    "target": 6,
    "last_submitted_at": "2024-12-19T10:00:00Z"
  },
  "timeline": [
    { "event": "Joined platform",    "done": true,  "date": "2024-11-01" },
    { "event": "Contract signed",    "done": true,  "date": "2024-11-03" },
    { "event": "Handle requested",   "done": true,  "date": "2024-11-05" },
    { "event": "Warmup started",     "done": true,  "date": "2024-11-08" },
    { "event": "Creator warmed",     "done": true,  "date": "2024-11-20" },
    { "event": "Active on platform", "done": true,  "date": "2024-12-01" }
  ],
  "deadlines": [
    {
      "id": "uuid",
      "label": "Handle request",
      "type": "handle",
      "target_at": "2024-12-21T18:00:00Z",
      "missed": false
    }
  ],
  "content": [
    {
      "id": "uuid",
      "title": "Tutorial #1",
      "submitted_at": "2024-12-10T00:00:00Z",
      "views": 125000,
      "status": "approved",
      "reviewed_by": "Nathan",
      "reviewed_at": "2024-12-10T04:00:00Z"
    }
  ],
  "feedback_log": [
    {
      "id": "uuid",
      "from_name": "Nathan",
      "note": "Great hook on Tutorial #1 — keep that energy.",
      "logged_at": "2024-12-10T05:00:00Z"
    }
  ],
  "checklist": [
    { "label": "Onboarding guide sent", "done": true,  "done_at": "2024-11-01T00:00:00Z" },
    { "label": "Nathan link shared",    "done": true,  "done_at": "2024-11-01T00:00:00Z" },
    { "label": "Sideshift link shared", "done": true,  "done_at": "2024-11-01T00:00:00Z" }
  ]
}
```

---

#### `POST /creators`
Create a new creator (when onboarding someone new).

**Request body:**
```json
{
  "name": "Devon Walsh",
  "phone": "+1-415-555-0106",
  "type": "regular",
  "color": "#4f46e5",
  "avatar_initials": "DW"
}
```

**Response:** Created creator object (same shape as GET /creators/:id)

---

#### `PATCH /creators/:id`
Update creator fields. Used for manual stage overrides and recording milestone dates.

**Request body (all fields optional):**
```json
{
  "contract_signed_at": "2024-12-18T00:00:00Z",
  "handle_requested_at": "2024-12-19T00:00:00Z",
  "handle_confirmed_at": "2024-12-19T12:00:00Z",
  "warmup_started_at": "2024-12-20T00:00:00Z",
  "warmup_approved_at": null,
  "active_since": null,
  "posting_target": 6
}
```

---

### Content Submissions

#### `GET /creators/:id/content`
All content submissions for a creator.

**Response:**
```json
{
  "submissions": [
    {
      "id": "uuid",
      "title": "Tutorial #1",
      "submitted_at": "2024-12-10T00:00:00Z",
      "views": 125000,
      "status": "approved",
      "reviewed_by": "Nathan",
      "reviewed_at": "2024-12-10T04:00:00Z"
    }
  ]
}
```

---

#### `POST /creators/:id/content`
Log a new content submission (called by Sideshift webhook or manually).

**Request body:**
```json
{
  "title": "GRWM Dec",
  "submitted_at": "2024-12-18T10:00:00Z"
}
```

---

#### `PATCH /content/:id`
Update submission status (Nathan approves/rejects).

**Request body:**
```json
{
  "status": "approved",
  "views": 210000,
  "reviewed_by": "Nathan"
}
```

---

### Feedback Log

#### `POST /creators/:id/feedback`
Log a feedback note (Nathan calls this after reviewing a video).

**Request body:**
```json
{
  "from_name": "Nathan",
  "note": "Hook is too slow — grab attention in the first 2 seconds."
}
```

**Response:**
```json
{
  "id": "uuid",
  "creator_id": "uuid",
  "from_name": "Nathan",
  "note": "Hook is too slow — grab attention in the first 2 seconds.",
  "logged_at": "2024-12-21T14:00:00Z"
}
```

---

#### `DELETE /feedback/:id`
Delete a feedback note (in case of error).

---

### Group Chats & Messages

#### `GET /chats`
All group chats with last message preview and alert status.

**Response:**
```json
{
  "chats": [
    {
      "id": "uuid",
      "creator_id": "uuid",
      "creator_name": "Sofia Chen",
      "creator_color": "#7c3aed",
      "creator_initials": "SC",
      "imessage_chat_guid": "chat-guid-from-bluebubbles",
      "participants": ["Sofia Chen", "Sarah (Befreed)", "Nathan"],
      "last_team_reply_at": "2024-12-20T18:00:00Z",
      "last_checkin_at": "2024-12-20T18:00:00Z",
      "has_alert": false,
      "last_message": {
        "sender_name": "Sofia",
        "body": "Filming another one today",
        "sent_at": "2024-12-20T19:30:00Z",
        "role": "creator"
      }
    }
  ]
}
```

---

#### `GET /chats/:id/messages`
Full message history for a group chat.

**Query params:**
| Param | Type | Description |
|---|---|---|
| `limit` | int | Messages to return (default 100) |
| `before` | timestamp | Pagination cursor |

**Response:**
```json
{
  "messages": [
    {
      "id": "uuid",
      "chat_id": "uuid",
      "role": "team",
      "sender_name": "Sarah",
      "body": "Hey Sofia! Welcome to Befreed 🎉",
      "sent_at": "2024-12-12T10:00:00Z"
    }
  ],
  "has_more": false
}
```

---

#### `POST /chats/:id/messages`
Send a message to a group chat via iMessage bridge.

**Request body:**
```json
{
  "sender_name": "Sarah",
  "body": "Hey! Just checking in 🙌"
}
```

**What the backend does:**
1. Calls BlueBubbles API to send the iMessage
2. Stores the message in the `messages` table
3. Updates `group_chats.last_team_reply_at` and `last_checkin_at`
4. Auto-resolves any `reply_gap` alert for this creator

**Response:** Created message object

---

#### `POST /chats/bulk-checkin`
Send the daily check-in message to multiple chats at once (used by Daily Check-in screen).

**Request body:**
```json
{
  "chat_ids": ["uuid1", "uuid2", "uuid3"],
  "message": "Hey! Just checking in — how's everything going? 🙌"
}
```

**Response:**
```json
{
  "sent": ["uuid1", "uuid2"],
  "failed": ["uuid3"],
  "errors": { "uuid3": "BlueBubbles delivery failed" }
}
```

---

### Alerts

#### `GET /alerts`
All active (unresolved) alerts sorted by severity.

**Query params:**
| Param | Type | Description |
|---|---|---|
| `creator_id` | uuid | Filter to one creator |
| `type` | string | Filter by alert type |

**Response:**
```json
{
  "alerts": [
    {
      "id": "uuid",
      "creator_id": "uuid",
      "creator_name": "Jade Kim",
      "type": "deadline_urgent",
      "severity": "flash",
      "title": "Jade Kim — Handle Request Deadline",
      "detail": "Handle request deadline in 2 hours",
      "time_label": "2h remaining",
      "created_at": "2024-12-20T16:00:00Z"
    }
  ]
}
```

---

#### `PATCH /alerts/:id/resolve`
Mark an alert as resolved.

**Response:**
```json
{ "id": "uuid", "resolved": true, "resolved_at": "2024-12-20T18:00:00Z" }
```

---

### Onboarding Checklist

#### `PATCH /creators/:id/checklist/:item_id`
Mark a checklist item as done.

**Request body:**
```json
{ "done": true }
```

---

### Deadlines

#### `POST /creators/:id/deadlines`
Create a deadline for a creator.

**Request body:**
```json
{
  "label": "Handle request",
  "type": "handle",
  "target_at": "2024-12-21T18:00:00Z"
}
```

---

#### `PATCH /deadlines/:id`
Update a deadline (mark missed, resolve, etc.)

**Request body:**
```json
{ "missed": true }
```

---

## iMessage Bridge (BlueBubbles)

### Incoming webhook (BlueBubbles → your backend)
BlueBubbles sends a POST to your backend when a new message arrives.

**Configure in BlueBubbles:** `Settings → Webhooks → New Message → https://your-backend/webhooks/imessage`

**Payload BlueBubbles sends:**
```json
{
  "type": "new-message",
  "data": {
    "guid": "imessage-unique-guid",
    "chatGuid": "iMessage;+;chat-guid",
    "handle": { "address": "+14155550101" },
    "text": "omg thank you!! so excited",
    "isFromMe": false,
    "dateCreated": 1703030400000
  }
}
```

**What your backend does with it:**
1. Match `chatGuid` to a `group_chats.imessage_chat_guid`
2. Determine role: `isFromMe = true` → `team`, else → `creator`
3. Match sender name from `handle.address` → look up in creators or team members table
4. Insert into `messages` table
5. If role is `creator`, run alert checks (reply gap timer reset, etc.)

### Outgoing (backend → BlueBubbles)
To send a message from the CRM, your backend calls:

```
POST http://localhost:1234/api/v1/message/text
Headers: Password: your-bluebubbles-password
Body: {
  "chatGuid": "iMessage;+;chat-guid",
  "message": "Hey! Just checking in 🙌",
  "method": "apple-script"
}
```

BlueBubbles runs on the Mac mini at a local address (or tunneled via ngrok/Cloudflare tunnel for remote access).

---

## Background Jobs

These run on a schedule (cron) on the backend:

| Job | Schedule | What it does |
|---|---|---|
| `check-posting-gaps` | Every hour | For every active creator, if `last_submitted_at` > 48h ago and no `posting_gap` alert exists → create one |
| `check-reply-gaps` | Every 30 min | For every group chat, if `last_team_reply_at` > 4h ago and no `reply_gap` alert exists → create one |
| `check-deadlines` | Every 15 min | For every unresolved deadline, check if within 2h (urgent) or 18h (warning) or missed → create/update alert |
| `check-checkins` | Every hour | For every group chat, if `last_checkin_at` > 24h → surface on check-in screen (no alert created, just data) |

---

## Frontend Integration

When your developer is done, the frontend connects by replacing the fake `DATA` object with API calls. Each screen maps to specific endpoints:

| Screen | Endpoints used |
|---|---|
| Pipeline | `GET /creators` |
| Creator List | `GET /creators?sort=&filter=` |
| Alerts | `GET /alerts` · `PATCH /alerts/:id/resolve` |
| Creator Detail | `GET /creators/:id` · `POST /creators/:id/feedback` · `PATCH /creators/:id` |
| Group Chats | `GET /chats` · `GET /chats/:id/messages` · `POST /chats/:id/messages` |
| Daily Check-in | `GET /chats` · `POST /chats/bulk-checkin` |

---

## Environment Variables

```
DATABASE_URL=postgresql://...         # Supabase connection string
SUPABASE_ANON_KEY=...
API_KEY=...                           # Internal auth key for frontend
BLUEBUBBLES_URL=http://localhost:1234 # Mac mini local address
BLUEBUBBLES_PASSWORD=...
```
