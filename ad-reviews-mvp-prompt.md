# Build brief: Ad Reviews (client approval of Meta ads) — MVP

You're working in our CRM codebase. Build the first version of **Ad Reviews**: a module where our team uploads Meta ads (images or videos, plus the ad copy) for a client, the client receives an SMS with a link, previews every ad in a realistic Facebook/Instagram mock feed, and either approves each ad or sends change requests. Everything is recorded in the CRM.

Read the whole brief before starting. It explains *why* as well as *what*, so you can make good calls where it's silent.

---

## 0. How to work on this

1. **Recon first.** Explore the codebase and answer the questions in §2. Don't write feature code yet.
2. **Plan, then stop.** Write a short plan: what you found, how this maps onto existing tables and patterns, the migrations you'll add, routes and components, and open questions. Wait for my go-ahead before implementing, unless I've told you to continue without stopping.
3. **Build in the milestone order in §13**, in small reviewable commits on a feature branch. Don't push to the main branch or deploy.
4. **Follow the CRM's conventions** for structure, data access, validation, styling and tests. If this brief conflicts with how the CRM works, or looks wrong once you've seen the code, follow the CRM and raise it in your plan.
5. **Build what's in scope.** Don't add features from §16 or abstractions "for later" beyond what the data model below already allows.
6. **This is a production CRM, so:**
   - Database changes are additive only: new tables, or new nullable columns if unavoidable. Don't alter or drop existing columns or tables. Every migration must be reversible.
   - Every new table gets row-level security (or the CRM's equivalent access control) from day one.
   - No real SMS outside production. SMS defaults to dry-run mode (§10).
   - Seed data only in local/dev; never touch real client data.
   - No secrets in the repo. Ask before adding dependencies other than shadcn/ui components.
   - Put the module behind a feature flag so it can be merged without being visible to clients.

## 1. Why we're building this

We run Meta ads for many clients, and every ad needs the client's sign-off before it goes live. Today that happens through calls and messages. It eats time, and context gets lost: if a colleague agreed changes with a client by phone and is then off sick, nobody else knows exactly what was said.

So the module must:

- **Be the single source of truth.** Everything about an ad lives in one timeline inside the CRM: client feedback, approvals, phone calls logged on the client's behalf, internal notes, every version. If it isn't in the timeline, it didn't happen.
- **Need as little human effort as possible.** "Send to client" texts the client a link. The client acts without logging in. The team is only notified when there's something to do.
- **Be effortless for clients.** Very minimal and uncluttered. A client should understand the page within seconds, without instructions.
- **Make approvals unambiguous.** An approval applies to one specific version of an ad. Sending a changed version resets it.

## 2. Recon: find these out before planning

We believe the CRM runs on Supabase (Postgres, Auth, Storage) with a React/Next.js front end, Tailwind and shadcn/ui. Verify this; if it's wrong, adapt and tell me.

1. Framework, routing, data-access pattern (server actions, API routes, RPC…), validation library, and how migrations are written and run.
2. How client accounts (we call them sub-accounts) are modelled, and how team users and permissions relate to them: tenancy, roles, existing RLS policies.
3. Where client contacts and their phone numbers live.
4. Whether clients can log into the CRM themselves. If they can, the review page must also be reachable inside their sub-account, not only via the SMS link.
5. Existing SMS sending: provider, helpers, message logs, sender ID.
6. Existing team notifications: in-app, email, Slack, etc.
7. File storage: buckets, upload patterns, size limits, signed URLs, image transformations.
8. UI: is shadcn/ui set up, which components exist, theme tokens, and which layout shell the team-side pages should live in.
9. Testing (unit, integration, Playwright), linting, type-checking, CI.
10. Any background job or scheduler. Not needed for the MVP, but note it for later (automatic reminders).

## 3. Scope

**In the MVP**

- **Team (desktop):** create review rounds per client account; add ads with copy and media in up to three formats; live preview; send to the client by SMS; an activity timeline per ad; internal notes; log feedback or an approval the client gave by phone, email or in a meeting; revise ads as new versions; mark ads as published; copy and revoke review links.
- **Client:** open the SMS link without logging in (and inside their sub-account if client logins exist); browse the round's ads; preview each one in a realistic mock feed; switch platform and format; approve each ad or send change requests; see their earlier feedback and what changed in new versions.
- **System:** statuses, an audit trail, SMS sending (dry-run by default), team notifications.

**Not in the MVP:** see §16. Design so those fit later without rework.

## 4. Concepts and data model

- **Client account:** the CRM's sub-account.
- **Round:** a batch of ads sent to a client together, e.g. "November campaign".
- **Ad:** one ad in a round.
- **Version:** a snapshot of an ad's copy and media. A version is a *draft* until it's sent; once sent it never changes. Revisions create a new version.
- **Current version:** the ad's latest *sent* version. It's the only version clients see or act on.
- **Event:** one entry in an ad's timeline. Events are append-only.

Reference design (Postgres). Adapt table names, foreign keys, tenancy columns and policies to the CRM's conventions; `<accounts>`, `<users>` and `<contacts>` stand for the CRM's existing tables.

```sql
create table ad_review_rounds (
  id                uuid primary key default gen_random_uuid(),
  account_id        uuid not null references <accounts>(id) on delete cascade,
  title             text not null,
  due_at            timestamptz,                     -- optional "please review by"
  brand_name        text not null,                   -- page name shown in the mockups
  instagram_handle  text,                            -- shown in Instagram mockups
  brand_avatar_path text,                            -- profile picture in the mockups
  status            text not null default 'draft'
                    check (status in ('draft','in_review','approved','archived')),
  owner_user_id     uuid not null references <users>(id),
  first_sent_at     timestamptz,
  approved_at       timestamptz,
  created_at        timestamptz not null default now(),
  updated_at        timestamptz not null default now()
);

create table ad_review_round_recipients (
  round_id   uuid not null references ad_review_rounds(id) on delete cascade,
  contact_id uuid not null references <contacts>(id),
  primary key (round_id, contact_id)
);

create table ad_review_ads (
  id           uuid primary key default gen_random_uuid(),
  round_id     uuid not null references ad_review_rounds(id) on delete cascade,
  name         text not null,                        -- e.g. "Black Friday – offer"
  position     int  not null,
  status       text not null default 'draft'
               check (status in ('draft','in_review','changes_requested','approved','published')),
  published_at timestamptz,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);

create table ad_review_versions (
  id              uuid primary key default gen_random_uuid(),
  ad_id           uuid not null references ad_review_ads(id) on delete cascade,
  version_number  int  not null,
  primary_text    text not null default '',
  headline        text not null default '',
  description     text not null default '',
  cta             text,                              -- stable code, e.g. 'LEARN_MORE'; null = no button
  destination_url text,
  display_link    text,                              -- optional; otherwise derived from destination_url
  change_summary  text,                              -- client-visible "what changed" (v2+)
  created_by      uuid not null references <users>(id),
  created_at      timestamptz not null default now(),
  sent_at         timestamptz,                       -- null = draft
  unique (ad_id, version_number)
);
-- at most one draft per ad
create unique index ad_review_versions_one_draft
  on ad_review_versions (ad_id) where sent_at is null;

create table ad_review_assets (
  id               uuid primary key default gen_random_uuid(),
  version_id       uuid not null references ad_review_versions(id) on delete cascade,
  format           text not null check (format in ('1x1','4x5','9x16')),
  media_type       text not null check (media_type in ('image','video')),
  storage_path     text not null,                    -- may be shared by several versions
  mime_type        text not null,
  width            int,
  height           int,
  duration_seconds numeric,
  size_bytes       bigint,
  poster_path      text,
  created_at       timestamptz not null default now(),
  unique (version_id, format)
);

-- The timeline. Append-only: app code never updates or deletes rows.
create table ad_review_events (
  id               uuid primary key default gen_random_uuid(),
  round_id         uuid not null references ad_review_rounds(id) on delete cascade,
  ad_id            uuid references ad_review_ads(id) on delete cascade,      -- null = round-level
  version_id       uuid references ad_review_versions(id) on delete set null,
  type             text not null check (type in (
                     'ad_created','version_created','version_discarded','version_sent',
                     'approved','approval_withdrawn','changes_requested',
                     'internal_note','marked_published','round_approved',
                     'link_created','link_revoked','sms_sent','sms_failed')),
  visibility       text not null check (visibility in ('client','internal')),
  body             text,                             -- feedback, note, or change summary
  actor_type       text not null check (actor_type in ('team_user','client_contact','system')),
  actor_user_id    uuid references <users>(id),      -- team actions, incl. logging on a client's behalf
  actor_contact_id uuid references <contacts>(id),   -- the client contact (direct, or on whose behalf)
  channel          text check (channel in ('review_link','client_portal','phone','email','meeting','other')),
  ip_address       inet,
  user_agent       text,
  metadata         jsonb not null default '{}',
  created_at       timestamptz not null default now()
);

create table ad_review_links (
  id           uuid primary key default gen_random_uuid(),
  round_id     uuid not null references ad_review_rounds(id) on delete cascade,
  contact_id   uuid not null references <contacts>(id),
  token_hash   text not null unique,                 -- SHA-256 of the raw token
  expires_at   timestamptz not null,
  revoked_at   timestamptz,
  last_used_at timestamptz,
  created_by   uuid references <users>(id),
  created_at   timestamptz not null default now()
);

-- Only if the CRM has no message log of its own
create table ad_review_sms (
  id                  uuid primary key default gen_random_uuid(),
  round_id            uuid not null references ad_review_rounds(id) on delete cascade,
  contact_id          uuid not null references <contacts>(id),
  to_phone            text not null,                 -- E.164
  body_redacted       text not null,                 -- message text with the token removed
  mode                text not null check (mode in ('dry_run','live')),
  status              text not null check (status in ('queued','sent','failed')),
  provider_message_id text,
  error               text,
  idempotency_key     text not null unique,
  created_at          timestamptz not null default now()
);
```

Rules the data model must uphold:

- Drafts (`sent_at is null`) never reach a client. Neither do ads without a sent version.
- Client-visible event types are `version_sent`, `approved`, `approval_withdrawn`, `changes_requested` and `marked_published`. Everything else is `internal`.
- The events table is append-only. Where the CRM's setup allows, enforce that with policies (insert and select only).
- A new version starts as a copy of the previous one: same text, and asset rows pointing at the same storage files. Changing only the text never needs a re-upload.
- Discarding a draft deletes that version and writes a `version_discarded` event.
- Only ads that were never sent can be deleted. Sent ads stay in the record.
- Raw link tokens are never stored, only their SHA-256 hash. Every SMS send and every "Copy link" creates a new link row. Earlier links stay valid until they expire or are revoked, so the team can copy a fresh link any time without breaking one already sent.

## 5. Status rules

Keep all transition logic in one module and run each transition atomically (one transaction or one Postgres function): insert the event, then update the ad and round statuses. The timeline is the source of truth; statuses are kept in sync with it.

**Ad status**

| Status | Meaning |
|---|---|
| `draft` | Never sent; invisible to the client |
| `in_review` | The current version is waiting for the client |
| `changes_requested` | The client sent feedback on the current version |
| `approved` | The client approved the current version |
| `published` | The team marked it live; the client's controls lock |

- Ad created → `draft`.
- A version is sent (first send or an update) → `in_review`. An earlier approval no longer counts, because it belonged to the earlier version.
- Client approves the current version → `approved`. Client withdraws the approval → `in_review`.
- Client sends feedback on the current version → `changes_requested`.
- The latest decision on the current version wins. A client may approve after asking for changes. To send feedback on an approved ad, they first switch the approval off, and that is recorded too.
- Team creates a draft revision → no status change. The client keeps seeing the current version until the draft is sent.
- Team marks an `approved` ad as published → `published`. This is final in the MVP; to change a published ad, the team creates a new ad.

**Round status**

- `draft` until the first send.
- `in_review` while any sent ad is `in_review` or `changes_requested`.
- `approved` when every sent ad is `approved` or `published` (unsent drafts are ignored). Entering `approved` writes one `round_approved` event and notifies the team once.
- Back to `in_review` if an approval is withdrawn or an update is sent.
- `archived` is set manually. It hides the round from default lists, and its links stop working.

## 6. UI principles (every screen in this module)

- **Use shadcn/ui throughout:** Button, Card, Badge, Textarea, Input, Label, Switch, ToggleGroup, Tabs, Dialog, AlertDialog, Select, Tooltip, ScrollArea, Separator, Skeleton, Avatar, and Sonner for toasts. If shadcn/ui isn't set up in the CRM, flag it in your plan; don't introduce a second UI library.
- **Minimal and calm.** Neutral background, one accent colour, generous whitespace. Nothing on screen that doesn't help the decision: no dashboards, charts or extra metadata.
- **Status colours, used consistently everywhere:** waiting is neutral or blue-grey, changes requested is amber, approved is green, published is a muted outline.
- **Desktop first.** Three panels: the left about 300px, the right about 360px, the center flexible. Panels scroll independently. The phone mockup scales to fit the viewport height, so it's fully visible at 1280×800.
- Use the CRM's theme tokens.
- **Accessible:** everything reachable by keyboard, labelled controls, visible focus, sufficient contrast, alt text.
- **Copy:** keep all user-facing text in one place, in the CRM's UI language and i18n setup. Show dates in the viewer's locale and timezone.

## 7. Team experience (desktop, inside the client's sub-account)

Add an **Ad reviews** section to each client sub-account.

**Rounds list.** Title, status badge, progress ("3 of 5 approved"), change requests waiting (highlighted), last activity and owner, plus a **New review round** button.

**New round dialog.**

- Title and an optional due date.
- Brand name, prefilled from the account.
- Instagram handle (optional). If it's empty, derive one from the brand name.
- Profile picture: the account's logo if the CRM stores one; otherwise an upload; otherwise initials.
- Recipients: the account's contacts, with phone numbers. Warn about contacts without a valid number.

**Round workspace.** The same three-panel shell as the client page (§8), so the team sees what the client will see, plus team controls.

- **Top bar:** round title, status, recipients and due date, with three buttons:
  - **Preview as client** opens a read-only client view.
  - **Copy link**: pick a recipient, and a fresh link is created.
  - **Send to client (n)**, where n is the number of unsent drafts.
- **Left:** the ad stack, with extra badges for "Draft" and "Unsent changes"; **+ Add ad**; reordering (up/down buttons are fine).
- **Center:** the same mockup and toggles the client sees. When the ad has a draft, show the draft, labelled "Draft v3 — not sent".
- **Right**, two tabs:
  - **Edit.** If the ad has a draft, show an editable form. Otherwise show a read-only view of the current version, with **Create new version** (plus **Mark as published** when the ad is approved). The form has:
    - Ad name
    - Primary text (multi-line, emojis allowed)
    - Headline
    - Description
    - Call to action (a select, including "No button")
    - Destination URL (validated)
    - Display link (optional)
    - Media: three slots, "1:1 Feed (1080×1080)", "4:5 Feed (1080×1350)" and "9:16 Stories & Reels (1080×1920)". Each takes an image or a video. At least one slot must be filled before the ad can be sent.
    - What changed (client-visible, required from v2 onwards; e.g. "Shortened the headline as you asked")
    - Discard draft

    Show character counters with soft guidance that never blocks: roughly the first 125 characters of primary text show before "See more", headlines around 40, descriptions around 30. Keep these numbers in config. Follow the CRM's saving pattern; if there isn't one, autosave on a short debounce and show a "Saved" state.
  - **Activity.** The ad's timeline as a chronological thread, newest at the bottom like a conversation.
    - Each entry shows who (team member, client contact or System), what happened, when, the version, the channel (e.g. a "Phone" badge) and any text. Internal entries are clearly marked "Internal".
    - At the bottom: a box for internal notes.
    - **Log client response** opens a dialog with the contact, the channel (phone / email / meeting / other), the type (feedback or approval) and a required note of what the client said. It records the decision exactly as if the client had made it, attributed to the staff member and the channel. This is how we stop context living in people's heads.

**Send dialog.**

- Lists the recipients with their phone numbers (each can be unticked) and what will be sent ("2 new ads, 1 updated ad").
- Shows the SMS text, editable, with the link placeholder kept, and says clearly whether this is a dry run or live.
- If no recipient has a valid number, sending is disabled with the reason, and **Copy link** remains available.
- On confirm: mark every draft in the round as sent, recompute statuses, create one link per recipient, send the SMS, and show the result per recipient. In dry-run mode, show the generated links so we can test.

## 8. Client experience

**Access**

- **SMS link:** `{APP_URL}/r/{token}`. Keep it short; it goes in a text message. No login. After validating the token you may swap it for a short-lived httpOnly session cookie scoped to that round. Either way, re-check expiry and revocation on every action.
- **Inside the sub-account,** if clients can log into the CRM: the same review page, with actions attributed to their contact.
- **A logged-in team member who opens a client link** gets a read-only page with a banner: "Team preview — actions are disabled". Staff must never act as the client by accident; they use **Log client response** instead.

**Desktop layout**

- **Header:** agency logo and name; round title; the due date, if set ("Please review by Thu 21 Nov"); progress ("2 of 5 approved", where published counts as approved); and a small "Reviewing as Anna".
- **Left panel:** a vertical stack of ad cards, each with a thumbnail, the ad name and a status badge ("Waiting for you", "Changes requested", "Approved", "Published"). Clicking a card loads that ad in the center.
- **Center:** the toggles (§9), then the mockup in a phone frame, centered, then a one-line disclaimer.
- **Right panel**, top to bottom:
  1. Ad name, version badge (e.g. "v2") and status.
  2. From v2 onwards, a "What changed" callout.
  3. **Requested edits:** a text box (placeholder "What would you like us to change?") and a **Send feedback** button.
  4. **Your feedback:** the client's earlier feedback on this ad, with dates and version labels. Entries logged by staff read "Logged by Sara from a phone call".
  5. Separated at the bottom: the **Approve** switch, labelled "Approve — ready to publish", with the helper text "No changes needed".

When every ad is approved, show a short banner: "All ads approved — thank you! We'll take it from here."

**Right-panel behaviour.** This is a core requirement; implement it exactly.

| Situation | Requested edits box | Send feedback button | Approve switch |
|---|---|---|---|
| Default: not approved, box empty | Enabled | Disabled (greyed out) | Off |
| Client has typed text (not just spaces) | Enabled | Enabled | Off |
| Switch turned on | Disabled (greyed out); typed text is kept | Disabled, even if text was typed | On |
| Switch turned off again | Enabled; the kept text is back | Enabled if there's text | Off |
| Feedback sent | Cleared; the note appears under "Your feedback" | Disabled | Off, still usable |
| Saving | Disabled | Disabled, with a spinner | Disabled |
| Ad published | Disabled | Disabled | Disabled, showing the final state |

- The switch saves immediately (approve or withdraw), with no confirmation dialog. Update the UI optimistically, show a short toast, and roll back with an error toast if saving fails.
- Unsent text is kept per ad when switching between ads, and survives a reload (local storage keyed by link and ad).
- When a new version arrives, the switch starts off. The "What changed" callout and the earlier feedback give the context.

**Edge states**

- An invalid, expired, revoked or archived link shows one neutral page: "This review link is no longer valid. Please contact {agency} for a new one." It reveals nothing about the round, and the wording is the same whatever the reason.
- A round with nothing sent yet shows "Nothing to review yet."
- Media that won't load or play gets a clear placeholder, never a broken layout.

**Mobile.** Desktop is the primary design target, but the link arrives by SMS, so most clients will first open it on a phone. In the MVP the client page must therefore be *usable* at 390px width. Use a single column: the header, a horizontally scrolling ad selector, the toggles, the mockup scaled to the screen, then the feedback box and switch. Polish comes later.

## 9. The ad mockups

Build **one presentational component** (e.g. `AdMockup`) that both the team editor and the client page use, so the team preview always matches what the client sees. Add a dev-only gallery page that renders every platform and placement from seed data, for fast visual checks.

**Toggles above the mockup** (shadcn ToggleGroup, small and quiet)

- **Platform:** Instagram | Facebook.
- **Format:** "1:1 Feed" | "4:5 Feed" | "9:16 Stories & Reels". When 9:16 is active, a small secondary toggle appears: Reels | Stories (default Reels).
- Formats the ad has no asset for are disabled, with a tooltip ("Not included in this ad").
- The default format is 4:5 if present, then 1:1, then 9:16. Keep the chosen platform and format when switching ads, if the next ad has that format.

**Layouts.** Aim for "recognisably a Facebook/Instagram ad", not a pixel copy. Use generic icons (lucide-react), and no Meta logos or copied brand assets.

- **Feed (both platforms):** a phone frame with a plain top bar. The ad sits between neutral grey skeleton posts above and below, so it reads as "in the feed". Scroll the ad into view on load.
- **Facebook feed**, top to bottom:
  1. Header: avatar, page name, and "Sponsored" with a globe icon.
  2. Primary text, *above* the media, clamped to 3 lines with a "… See more" that expands on click.
  3. Media.
  4. Link bar: display link (uppercase, small, muted), headline (bold), description (muted, one line), and the CTA button on the right.
  5. Like / comment / share row.
- **Instagram feed**, top to bottom:
  1. Header: avatar, handle, "Sponsored".
  2. Media.
  3. A full-width CTA bar under the media, with the CTA label and a chevron.
  4. Action row: heart, comment, share, bookmark.
  5. Caption: the handle in bold, then the primary text clamped to 2 lines with "… more".

  Headline and description are hidden by default.
- **Reels (9:16):**
  - Full-bleed media.
  - Bottom-left overlay: avatar, name and "Sponsored"; the primary text clamped to 2 lines with "… more"; the CTA button below.
  - Generic action icons down the right edge.
  - A gradient behind the text for legibility.
- **Stories (9:16):**
  - Full-bleed media.
  - At the top: a progress bar, plus avatar, name and "Sponsored".
  - At the bottom: a CTA pill.
  - No primary text by default.
- For 9:16, Facebook and Instagram share these layouts; the platform toggle switches the name (page name or Instagram handle).

**Configurable rules.** Keep per-placement rules in one config object: which fields show, line clamps, the "See more" or "more" label, CTA style and position, and the character guidance from §7. Meta changes its UI often, and we want to adjust it in one place.

**Content**

- Keep line breaks and emojis. Never render user text as HTML.
- If the display link is empty, show the destination URL's domain.
- Video: muted, autoplay, looping and inline, with a small mute/unmute button and a poster frame if one exists.
- CTA codes map to labels in config: No button, Learn more, Shop now, Sign up, Book now, Contact us, Get offer, Get quote, Order now, Apply now, Download, Subscribe, Send message, Call now. Store stable codes like `LEARN_MORE` so we can map them to Meta's API later.
- Disclaimer under the mockup: "Preview is approximate. Facebook and Instagram may display text slightly differently depending on device and placement."

## 10. SMS and team notifications

**SMS**

- Use the CRM's existing SMS integration if there is one. If not, create a small `sendSms({ to, body })` interface with two adapters: a dry-run adapter that records the message and never sends, and one live adapter. In your plan, ask me which provider to use (e.g. Twilio or 46elks).
- `AD_REVIEW_SMS_MODE=dry_run|live`, defaulting to `dry_run`. Only I switch production to live.
- Use the sender ID or number the CRM already uses; otherwise make it configurable.
- Normalise phone numbers to E.164. Skip contacts without a valid number and report them in the send dialog.
- **Sends are idempotent.** The button disables on click and the server uses an idempotency key, so a double-click never sends twice.
- Log every message with the token redacted. Show failures with a retry button.
- **Default templates** are editable in the send dialog. Keep all copy in one place, in the CRM's UI language. Stay under 160 characters and avoid emojis, which cut the per-SMS character limit.
  - First send: `Hi {firstName}! Your new ads from {agencyName} are ready for review: {link}`
  - Update: `Hi {firstName}! We've updated {count} ad(s) based on your feedback: {link}`

**Team notifications**

- Use the CRM's notification system (in-app, plus email or Slack if it exists).
- Notify the round owner, and the account's assigned team members if the CRM tracks them, when:
  - a client sends feedback;
  - a client withdraws an approval;
  - a client approves an ad that had changes requested;
  - a round becomes fully approved ("Ready to publish").
- Don't notify on each individual approval; those show in the timeline and the rounds list. Never notify the person who performed the action.
- Each notification includes the client account, round, ad, what happened, a short excerpt and a deep link to the ad's Activity tab.

## 11. Security and privacy

- **Tokens:** 32 random bytes from a cryptographically secure generator, base64url-encoded. Store only the SHA-256 hash. Default expiry is 30 days (`AD_REVIEW_LINK_TTL_DAYS`), and links are revocable. Update `last_used_at` on use.
- **All client access goes through server-side code.** That code validates the token (or the client's portal session) and then queries strictly within that round. Never trust an ad, version or round id from the browser without checking it belongs to the authorised round.
- **Client responses are built from an explicit data shape** (an allow-list), never by passing database rows through. The shape below is the *only* data a client may receive. Internal events, drafts, unsent ads, other rounds and other accounts never leave the server.

  ```ts
  type ClientReview = {
    agency: { name: string; logoUrl?: string }
    round: {
      title: string; dueAt?: string; status: string
      brand: { name: string; instagramHandle?: string; avatarUrl?: string }
    }
    viewer: { firstName: string; readOnly: boolean }
    ads: Array<{
      id: string; name: string; position: number; status: string; publishedAt?: string
      version: {
        id: string; number: number
        primaryText: string; headline: string; description: string
        cta: string | null; displayLink: string; changeSummary?: string
        assets: Partial<Record<'1x1' | '4x5' | '9x16', {
          type: 'image' | 'video'; url: string; posterUrl?: string
          width?: number; height?: number
        }>>
      }
      approval: { approved: boolean; at?: string; byName?: string; channel?: string }
      feedback: Array<{
        versionNumber: number; body: string; at: string; byName: string; channel: string
      }>
    }>
  }
  ```

- **Row-level security on every new table.** Team access follows the CRM's existing account permissions, and there is no anonymous table access. If the public page needs elevated database access (e.g. a Supabase service role), use it only on the server and only after validating the token. Never expose it to the browser.
- **Storage:** a private bucket. Files are reachable only through short-lived signed URLs (about an hour), generated after authorisation. Validate file type and size server-side. No SVG.
- **Review page headers:** `Referrer-Policy: no-referrer` and `X-Robots-Tag: noindex, nofollow`. No third-party scripts on the page.
- **Rate-limit** the public review endpoints, per IP and per token.
- **Audit:** every decision stores the actor, channel, version, timestamp, IP and user agent.
- **Logs:** never log raw tokens or full phone numbers.

## 12. Media handling

- Use a storage path convention, e.g. `ad-reviews/{accountId}/{roundId}/{uuid}.{ext}`.
- Upload straight from the browser to storage (signed upload URLs; resumable uploads for big videos). Don't stream files through the app server.
- **Accepted files:** JPG, PNG and WebP images; MP4 and MOV videos.
  - Keep size limits in config. Start with 30 MB for images and 500 MB for video, within whatever the storage plan allows; tell me if the plan's limit is lower.
  - Recommend MP4 (H.264). Some MOV files (HEVC) won't play in Chrome; if a file can't play in the browser, say so clearly in the editor.
- On upload, read the dimensions, and the duration for video. If the aspect ratio is more than about 2% off the slot, show a warning but don't block.
- Lazy-load media. For stack thumbnails, use storage image transformations if available; otherwise scale with CSS.
- Discarding a draft doesn't delete files that other versions still reference. Orphan cleanup comes later.

## 13. Milestones

1. Recon and plan, then **stop for approval**.
2. Schema, policies, storage bucket, the transition module with unit tests, and a seed script. The seed creates one account with one round of 3–4 ads, covering images, a video, a missing format, and a v2 with a change summary.
3. `AdMockup` with every layout, plus the dev gallery.
4. Team side: rounds list, new round, round workspace, ad editor with uploads and live preview.
5. Client page (token and portal access). Read-only first, then decisions and feedback per the behaviour table, then the mobile layout.
6. Activity timeline, internal notes, **Log client response**, notifications.
7. Versions (creating and discarding drafts, sending updates), the send dialog, and SMS (dry-run first, then the live adapter).
8. Hardening: security tests, empty and error states, an accessibility pass, docs.

## 14. Acceptance criteria

**Team**

- [ ] I can create a round in a client sub-account, add ads with copy and media in one to three formats, reorder them, and see a live preview identical to the client's view.
- [ ] "Send to client" does all of the following:
  - sends one SMS per selected recipient with a working link (in dev, dry-run mode shows the links);
  - marks drafts as sent;
  - updates statuses and the timeline.
- [ ] An ad's Activity tab shows every event in order, with who, when, channel and version. Internal notes are marked as internal.
- [ ] I can log feedback or an approval the client gave by phone, email or in a meeting. It changes the status and shows the staff member and channel.
- [ ] I can create v2 of an ad: text and media carry over, "What changed" is required, and the client keeps seeing v1 until I send the update.
- [ ] I can mark an approved ad as published, and copy or revoke links.

**Client**

- [ ] The SMS link opens without a login and shows only that round's sent ads, each at its current version.
- [ ] On desktop there are three panels. Clicking an ad in the left stack shows it in the center, the platform and format toggles switch the mockup, and formats without an asset are disabled.
- [ ] The right panel follows the behaviour table exactly.
- [ ] Approving or withdrawing saves immediately and creates an event with IP and user agent.
- [ ] Sending feedback does all of the following:
  - sets the ad to "Changes requested";
  - clears the box;
  - lists the note under "Your feedback";
  - adds it to the team timeline;
  - notifies the owner.
- [ ] After an update, the client sees v2 with the switch off, plus the "What changed" note and earlier feedback.
- [ ] When every ad is approved, the round becomes `approved`, the client sees the thank-you banner, and the team gets one notification.
- [ ] The page is usable on a 390px-wide phone.

**Security and quality**

- [ ] Automated tests prove that client responses never include internal events, drafts, unsent ads, or other rounds' or accounts' data.
- [ ] Invalid, expired, revoked and archived links all show the same neutral page.
- [ ] A logged-in team member who opens a client link gets the read-only preview.
- [ ] Double-clicking "Send" sends only one SMS per recipient.
- [ ] Lint, type-check and all tests pass; migrations apply and roll back cleanly.

## 15. Tests to write

- **Unit:**
  - the transition module, table-driven: every event in every status, for ads and rounds;
  - token generation and validation: valid, expired, revoked, unknown;
  - the right-panel state logic, as a pure function or as component tests covering every row of the behaviour table.
- **Integration/API:**
  - the client data shape never leaks internal data (§14);
  - team endpoints respect account permissions;
  - **Log client response** requires a team session;
  - sends are idempotent.
- **End-to-end** (if the CRM has Playwright):
  1. Team creates a round, adds an ad with an image, and sends it (dry-run).
  2. The link opens in a fresh browser context.
  3. The client approves one ad and requests changes on another.
  4. Team sees the timeline and the notification, creates v2 and sends the update.
  5. The client sees v2 with the switch reset, and approves it.
  6. The round becomes approved.

## 16. Not in the MVP (design so these fit later)

- Automatic reminders and escalation, e.g. an SMS after 24h and 48h without a reply and a flag to the owner at 72h. This needs a scheduled job that finds rounds waiting on the client.
- Capturing the client's SMS replies into the right timeline.
- AI assistance (e.g. with Claude):
  - turning vague feedback into a concrete change list per ad and field;
  - drafting revised copy for the team to accept;
  - summarising calls into the timeline;
  - pre-checking copy for truncation and Meta policy risks.
- Comments on specific fields, and clients suggesting replacement text directly.
- An "Approve all" button for clients.
- Multiple text options per ad; carousel and collection ads.
- 9:16 safe-zone overlays for the team.
- Meta Marketing API: pushing approved versions to Meta as paused ads, and real Meta-rendered previews.
- Post-launch performance in the same view.

## 17. When you're done, hand back

- What you built, and any deviations from this brief with the reasons.
- How to run it locally, the new environment variables, and the migrations added.
- How to test the full flow end to end in dry-run mode.
- Screenshots: the client page on desktop and on a phone, the team workspace, and the Activity tab.
- Known gaps and suggested next steps.
