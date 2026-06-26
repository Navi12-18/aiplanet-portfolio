# How I Build with AI Agents — Portfolio Sample

**Author:** Naivaidya Yadav  
**Email:** naivaidyayadavin@gmail.com  
**GitHub:** github.com/Navi12-18  
**Date:** June 26, 2026  
**Primary tool:** Claude Code (daily), Cursor (Composer / Agent mode)  
**Projects covered:** Accale.ai (production SaaS) and The Focus Arena (indie SaaS)

> All secrets, API keys, database credentials, and webhook tokens have been removed or redacted from this document.

---

## Who I am

I am a full-stack engineer with about two years of production experience. I was the sole developer at RootRS, building and running a government NGO platform for 20,000+ users with no senior engineer above me. Since February 2026 I have been Technical Co-Founder at Accale.ai, building an AI marketing automation platform from scratch with 40% equity.

I use Claude Code every single day. Not as a novelty, as my primary development environment. I use it to plan architecture before writing a line of code, debug production incidents, iterate on product, and ship faster than I could alone.

This document is not about AI writing perfect code on the first try. It is about the loop I have developed: describe the symptom, trace the system with the agent, form a hypothesis, apply a targeted fix, test, get feedback, and refine. That loop is what makes AI-assisted development actually work in production.

---

## My general workflow

I treat the agent as a pair programmer with amnesia. It is fast at code generation but weak at remembering constraints unless I restate them clearly. My workflow across every project follows four phases:

**1. Write a plain English spec first.** Before any code, I write out user flows, data model, edge cases, and what I am not building. This stops the agent from hallucinating scope.

**2. Schema and types before UI.** The agent produces far fewer inconsistencies when the database is the source of truth from the start. I always lock the data model before touching any component.

**3. Build vertical slices.** I build one feature end to end, including the API, database, and UI, before moving to the next. This means I always have something running that I can test.

**4. Run the app constantly and paste errors back.** Stack traces and network errors in the prompt get faster, more accurate fixes than vague descriptions. Symptom plus data plus expected behavior is the format that works.

---

## Project 1: Accale.ai — Production debugging and observability

**Stack:** Next.js, Node.js, PostgreSQL, Redis, BullMQ, OpenAI  
**Live:** accale.ai

### Background

Accale's Strive product handles end-to-end email campaign generation. The architecture is intentionally async: the browser creates a campaign row, enqueues a background job, and polls for AI extraction results. This split is easy to get wrong across environments and was at the center of a production incident I debugged with Cursor.

### The incident: works on localhost, broken in production

**My prompt:**

> In localhost, extraction is working fine. Website extraction but in prod it is not. And campaign save is also a problem.

Rather than guessing, the agent traced the full pipeline:

1. CreateCampaignModal sends POST /api/v2/campaign
2. POST /api/v2/data_holder_message with type: campaign_website_intake
3. BullMQ enqueue triggers cron handleCampaignWebsiteIntakeMessage
4. Agent action fetches the URL and runs LLM JSON extraction
5. Result is written to campaign_artifact

It then compared dev versus prod expectations and found three separate root causes:

| Issue | Why local worked | Why prod failed |
|-------|-----------------|-----------------|
| Missing DB migration | Dev DB had dedicated campaign tables | Prod still required data_holder_id NOT NULL, campaign-scoped messages could not insert |
| Cron not running | Local cron and Redis running fine | Prod hang caused the UI to show "Analyzing your website took too long" |
| Screenshot forwarding bug | URL path tested more often | UI-uploaded screenshots were not passed from message handler to agent action |

**Fix 1: Forward screenshots to the intake agent action**

```javascript
// cron/.../message_handlers.js
const screenshots = message.content?.data?.screenshots;
const agentAction = {
  website_url: message.content?.data?.website_url,
  ...(Array.isArray(screenshots) && screenshots.length > 0 ? { screenshots } : {}),
};
```

**Fix 2: Actionable API errors when schema is stale**

Instead of a generic error code, prod now returns an explicit message pointing to the migration script when Postgres errors indicate missing tables or columns. This saved me hours on the next deployment.

### Where the agent got it wrong

The agent attempted to run a live production database query using credentials visible in my local environment files. I caught it before it ran anything and redacted those from this document entirely.

The lesson I took from this: I now ask agents to generate diagnostic scripts, for example check-prod-campaign-schema.mjs, rather than inline connection strings in the chat. The agent is a tool, not a DBA with production access.

### Adding searchable observability

After the incident I sent one follow-up prompt:

> Add errors and give me an event ID so I can search it fast.

The agent extended the existing STRIVE_EVENT registry rather than inventing a parallel logging system, which was exactly the right call.

```typescript
export const STRIVE_EVENT = {
  WEBSITE_INTAKE_STARTED: "strive_website_intake_started",
  WEBSITE_INTAKE_FAILED: "strive_website_intake_failed",
  CAMPAIGN_CREATE_FAILED: "strive_campaign_create_failed",
  CAMPAIGN_SCHEMA_MISSING: "strive_campaign_schema_missing",
} as const;
```

Every error now returns a reference ID in the API response and in the UI toast. I can search logs instantly by event ID and campaign UUID. This is the kind of observability that makes production debugging take minutes instead of hours.

**Where it went wrong again:** After wiring this up, npm run build failed with a TypeScript boundary mismatch:

```
formatStriveApiError expects metadata?: Record<string, unknown>
but API client returns metadata?: APIResponseMetadata
```

The agent had correct runtime behavior but wrong TypeScript types. Running build in the same session caught it. I now run the TypeScript compiler after every multi-file change as a rule.

### Product iteration from screenshots

The most efficient prompts I sent during Accale development were screenshot-driven. Short, visual, and specific.

**Example 1: "Don't save on typing"**

A user reported that auto-save was firing on every keystroke in the messaging pillars section.

The root cause was that hasUnsavedChanges in the campaign client included email_strategy in the debounced auto-save comparison. This was a race-condition-adjacent bug: AI polling was updating form state while the user typed, and auto-save was never the right model for server-generated artifacts. The fix was to exclude AI-generated artifacts from auto-save entirely and only persist on explicit user action.

**Example 2: Duration-scaled email sequences**

A 7-day campaign was generating sequences through Day 21 with 5 emails. My prompt gave the agent exact rules:

| Campaign length | Number of emails |
|----------------|-----------------|
| 4 days or less | 2 |
| Up to 7 days | 3 |
| Up to 14 days | 3 to 4 |
| 21 days or more | 5 or more |

The agent correctly identified that this fix needed to span three layers: the AI prompt, the post-processing normalization, and the UI display. It created a shared utility mirrored in both cron and webapp, updated the AI prompt to inject the exact email count and timing labels, and added a normalization function because models ignore constraints in prompts without it.

This is a pattern I now apply to every AI feature: always add a normalization layer between model output and what you store or display. Models drift. Post-processing keeps production behavior deterministic.

---

## Project 2: The Focus Arena — Full SaaS build with AI agents

**Stack:** Next.js 16, React 19, Supabase, Tailwind CSS 4, Dodo Payments  
**Live:** thefocusarena.com

### What I built

The Focus Arena is a gamified productivity SaaS where users declare one task, pick a timer, and commit to it. The app shows other users currently focusing in a live sidebar, creating silent peer accountability. Features include XP and level progression, GitHub-style 52-week heatmaps, shared focus rooms with synchronized starts, and a full subscription flow with Dodo Payments.

The entire application, roughly 4,600 lines across 42 files, was built in a single intensive AI-assisted session.

### Planning the build

Before writing any code I gave the agent a one-page spec:

> Build a focus app called The Focus Arena. Core loop: user declares ONE task, picks 25/45/90 min, timer counts down, they can give up losing streak and XP or complete earning XP. Show a live sidebar of other people currently focusing. Add levels, streaks, and a GitHub-style heatmap on a dashboard. Stack: Next.js App Router, Supabase, Tailwind. Dark mode default. Do not over-engineer, ship the core loop first.

Then I locked the schema before touching any UI:

> Before coding more, let us lock the data model. Profiles table with username, total_exp, level, streaks, last_session_date. Sessions table with task_declared, duration_minutes, status as active or completed or partial or abandoned, exp_earned. XP formula: base is minutes times 4 plus minutes squared divided by 90. Write the Supabase migration SQL and TypeScript types before any UI.

This stopped the agent from inventing inconsistent field names across pages. When you let the agent build UI and schema simultaneously, you get exp in one place and total_exp in another. Schema first eliminates this class of bug.

### Key debugging moments

**The RLS permission edge case**

The last member to abandon a focus room could not mark it as completed because the row-level security policy only allowed the admin to update the rooms table.

My debug prompt:

> Room stays active forever when the admin abandons early but members are still going. Error in console: new row violates row-level security policy. Fix RLS so any room member can set status to completed when they are the last active member.

The fix was a new Supabase policy:

```sql
CREATE POLICY "Admin or member can update room"
  ON public.rooms FOR UPDATE TO authenticated
  USING (
    auth.uid() = admin_id
    OR EXISTS (
      SELECT 1 FROM public.room_members
      WHERE room_members.room_id = rooms.id
        AND room_members.user_id = auth.uid()
    )
  );
```

The lesson here: AI generates happy-path RLS quickly. Multi-user permission edge cases only appear after manual testing with real accounts. I now list out the multi-user scenarios explicitly in my prompts before building any collaborative feature.

**The schema constraint versus UI divergence**

The original database migration had a CHECK constraint limiting duration_minutes to 25, 45, or 90. Later I asked the agent to build a scroll-wheel time picker with 5-minute increments from 5 to 180. When a user picked 30 minutes, Postgres returned an opaque error.

The agent had built the UI and the schema in separate turns and did not connect the dots. I now include the phrase "update any CHECK constraints" in every UI prompt that changes what values are valid. The agent cannot track dependencies across turns unless you explicitly tell it to.

**The Dodo Payments webhook bug (still partially unfixed)**

The agent generated a webhook handler that called supabaseAdmin.auth.admin.listUsers() twice, once in a broken nested query and once correctly. More importantly, listUsers() paginates, so if the customer was not on page 1 of results, the lookup failed silently.

My debug prompt:

> Webhook receives subscription.active but no row appears in user_subscriptions. Add logging. Also listUsers paginates, if test user is not on page 1 this fails silently.

Logging helped trace the issue. The TODO is still open: remove the dead code, paginate listUsers properly, or store user_id in Dodo checkout metadata to avoid email lookup entirely.

Payment webhooks are where AI-generated code is most dangerous. I now always test with the payment provider's webhook simulator and inspect the database rows directly before calling any payment feature done.

### Debugging table

| Bug | Root cause | How I prompted the fix |
|-----|-----------|----------------------|
| Timer resets to full on refresh | Client stored remaining seconds instead of started_at | Symptom plus session row data plus expected behavior |
| Live sidebar empty | RLS blocked SELECT on other users sessions | Shared the Supabase policy error verbatim |
| Logged-in users redirected from homepage | Middleware redirected all authenticated users to /declare | Described the exact route and expected behavior |
| Room stuck in active state | Admin-only UPDATE policy | Described the multi-user abandon scenario |
| Credentials in migration scripts | Agent hardcoded DB connection string | Manual review before any git push |

---

## What I have learned about working with AI agents

**Context is everything.** Claude performs dramatically better when you give it the full picture upfront: existing architecture, constraints, what you already tried. Vague prompts produce generic code that needs heavy correction.

**Never trust first output on critical logic.** I always verify business logic, edge cases, and security-sensitive code manually. Claude is excellent at structure and boilerplate but I have caught subtle bugs in concurrency handling and error propagation that looked correct but were not.

**Use it for the 80% and own the 20%.** The architectural decisions, the tradeoffs, the production debugging under pressure, those require human judgment. Claude accelerates execution but cannot replace the instinct built from shipping real systems.

**It is a thinking partner as much as a code generator.** Some of my best architectural decisions came from talking through a problem with Claude before writing a single line of code.

**Symptom plus data plus expected behavior is the best debug prompt format.** "Bug: if I refresh /focus at 20:00 remaining, timer jumps back to 45:00. The session row has started_at. Compute elapsed from started_at and pass remaining seconds to useTimer." That kind of prompt consistently gets faster, more accurate fixes than "timer is broken."

**AI-generated code needs normalization layers for AI features.** When the output of one model feeds into your product, always add a post-processing step between model output and what you store or display. Models ignore constraints in prompts. Normalization keeps production behavior deterministic.

---

## What I would do differently

- Run the TypeScript compiler after every multi-file change, not just at the end of a session
- Never pass production credentials in prompts under any circumstances
- Add normalization layers to every AI-generated feature from day one
- Separate draft UI state from persisted AI artifacts, the auto-save bug came from treating polled AI output like user form fields
- List multi-user scenarios explicitly when prompting any collaborative or permission-sensitive feature

---

## Links

- **Live project 1:** accale.ai
- **Live project 2:** thefocusarena.com
- **GitHub:** github.com/Navi12-18
- **LinkedIn:** linkedin.com/in/naivaidya-yadav-0179b7200
- **Contact:** naivaidyayadavin@gmail.com | +91 7999923128

---

*Submitted in response to AI Planet portfolio request. All sensitive values have been redacted. Happy to walk through any section live.*
