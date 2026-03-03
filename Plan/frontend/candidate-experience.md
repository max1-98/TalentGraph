# Frontend — Candidate Experience

> Describes the candidate-facing screens and their content hierarchy.
> See `layout.md` for navigation structure and design principles.

---

## Home Feed (daily engagement surface)

The primary entry point for returning candidates. Content is personalised
to the candidate's skill profile, career stage, and sector.

**Card types surfaced in the feed:**
- **Skill trend cards:** "Demand for [Skill X] is up 18% in your sector
  this month" — with a chart thumbnail and a link to the skill detail
- **Peer move cards:** "5 professionals with a profile similar to yours
  moved into [Role X] this quarter" — anonymised, aggregate
- **Career insight cards:** algorithmic observations drawn from the candidate's
  profile and market context — e.g., "Your skill profile is best aligned with
  [Role Type] based on recent hiring patterns"
- **Industry spotlight cards:** hiring activity, salary movement, notable
  employers active in the candidate's sector
- **Company spotlight cards:** culture signal, growth trajectory, open roles
  relevant to the candidate

Feed is **not** a social timeline. There are no status updates, no likes-as-
primary-metric, no user-generated noise. Every card carries professional signal.

---

## Match Notifications (premium, curated section)

Accessed via the Matches nav item. Distinct from the feed — this is the
high-signal, low-volume surface.

**Match card anatomy:**
- Role title + company name
- Match score teaser (e.g., "Strong match — 87%") visible before expansion
- Score breakdown on expand: skill coverage, semantic relevance, career
  trajectory (plain-language labels; values from the scoring engine)
- Apply CTA: "I'm interested" — submits the candidate's profile to the employer
- Dismiss: removes the card without applying; no penalty
- ~7 active match cards at any time; older cards age out as new ones arrive

Score is revealed with a brief animation on first open — the "reveal moment"
reinforces the premium feel of each match.

---

## Gap Finder View

Accessible from Profile or as a contextual CTA after a match card.

**Layout:**
- Target role selector (search with taxonomy autocomplete)
- Radar chart: current skill profile vs role benchmark (normalised dimensions)
- Gap list below chart: ordered by priority (see `algorithm/gap-finder/`)
  - Each gap shows: skill name, current level vs required, priority label
  - Inline bridge suggestion for top 1–3 gaps ("To close this gap: ...")
- "Save target role" option — persists the comparison and enables change
  tracking over time

The view is actionable by design: every gap has a next step.

---

## Career Path View

Shows data-driven trajectory projections based on the candidate's profile
and aggregate peer data.

**Layout:**
- Current role position on a career graph (node)
- Outgoing edges: most common next-role transitions from this position
  (edge weight = transition frequency among similar profiles)
- Hover/expand on a destination node: required skills delta, median time
  to transition, salary range movement
- "Set as target" CTA links to the Gap Finder for the selected destination

Data is anonymised aggregate — no individual peer profiles are exposed.

---

## Profile View

**Layout:**
- Top: profile completeness ring + current match-readiness label
- Skills section: skill cloud or list, with confidence/seniority per skill;
  taxonomy-linked; editable inline
- Experience section: timeline of roles, with AI-extracted highlights
- Career summary: auto-generated from structured data; editable
- Privacy controls: per-section visibility toggle (public / employers only /
  private)

Completeness ring animates on update. Each section shows its contribution
to overall completeness and to estimated match quality.

---

## Insights Dashboard

An optional deeper analytics view, accessible from Profile.

- Match activity over time: how many roles the algorithm evaluated the
  candidate against, trend line
- Score distribution: where the candidate scores well vs. which dimensions
  drag the composite score down
- Market demand pulse: top skills being searched in the candidate's sector
- Skill trajectory: how the candidate's skill profile has changed over time
  (based on profile update history)
