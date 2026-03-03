# Frontend — Layout

> **Stack note:** likely TypeScript — unconfirmed. Layout described
> independently of framework or component library.

---

## Unified App Model

TalentGraph is a **single unified application**. A user is not forced to
declare "candidate" or "recruiter" at account creation. Role context is a
soft toggle inferred from action (posting a role activates hiring view;
receiving a match activates talent view). Both views are accessible from the
same session.

---

## Design Aesthetic

Inspired by editorial and paper-first design (generous white space, clean
serif or neutral sans-serif typography, minimal UI chrome). The goal is a
**calm, high-trust surface** that reads as professional and considered, not
SaaS-busy. Think: a well-designed magazine, not a dashboard.

Guiding principles:
- Generous whitespace — content breathes
- Typography-led hierarchy — size and weight over colour and borders
- Minimal button count — actions surface contextually, not as a permanent
  toolbar
- Almost no decorative UI elements — let content carry the visual interest

---

## Navigation Structure

Primary navigation: **4 items maximum** for the talent view.

| Item | View |
|---|---|
| Feed | Professional insight and content feed (daily engagement) |
| Matches | Curated match notifications (premium, high-signal) |
| Profile | Personal profile, completeness ring, privacy controls |
| Hiring | Hiring dashboard (visible when the user has posted a role; hidden otherwise) |

Navigation is persistent but minimal — a slim sidebar on desktop, a bottom tab
bar on mobile. No mega-menus, no flyouts.

"Hiring" tab appears conditionally once a user has posted at least one role.
This keeps the default experience clean for candidates who are not hiring.

---

## Primary Views

### Feed (daily surface)
Full-width content stream. Cards for skill trends, peer moves, industry
insights, and career intelligence. No pagination — infinite scroll with
anchor point on return visit.

### Matches (curated alerts)
A distinct, non-feed surface. Cards presented in a clean list — not swiped,
not scrolled infinitely. Maximum ~7 active match cards at any time. Each card
shows: role, company, match score teaser, "View match" CTA. Score breakdown
revealed on expansion.

### Profile
Left-column layout on desktop: completeness ring + section nav. Main area:
section content (skills, experience, career summary). Progressive disclosure
— sections collapse until filled.

### Hiring
Mirrors the recruiter interface design (see `hiring-experience.md`). Available
on the same account — no separate login.

---

## Mobile-First Principles

- All layouts designed at 375px viewport first; desktop is an enhancement
- Cards are the core layout unit — they stack vertically on mobile, expand
  on desktop
- Touch targets minimum 44px; no hover-only interactions
- Bottom navigation on mobile, left sidebar on desktop (≥768px breakpoint)

---

## Micro-Interaction Guidelines

- **Match reveal:** match quality score animates in (e.g., counter or fade)
  when the user opens a match card — creates a premium "reveal moment"
- **Feed scroll:** smooth, no layout shift; card entrance is a subtle fade-up
- **Profile completeness ring:** animates on update — positive reinforcement
  of profile investment
- **Match notification badge:** appears on the Matches nav item; clears on view

---

## Extension Principle

The 4-item nav accommodates future features within existing sections rather
than by adding top-level items. New features live inside Feed, Matches, Profile,
or Hiring as contextual panels — not new primary destinations.
