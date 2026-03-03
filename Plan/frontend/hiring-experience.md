# Frontend — Hiring Experience

> Describes the hiring-side screens. This view is available to any user —
> not locked to a separate "employer account". A founder, a solo operator, or
> a team hiring manager all use the same interface. See `layout.md` for
> navigation structure.

---

## Access Model

The Hiring tab appears in the navigation once a user has posted at least one
role. It is hidden by default to keep the default candidate experience clean.
Users who want to hire can activate it from their profile settings or from a
"Post a role" CTA surfaced contextually.

There is no separate login, no "recruiter seat", and no hard role separation.
A user can switch between looking for opportunities and hiring within the same
session.

---

## Post a Role

The primary action in the hiring flow.

**Form fields (minimal):**
- Role title (required)
- Job description — free text (required; algorithm infers skills from this)
- Key skills — optional tag input with taxonomy autocomplete; algorithm uses
  this to sharpen match scoring if provided
- Seniority level — optional
- Location / remote policy — optional

"Post Role" submits the JD to the compiler (see `compiler/overview.md`), which
generates a query tensor. The match pipeline runs against the candidate pool
and selects the top N matches. Those candidates are notified.

**Framing shown to the user after posting:**
"Your role has been posted. Matched candidates have been notified and may
apply. You'll see their profiles here as they respond."

This framing is important: the employer does not "search" and does not
"contact" — they post and wait. The algorithm handles selection and outreach.

---

## Applicant View

Shows candidates who responded to a match notification with "I'm interested".

**Applicant card anatomy:**
- Candidate name (or anonymised until consent is granted, if privacy model
  requires it)
- Composite match score + score breakdown (same dimensions as the scoring
  engine: skill coverage, semantic relevance, career trajectory, practical fit)
- Skill highlights: top skills that drove the match
- Experience summary: AI-generated summary relevant to the posted role
- "View full profile" CTA
- Shortlist / Pass actions

**Score breakdown** uses plain-language labels consistent with the candidate-
facing view. Both sides see the same dimensions described the same way.

No "search the whole candidate pool" interface exists. Hiring is entirely
pull-based: post a role → algorithm notifies matched candidates → interested
candidates appear in this view.

---

## Multi-Role View

When a user has posted more than one role, the Hiring tab shows a role list.
Each role card shows:
- Role title
- Posted date
- Applicant count (responded to match notification)
- Status: active / paused / filled

Selecting a role navigates to its Applicant View.

---

## Team-Fill View

For users who have posted multiple roles simultaneously (e.g., filling a team
of 3), an optional "Team view" groups applicants across roles and surfaces
complementarity signals:
- "These two candidates together cover the full skill requirement for your
  engineering team"
- Diversity summary across the team (skill diversity, not demographic — the
  latter is out of scope at this design stage)

This view is powered by the team selection algorithm
(see `features/team-selection.md`).

---

## Design Principles for Hiring

- **Pull, not push:** no interface for cold-contacting candidates who have not
  responded to a match
- **Transparency:** match score breakdown is always shown; hiring is not a
  black box
- **Minimal input required:** the algorithm does the heavy lifting from a
  plain-language JD; structured fields are optional enhancements
