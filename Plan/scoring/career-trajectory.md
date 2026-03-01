# f4 — Career Trajectory Fit

Evaluates whether the candidate's career arc aligns with what the role requires
— not just current level, but progression pattern, industry background, and
stability signals.

---

## Formula

f4(c, q) = w_sen * g(c_seniority, q_seniority)
          + w_ind * cos(c_industry, q_industry)
          + w_stab * c_stability
          + w_lead * c_leadership

## Components

### Seniority Gaussian

g(c, q) = exp( -(c_seniority - q_seniority)^2 / (2 * sigma^2) )

A Gaussian penalty centred on the target seniority level. Being too junior is
penalised. Being too senior is also penalised (overqualified, flight risk).

- sigma controls tolerance (default 0.15)
- Tunable per search: a startup searching for "senior" may set sigma wide
  (they accept a range), while a bank searching for "VP" sets sigma narrow
  (exact level matters)
- Seniority is a continuous 0–1 scale from intern to C-suite

### Industry Alignment

Cosine similarity between the candidate's industry vector and the query's
target industries. The industry vector is sparse, weighted by recency and
duration of experience in each industry.

### Stability Score

Pre-computed scalar capturing average tenure and tenure consistency. A candidate
with steady three-year stints scores higher than one with frequent six-month
jumps. Not inherently good or bad — but correlated with what many employers seek.

### Leadership Score

Pre-computed scalar reflecting team size managed, scope of responsibility, and
duration in leadership roles. Used when the query specifies a leadership
requirement.

## Career Trajectory Vector (16 dimensions)

The career_trajectory field in the tensor encodes progression signals:

- Seniority slope — rate of title progression over time
- Scope slope — rate of scope expansion (team size, budget, impact)
- Tech diversity rate — how quickly new technologies are adopted
- Industry transitions — number of industry changes (normalised)
- Average tenure and tenure variance
- Recent momentum — is progression accelerating or slowing?
- Gap months — total career gaps (normalised)
- Specialisation index — Herfindahl index of skill concentration (high =
  deep specialist, low = generalist; neither is inherently better)
- Leadership onset — years into career before first leadership role
- Current velocity — PCA of skill vector delta over last two years
- Company tier trend — moving up or down in company prestige
- Education recency — continuous learning signal

## Edge Cases

- **Career changer:** industry cosine may be low, but recent momentum and
  tech diversity rate will be high — the trajectory vector captures nuance
  that a simple "years in industry" metric would miss
- **Gap in career:** the gap_months field is penalised smoothly, not as a
  hard filter
- **Overqualified candidate:** the Gaussian naturally penalises seniority
  mismatch in both directions
