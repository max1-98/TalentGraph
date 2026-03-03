# Product — Engagement and Retention Design

## Core Insight: Matches Are Too Sparse for a Daily Hook

A candidate realistically receives ~7 eligible match notifications across their
active search window. That is not enough to build a daily habit around.

**Match notifications are a premium, high-signal event — not the daily hook.**

The daily hook is a **content and insight feed**. The weekly hook is match and
opportunity alerts. Engagement design must respect this distinction.

---

## The Habit Loop (Hooked Model, Nir Eyal)

### Daily Loop — Feed Layer
- **Trigger (external):** push or email — "3 new skill trends in your sector"
- **Trigger (internal):** professional curiosity, career anxiety, peer comparison
- **Action:** open app, scroll feed
- **Reward:** variable — a surprising insight, a peer move, a skill demand spike
- **Investment:** like, save, or share a post → feed personalises further

### Weekly Loop — Match Layer
- **Trigger:** "2 new roles match your profile this week" digest (email/push)
- **Action:** open match inbox, review match cards
- **Reward:** variable — one match is exceptional, score revealed with animation
- **Investment:** profile update nudge ("add X to unlock better matches")

The variable reward principle (unpredictable payoff) applies to both loops.
Feed content is not fully predictable. Match quality varies. The "exceptional
match" is rare enough to feel like a win.

---

## Feed Content Types

The candidate feed is personalised and includes:
- **Skill trend signals:** "Python demand up 18% in your sector this month"
- **Peer moves:** "5 professionals with your profile transitioned to [Role X]
  this quarter" (anonymised by default)
- **Career insight cards:** algorithmic observations based on profile and market
- **Industry spotlights:** hiring activity, salary movement, in-demand roles
- **Company spotlights:** culture, growth trajectory, open roles

The feed does not include generic news or content unrelated to career.
It is a **professional intelligence layer**, not a social timeline.

---

## Streak Mechanics

**Profile Freshness Indicator**
- A visual "freshness score" (e.g., a gauge or timestamp label) signals when
  a profile is stale
- Nudge: "Update your profile to stay visible to match searches"
- Streak: number of consecutive weeks the profile has been updated
- Reward: "Fresh profile" badge; match algorithm weights recency of updates

**Match Engagement Streak**
- Streak for weekly match digest open/review (not just open — interaction)
- Shown as a subtle count ("3-week streak"), not a dominant UI element

---

## Badge System

Badges mark meaningful milestones, not arbitrary activity:
- **First Match** — first algorithmic match notification received
- **First Apply** — first application submitted via match
- **Skill Pioneer** — added a skill before it appeared in the main taxonomy
- **Gap Closer** — completed a suggested skill bridge
- **10-Week Active** — maintained profile freshness for 10 consecutive weeks
- **Interview Scheduled** — (employer-confirmed or self-reported)

Badges are visible on the public profile (opt-in). They signal platform
credibility and professional investment.

---

## Notification Philosophy

Notifications are **milestone-triggered only**, not periodic noise:
- New match: send immediately, with score teaser
- Weekly digest: send if there is at least one new signal (match, skill trend,
  peer move) — suppress if nothing new
- Freshness nudge: send once per 3-week lapse, not every week
- Badge earned: send once

No "we miss you" emails. No engagement bait. Every notification should
deliver information the user would want even if they weren't trying to
re-engage with the platform.

---

## Social Proof Signals

Surface aggregate data to signal platform vitality:
- "47 employers searched this skill in your sector this week"
- "Your profile appeared in 12 match evaluations this month"
- "Candidates with your profile received 3.2 match notifications on average
  this quarter"

These signals are shown contextually (profile page, match inbox) not as a
permanent counter. They create ambient awareness of market activity.
