# Product — Onboarding Design

## Principles

- **Value first:** deliver a useful output before asking for full commitment
- **Progressive disclosure:** do not show the full platform on session one
- **Minimal friction:** the shortest possible path to the first "aha moment"
- **Role fork is soft:** users are not forced to declare "I am a candidate" or
  "I am an employer" — both paths are available and can be revisited

---

## Entry Points

Three entry paths into talent onboarding:
1. **CV upload** — primary path, fastest to value
2. **LinkedIn import** — convenience path (likely OAuth — unconfirmed)
3. **Manual entry** — fallback, broken into micro-steps

---

## Talent Onboarding Flow

### Step 1 — Entry and framing
- Landing prompt: "See where you stand in 3 minutes"
- Role fork: soft toggle (talent / hiring / both) — skippable
- Consent note: profile is private by default until explicitly made visible

### Step 2 — Profile seed
- CV upload → AI parse → instant profile draft shown
- Candidate confirms or edits key fields: current role, skills, seniority
- Micro-step framing: "Step 1 of 3 — tell us where you are now"

### Step 3 — First value moment (activation event)
- Run gap analysis against a suggested or selected target role
- Show results immediately: radar chart of current skills vs role benchmark,
  top 3 gaps, top 1 bridge suggestion
- This is the aha moment — the candidate sees something they did not know
  before, sourced from their own profile, in under 3 minutes

### Step 4 — Match preview
- Show 1–2 simulated or real match results (real if employer pool exists;
  simulated demo match if not yet)
- Frame: "This is the kind of role that would match your profile — we'll alert
  you when a real one appears"

### Step 5 — Done
- Profile is live (private); match alerts are enabled
- One optional completion nudge: "Add 2 more skills to improve match quality"
  (shown with impact estimate, not required)
- Land on the feed or profile view — onboarding is complete

**Activation event definition:** candidate completes gap analysis result view
or receives a match preview within the onboarding session.

---

## Employer (Hiring) Onboarding Flow

### Step 1 — Company context
- Company name, sector, size (optional)
- Brief framing: "Post a role and see who the algorithm finds for you"

### Step 2 — First role post
- Minimal JD entry: title, description, key skills (optional — algorithm
  infers skills from description if not provided)
- "Post Role" action triggers algorithm

### Step 3 — First result
- Show top candidate matches (real or illustrative if pool is thin)
- Score breakdown per candidate: why the algorithm selected them
- Frame: "These candidates have been notified and may apply"

### Step 4 — Done
- Employer lands on applicant view for their posted role
- Prompt: invite a colleague to co-review candidates (viral growth vector)

**Activation event definition:** employer sees at least one candidate match
result after posting their first role.

---

## Progressive Disclosure Rules

- On session 1: show profile, gap analysis, one match or preview
- On session 2: show full match inbox, feed personalisation
- On session 3+: surface badges, streak, leaderboard, community features
- Never surface the hiring interface during talent onboarding (and vice versa)
  unless the user explicitly toggles roles
