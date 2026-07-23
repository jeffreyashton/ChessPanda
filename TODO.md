# 🐼 ChessPanda — TODO

<!--
  ┌─────────────────────────────────────────────────────────────────┐
  │  THE PIPELINE — a task is a value moving through states.        │
  └─────────────────────────────────────────────────────────────────┘

       capture        triage         start         finish
          │             │              │              │
          ▼             ▼              ▼              ▼
      [Inbox] ────▶ [Next] ─────▶  [Now] ─────▶  [Done]
          │             ▲             │
          │             └── unblock ──┤
          ▼                           ▼
     [Someday]                    [Blocked]
          │                           │
          └────────▶ [Dropped] ◀──────┘

  Inbox     stdin. Unparsed input. Never work straight from here.
  Next      the ready queue. Unblocked, and you would start it today.
  Now       the call stack. What is actually executing. Three frames, max.
  Blocked   awaiting I/O. Name the await, or it is a leak.
  Someday   cold storage, lazily evaluated. Maybe never. That is allowed.
  Dropped   garbage collected. Killing a task on purpose is a feature.
  Done      the commit log. Append-only, newest on top, dated.

  ┌─────────────────────────────────────────────────────────────────┐
  │  INVARIANTS — if one breaks, the system is lying to you.        │
  └─────────────────────────────────────────────────────────────────┘

    1. Now holds ≤ 3 items.
       Context switching is a cache miss. You have three cores, not thirty.

    2. Every project has exactly one obvious Next.
       Zero Next actions means the project is stalled, not "in progress."

    3. Every Blocked item names what it awaits AND who owns the await.
       "Blocked on Supabase" is a leak.
       "awaits: .env  ·  owner: me" is a promise someone can resolve.

    4. The Inbox is empty after the weekly review.
       An unbounded queue is a memory leak with extra steps.

    5. Done is append-only.
       You never delete history. You revert — and the revert is also history.

  ┌─────────────────────────────────────────────────────────────────┐
  │  ROW FORMAT                                                     │
  └─────────────────────────────────────────────────────────────────┘

    - [ ] verb the object  ·  note
    - [x] verb the object
    - YYYY-MM-DD  what shipped

  Lead with a verb. A task with no verb is a topic, and topics cannot be
  finished — only abandoned.
-->

## Now
<!-- ≤ 3. Want a fourth? Something here moves out first. -->

## Next
<!-- Ready to run. No unmet dependencies. -->

- [ ] Point chesspanda.com (GoDaddy) at a Cloudflare Pages app  ·  keep the Square store live on the apex or a subdomain; lab.chesspanda.com already exists

## Blocked
<!-- - [ ] thing  ·  awaits: X  ·  owner: Y -->

## Inbox
<!-- Raw capture. Triage at the weekly review, then empty this. -->

## Someday
<!-- Not scheduled. Not guilt. -->

## Dropped
<!-- Killed on purpose, with a one-line reason. -->

## Done
<!-- Newest on top. Append-only. -->
