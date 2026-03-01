# cautious-octo-invention

A practical blueprint for building an 18+ AI character chat app with user-generated personas, avatar uploads, moderation, and subscription-ready architecture.

## Product guardrails

- **18+ only** with age gate and policy acceptance.
- **No illegal content** and no platform-bypass behavior.
- **User creativity first**: custom characters, private/public visibility, persona controls.
- **Moderation by design**: uploads, prompts, and generated output are checked.

---

## 1) Database schema (PostgreSQL)

> Assumes Supabase/Postgres with UUID PKs and `created_at` / `updated_at` timestamps.

### `users`
- `id uuid pk` (auth user id)
- `display_name text`
- `birth_date date` (for age checks)
- `is_age_verified boolean default false`
- `safety_strikes int default 0`
- `subscription_tier text default 'free'` (`free`, `pro`)

### `characters`
- `id uuid pk`
- `owner_id uuid fk -> users.id`
- `name text not null`
- `tagline text`
- `description text`
- `personality jsonb` (style sliders / traits)
- `scenario text`
- `example_dialogue jsonb`
- `avatar_url text`
- `visibility text default 'private'` (`private`, `unlisted`, `public`)
- `is_nsfw boolean default false`
- `status text default 'active'` (`active`, `pending_review`, `removed`)

### `character_assets`
- `id uuid pk`
- `character_id uuid fk -> characters.id`
- `asset_type text` (`avatar`, `gallery`)
- `url text`
- `moderation_label jsonb`
- `status text default 'pending'` (`pending`, `approved`, `rejected`)

### `conversations`
- `id uuid pk`
- `user_id uuid fk -> users.id`
- `character_id uuid fk -> characters.id`
- `title text`
- `mode text default 'roleplay'` (`roleplay`, `story`, `companion`)
- `is_archived boolean default false`

### `messages`
- `id uuid pk`
- `conversation_id uuid fk -> conversations.id`
- `sender text` (`user`, `assistant`, `system`)
- `content text`
- `token_count int`
- `moderation jsonb`
- `blocked boolean default false`

### `memory_items`
- `id uuid pk`
- `conversation_id uuid fk -> conversations.id`
- `key text`
- `value text`
- `confidence real`
- `source_message_id uuid fk -> messages.id`

### `reports`
- `id uuid pk`
- `reporter_id uuid fk -> users.id`
- `target_type text` (`character`, `message`, `user`)
- `target_id uuid`
- `reason text`
- `notes text`
- `status text default 'open'` (`open`, `triaged`, `resolved`)

### `audit_events`
- `id uuid pk`
- `actor_id uuid`
- `event_type text`
- `target_type text`
- `target_id uuid`
- `metadata jsonb`

### Suggested indexes
- `characters (visibility, is_nsfw, status)`
- `conversations (user_id, updated_at desc)`
- `messages (conversation_id, created_at)`
- `reports (status, created_at)`

---

## 2) API route map (Next.js App Router)

### Auth / account
- `POST /api/auth/verify-age`
  - Input: date of birth + policy consent
  - Output: `is_age_verified = true|false`

### Character management
- `POST /api/characters`
- `PATCH /api/characters/:id`
- `GET /api/characters/:id`
- `GET /api/characters?visibility=public&query=...`
- `POST /api/characters/:id/assets` (signed upload URL)
- `POST /api/characters/:id/publish`

### Chat
- `POST /api/conversations`
- `GET /api/conversations`
- `GET /api/conversations/:id/messages`
- `POST /api/conversations/:id/messages`
  1. Validate auth + age verification
  2. Moderate user input
  3. Build prompt (system + persona + memory + recent turns)
  4. Generate model response (streaming)
  5. Moderate output
  6. Persist messages + memory extraction

### Moderation / trust & safety
- `POST /api/moderation/check-text`
- `POST /api/moderation/check-image`
- `POST /api/reports`
- `POST /api/admin/reports/:id/resolve`

### Billing (optional)
- `POST /api/billing/create-checkout`
- `POST /api/billing/webhook`

---

## 3) Prompt templates

### System prompt (base)
```txt
You are roleplaying as the selected character profile.
Follow the character's personality, voice, and scenario.
Do not generate disallowed content per platform policy.
If a request is disallowed, refuse briefly and continue the scene safely.
Stay in-character unless user explicitly asks for meta discussion.
```

### Character persona injection
```txt
[Character Name]: {{name}}
[Tagline]: {{tagline}}
[Description]: {{description}}
[Personality JSON]: {{personality}}
[Scenario]: {{scenario}}
[Example Dialogue]: {{example_dialogue}}
```

### Memory block
```txt
Relevant memory from prior conversation:
{{memory_items_ranked}}
Use memory naturally; do not invent facts outside memory/context.
```

### Refusal fallback style
```txt
I can’t help with that request. Want to continue the roleplay in another direction?
```

---

## 4) Safety architecture (practical)

1. **Pre-input checks**: user text moderation before model call.
2. **Pre-output checks**: evaluate draft output and regenerate or refuse if needed.
3. **Image checks**: block non-consensual, illegal, or policy-violating uploads.
4. **Rate limits**: per-user and per-IP message caps.
5. **Human review path**: reports queue + admin tools.
6. **Auditability**: immutable moderation/audit events.

---

## 5) 7-day MVP build plan

### Day 1
- Scaffold Next.js + Supabase project.
- Implement auth and age-gate flow.

### Day 2
- Create schema + migrations.
- Add row-level security policies (owner-only private data).

### Day 3
- Build character creator UI + avatar upload.
- Add text/image moderation checks.

### Day 4
- Implement conversation + streaming chat endpoint.
- Add prompt assembly (system + persona + memory).

### Day 5
- Build conversation UI + message persistence.
- Add search/public discovery for characters.

### Day 6
- Reports, blocking, and basic admin moderation dashboard.
- Add rate limits and abuse controls.

### Day 7
- QA pass, analytics events, and deployment hardening.
- Optional Stripe paywall for pro features.

---

## 6) MVP acceptance criteria

- 18+ verification enforced before chat.
- Users can create/edit/publish characters.
- Users can upload avatars that pass moderation.
- Chats stream responses and save history.
- Report flow works end-to-end.
- Basic safety checks run on both input and output.

---

## 7) What to do next (immediate, practical)

If you are asking "what do I do now?", start here:

1. **Create your project shell (today)**
   - Create a Next.js app and a Supabase project.
   - Set environment variables for DB, auth, storage, and AI API key.

2. **Implement non-negotiables first**
   - Age gate before account usage.
   - Basic moderation checks on input + output.
   - Report endpoint and admin review flow.

3. **Build only the smallest useful flow**
   - Sign up → pass 18+ gate → create character → upload avatar → chat with that character.
   - Skip billing/admin polish until this flow works end-to-end.

4. **Ship a private alpha**
   - Invite 5–10 trusted testers.
   - Track: daily active users, messages/session, report rate, and block rate.

5. **Use this launch checklist before going public**
   - [ ] Terms, privacy policy, and community rules are visible.
   - [ ] Logging and audit events are enabled.
   - [ ] Abuse rate limits are configured.
   - [ ] Character/content takedown path is tested.
   - [ ] Backups + rollback process are documented.

### Suggested first implementation order

1. Schema migration files for `users`, `characters`, `conversations`, `messages`.
2. `/api/auth/verify-age`.
3. `/api/characters` create/edit/list.
4. `/api/conversations/:id/messages` with streaming.
5. Moderation endpoints and report flow.

### If you want help from me in the next step

Ask for one of these and I can generate it directly:
- "Write the SQL migrations"
- "Scaffold the Next.js API routes"
- "Create the character creator page"
- "Implement streaming chat handler"
