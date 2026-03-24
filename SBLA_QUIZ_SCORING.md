# SBLA Beauty Quiz — Scoring & Logic Reference

> **File:** `quiz.html` (single-file app, all logic inline)
> **Live URL:** https://sbla-beauty-quiz.vercel.app
> **Repo:** https://github.com/robo-gs-team/sbla-beauty-quiz-v1
> **Last updated:** March 2026

---

## ★ OVERVIEW (Simplified Summary)

> _Read this section first. Everything below is the full technical detail._

The SBLA Beauty Quiz is a 6-question skin diagnostic that recommends the right SBLA product for each shopper based on where they're aging, how severe their concern is, and how invested they are in their skincare routine.

**How it works in plain English:**

1. **The shopper picks their primary concern** — eyes, face, neck/jaw, lips, or "honestly everything." This is the biggest signal.
2. **They pick a secondary concern** (if applicable) — this can push a bundle recommendation.
3. **They answer a diagnostic** — for neck shoppers this determines whether they get the Standard Wand or the XL. For everyone else it confirms their primary zone.
4. **They describe their skincare commitment** — "just starting out" vs. "skincare is a serious investment." High-investment shoppers are flagged for premium bundle escalation.
5. **They describe their skincare experience** — this only affects the copy shown on their result page, not the product recommendation.
6. **They describe their ideal outcome** — zones they pick here add small confirmation points. Picking "transformation everywhere" routes them to a bundle regardless.

**What they get at the end:**

| Scenario | What they see |
|---|---|
| One clear zone concern | Single hero product matched to that zone |
| Two competing zone concerns (close scores) | Hero product + bundle upsell below the CTA |
| High-investment shopper with multiple zones | Bundle as the primary hero (no single SKU) |
| "Honestly everything" in Q1 OR "transformation" in Q6 | Immediately routed to full-routine bundle |

**The 7 possible results:**

| # | Result | Hero Product |
|---|---|---|
| 1 | Eye Zone | Eye Lift Wand |
| 2 | Face Zone | Liquid Facelift Wand |
| 3 | Neck Zone — XL | Neck, Chin & Jawline Sculpting Wand XL |
| 4 | Neck Zone — Standard | Neck, Chin & Jawline Sculpting Wand |
| 5 | Lip Zone | Double The Plump Lip Plump & Sculpt |
| 6 | Multi-Zone Bundle (Primary) | Power Trio Set or Essential Collection |
| 7 | Hard Bundle — Full Routine | Essential Collection or Ultimate Sculpting Set |

---

---

## PART 1 — QUIZ STRUCTURE

### Overview

| Question | Screen | Type | Scoring Role |
|---|---|---|---|
| Q1 | `q1` | Zone cards + full-width option | Primary zone signal (+50 pts) |
| Q2 | `q2` | Dynamic radio list | Secondary zone signal (+25 pts) |
| Q3 | `q3` | Dynamic radio list (conditional) | Diagnostic / zone confirmation (+10 or +5 pts) |
| Q4 | `q4` | Radio list | Skincare commitment → escalation flag or +5 pts |
| Q5 | `q5` | Radio list | Copy flag only — no zone scoring |
| Q6 | `q6` | Dynamic radio list (filtered) | Outcome aspiration → zone confirmation (+10 pts) or hard trigger |

**Total maximum zone points possible (single zone):** 50 + 25 + 10 + 5 + 10 = **100 pts**

---

### Q1 — Primary Concern

**Question text:** "What area of your face are you most focused on right now?"

**Options:**

| Option Label | Internal Value | Score Applied |
|---|---|---|
| Eyes (eye area icon) | `eye` | `scores.eye += 50` |
| Face (firmness, overall) | `face` | `scores.face += 50` |
| Neck & jaw | `neck` | `scores.neck += 50` |
| Lips | `lip` | `scores.lip += 50` |
| **Honestly? Everything.** (full-width) | `all` | ALL zones `+= 20` each; `hardBundleTrigger = true` |

**Flow after Q1:**
- If `val === 'all'` → skip Q2 entirely → go directly to Q3 (progress bar updated to 2/5)
- All other answers → go to Q2

---

### Q2 — Secondary Concern

**Question text:** "Is there another area you've been thinking about too?"

**Options:** Dynamically built — shows all zones EXCEPT the one chosen in Q1. Always includes "No, I'm focused on [Q1 zone]" as a last option.

**Scoring:**

| Answer | Score Applied |
|---|---|
| Any zone (eye / face / neck / lip) | `scores[zone] += 25` |
| "No, focused on one" (`none`) | No score change |

**Flow:** Always proceeds to Q3.

---

### Q3 — Diagnostic (Conditional)

Q3 has **two completely different versions** depending on whether the shopper is neck-routed.

**Neck-routed condition:** `state.q1 === 'neck' OR state.q2 === 'neck'`

---

#### Q3A — Neck Version (neck-routed shoppers)

**Question text:** "When you focus on your neck and jaw area, what stands out most?"

| Option | Value | Score Applied | State Effect |
|---|---|---|---|
| Loose, crepey, or sagging skin | `standard` | `scores.neck += 5` | `neckResolution = 'standard'` |
| Under-chin fullness or loss of jawline | `xl` | `scores.neck += 5` | `neckResolution = 'xl'` |
| Both — the full area has changed | `both` | `scores.neck += 5` | `neckResolution = 'xl'`; `bundleConsiderationFlag = true` |

> **Important:** "Both" defaults `neckResolution` to `xl` (XL is always the stronger recommendation when both concerns are present) AND sets `bundleConsiderationFlag = true`, which contributes to bundle escalation later.

---

#### Q3B — Non-Neck Version (all other shoppers)

**Question text:** "What does your skin tell you most?"

| Option | Value | Score Applied |
|---|---|---|
| Dry or tight, especially after cleansing | `face` | `scores.face += 10` |
| Puffiness in the morning around eyes | `eye` | `scores.eye += 10` |
| Lost bounce and elasticity | `neck` | `scores.neck += 10` |
| Lines around lips getting more noticeable | `lip` | `scores.lip += 10` |

> Note: In Q3B, the answer maps directly to a zone and adds +10 to that zone, even if it differs from the Q1 / Q2 primary.

---

### Q4 — Skincare Commitment

**Question text:** "How would you describe your relationship with skincare right now?"

| Option | Value | Score Applied | Flag |
|---|---|---|---|
| Just starting to build a routine | `entry` | `+5` to whichever of eye/face has more points | — |
| Consistent morning and night routine | `engaged` | `+5` to current winning zone | — |
| Tried a lot, struggle with consistency | `results_chaser` | `+5` to current winning zone | — |
| Skincare is a serious priority — I invest | `high_intent` | `+5` to ALL zones | `escalationFlag = true` |

**`escalationFlag` mechanics:**
- Set when `q4 === 'high_intent'`
- When combined with any bundle signal later, escalates the bundle tier
- Triggers premium bundle as hero (State 6 override) if multiple zones are competitive

**"Entry" tie-break logic:** If `scores.eye >= scores.face`, give +5 to eye; otherwise give +5 to face. This prevents the generic "entry" answer from accidentally boosting a zone that was never selected.

---

### Q5 — Skincare Experience (Copy Flag)

**Question text:** "What's your experience with targeted skincare?"

> **This question does NOT affect zone scores.** It sets a `q5_copyFlag` only, which personalizes the sub-headline shown on the result page.

| Option | Value | Copy Shown on Result Page |
|---|---|---|
| Yes — creams and serums, didn't see results | `mechanism_differentiator` | "You've tried the creams. Here's why the wand delivery system is clinically different." |
| Yes — I've used tools or devices | `upgrade_positioning` | "You've used tools before. SBLA's wand technology is the targeted evolution." |
| I'm fairly new to targeted treatments | `education_first` | "New to targeted treatments? Here's exactly what this formula addresses — and why it works." |
| I've used SBLA before | `loyalty` | "You know SBLA. Here's your next step in the routine." |

---

### Q6 — Outcome Aspiration

**Question text:** "What would make you feel like it actually worked?"

**Options are filtered** based on zones the shopper expressed interest in (Q1 + Q2 combined). Only zones in `{ q1, q2 }` appear, plus the "transformation" option which always shows.

- Exception: If `q1 === 'all'`, all 5 options are shown.

| Option | Value | Score Applied | Trigger |
|---|---|---|---|
| Rested, refreshed — eyes open and awake | `eye` | `scores.eye += 10` | — |
| Firmer skin, smoother — like it used to | `face` | `scores.face += 10` | — |
| Lifted, sculpted neck and jaw | `neck` | `scores.neck += 10` | — |
| Lips naturally fuller and defined | `lip` | `scores.lip += 10` | — |
| **Overall transformation — feel like myself** | `all` | No score change | `hardBundleTrigger = true` → immediately routes State 7 |

**After Q6:** → Analyzing screen (1.8s delay) → `calculateResult()`

---

---

## PART 2 — SCORING ENGINE

### Zone Score Accumulation Table

Maximum possible score per zone (single zone focus, best-case path):

| Question | Points | Condition |
|---|---|---|
| Q1 primary | +50 | Zone selected as primary |
| Q2 secondary | +25 | Zone selected as secondary |
| Q3 diagnostic | +10 | Zone confirmed by Q3B answer |
| Q4 commitment | +5 | Any answer adds to winning zone |
| Q6 outcome | +10 | Zone confirmed in Q6 |
| **Total** | **100** | — |

For neck (Q3A path): Q3 adds only +5 (not +10), so max neck score = 95.

---

### Hard Bundle Triggers

Two conditions immediately route to **State 7** and bypass all scoring logic:

| Trigger | When It Fires |
|---|---|
| `hardBundleTrigger` via Q1 | User selects "Honestly? Everything" in Q1 |
| `hardBundleTrigger` via Q6 | User selects "An overall transformation" in Q6 |

When either fires: `calculateResult()` calls `renderResult(7)` immediately and returns — no further logic runs.

---

### Lip Eligibility Gate

The Lip SKU is restricted. It can only be recommended if:

```
scores.lip > 30
```

**Rationale:** The lip product is a lower-tier, lower-intent SKU. A shopper must explicitly signal lip concern across multiple questions to unlock it. A single Q1 lip selection (50 pts) passes this gate. But lip concern mentioned only via Q3B (10 pts) or Q6 (10 pts) alone does not.

**If lip wins but fails eligibility:** The engine temporarily sets `scores.lip = 0`, re-runs `getWinningZone()` to find the next best zone, then restores the lip score. The shopper gets their second-best result with no visible disruption.

---

### Competing Zones — Bundle Upsell Flag

After the winning zone is determined, the engine checks whether a second zone is "close enough" to warrant showing a bundle upsell:

```
if (scores[winZone] - secondZone.score) <= 10 AND secondZone.score > 0
  → bundleUpsellFlag = true
```

**Additional bundle signal for neck primary:**
```
if winZone === 'neck' AND secondZone.score > 20
  → bundleUpsellFlag = true
```

**`bundleUpsellFlag` effect:**
- Shows a secondary bundle product below the main CTA on the result page
- Does NOT change the hero product (unless combined with `escalationFlag`)

---

### Multi-Zone Primary Escalation (State 6)

If ALL three conditions are true, the shopper is moved from a single-SKU result to a bundle-primary result:

```
bundleUpsellFlag === true
AND escalationFlag === true (Q4 = high_intent)
AND (scores[winZone] - secondZone.score) <= 10
  → renderResult(6)
```

**What this means in practice:** A high-investment shopper who is genuinely torn between two zones gets a bundle as their primary recommendation rather than a single product with an upsell nudge.

---

### Result State Determination (Full Logic)

```
1. If hardBundleTrigger → State 7, exit

2. Calculate lipEligible = (scores.lip > 30)

3. Find winZone = highest scoring zone
   If winZone === 'lip' AND !lipEligible:
     → temporarily zero lip, recalculate winZone, restore lip

4. Calculate bundleUpsellFlag:
   → if gap between 1st and 2nd zone ≤ 10 pts
   → or if neck wins AND 2nd zone score > 20

5. Apply escalation:
   → if escalationFlag AND (bundleUpsellFlag OR bundleConsiderationFlag)
   → ensure bundleUpsellFlag = true

6. Check State 6 override:
   → if bundleUpsellFlag AND escalationFlag AND gap ≤ 10 → State 6, exit

7. Map winZone to result state:
   eye   → State 1
   face  → State 2
   neck  → State 3 (XL) or State 4 (Standard) based on neckResolution
   lip   → State 5 (if lipEligible, else fallback to face → State 2)
   else  → State 2 (face fallback)

8. renderResult(resultState)
```

---

---

## PART 3 — RESULT STATES

### State 1 — Eye Zone

**Hero:** Eye Lift Wand — $89.40 (was $149, Save $60)
**Mechanism:** SBLA66Peptide™ — clinically proven to lift, tighten, and open the eye area.
**B&A:** Combined image (single photo, before/after side by side) — "Results over time"
**Zone label:** YOUR EYE AREA
**Bundle upsell (if `bundleUpsellFlag`):** Your Best Face Set — $171 (was $380)

---

### State 2 — Face Zone

**Hero:** Liquid Facelift Wand — $107.40 (was $179, Save $72)
**Mechanism:** Line Relaxing Complex — peptides and amino acids that eliminate fine lines within hours.
**B&A:** Combined image — "After 21 days"
**Zone label:** YOUR FACE
**Bundle upsell (if `bundleUpsellFlag`):** Your Best Face Set — $171 (was $380)

---

### State 3 — Neck Zone (XL)

**Triggered by:** `neckResolution === 'xl'` (under-chin fullness selected, or "both", or no Q3 neck answer)
**Hero:** Neck, Chin & Jawline Sculpting Wand XL — $77.40 (was $129, Save $52)
**Mechanism:** Fat Melting Complex — clinically proven to melt fat under the chin and define the jawline.
**B&A:** Combined image — "In as little as 14 days"
**Zone label:** YOUR NECK & JAWLINE
**Bundle upsell logic:**
- If `escalationFlag`: → Power Trio Set ($204, was $457)
- If secondary zone is eye or face: → Power Trio Set
- Otherwise: → Neck + Neck XL Duo Set ($117, was $218)

---

### State 4 — Neck Zone (Standard)

**Triggered by:** `neckResolution === 'standard'` (crepey/sagging skin explicitly selected in Q3A)
**Hero:** Neck, Chin & Jawline Sculpting Wand — $53.40 (was $89, Save $36)
**Mechanism:** Macro-Sphere technology — firms, lifts, and sculpts the neck and jawline progressively.
**B&A:** Split image (before/after separate photos, Avis W. Age 50) — "After 6 weeks"
**Zone label:** YOUR NECK & JAWLINE
**Bundle upsell (if `bundleUpsellFlag`):** Neck + Neck XL Duo Set — $117 (was $218)

---

### State 5 — Lip Zone

**Triggered by:** `winZone === 'lip' AND lipEligible` (scores.lip > 30)
**Hero:** Double The Plump Lip Plump & Sculpt — $31.20 (was $52, Save $21)
**Mechanism:** LIPerfection™ — immediate plumping and definition. No injections.
**B&A:** Combined image (face B&A, perioral region) — "After 4 weeks"
**Zone label:** YOUR LIPS
**Bundle upsell (if `bundleUpsellFlag`):** The Ultimate Sculpting Set — $268.80 (was $598)

> **Note:** Lip is a Tier P4 SKU (lowest priority). It can only be recommended when the shopper's lip score exceeds 30, meaning they actively selected lip in Q1 (50 pts) and/or mentioned it in other questions. A lip mention in only Q3 or Q6 (≤20 pts total) does NOT qualify.

---

### State 6 — Multi-Zone Bundle (Primary)

**Triggered by:** `bundleUpsellFlag AND escalationFlag AND gap ≤ 10 pts`
**`isBundlePrimary = true`** — the result page shows "YOUR COMPLETE ROUTINE" and the headline changes to "You're ready for the full routine. Here's where to start."

**Hero selection:**
- If `escalationFlag`: → The Essential Collection ($246, was $546)
- Otherwise: → The Power Trio Set ($204, was $457)

**Secondary upsell (if `escalationFlag`):** The Ultimate Sculpting Set — $268.80 (was $598), labelled "Add lips to your routine"

**B&A:** Split image (Gretchen, 45) — "In as little as 14 days"

---

### State 7 — Hard Bundle (Full Routine)

**Triggered by:** `hardBundleTrigger === true` (Q1 = "everything" OR Q6 = "transformation")
**`isBundlePrimary = true`** — no secondary upsell shown. They are already at the top recommendation.

**Hero selection:**
- If `escalationFlag`: → The Ultimate Sculpting Set ($268.80, was $598) — every wand
- Otherwise: → The Essential Collection ($246, was $546) — all four wands

**B&A:** Split image (Gretchen, 45) — "In as little as 14 days"

> **Note:** State 7 never shows a secondary bundle upsell block. The shopper is already at the ceiling of the product catalogue.

---

---

## PART 4 — PRODUCT CATALOGUE

### Single-Zone SKUs

| Product | Zone | Tier | Price | Original | Savings |
|---|---|---|---|---|---|
| Eye Lift Wand | eye | P1 | $89.40 | $149 | Save $60 |
| Liquid Facelift Wand | face | P1 | $107.40 | $179 | Save $72 |
| Neck, Chin & Jawline Sculpting Wand | neck | P2 | $53.40 | $89 | Save $36 |
| Neck, Chin & Jawline Sculpting Wand XL | neck | P2 | $77.40 | $129 | Save $52 |
| Double The Plump Lip Plump & Sculpt | lip | P4 | $31.20 | $52 | Save $21 |

### Bundle SKUs

| Product | Tier | Price | Original | Savings | Contents |
|---|---|---|---|---|---|
| Your Best Face Set | P4 | $171 | $380 | Save $209 | Eye Lift + Liquid Facelift |
| Neck + Neck XL Duo Set | P3 | $117 | $218 | Save $101 | Standard + XL neck wands |
| The Power Trio Set | P3 | $204 | $457 | Save $253 | Liquid Facelift + Eye Lift + Neck XL |
| The Essential Collection | P3 | $246 | $546 | Save $300 | All four wands |
| The Ultimate Sculpting Set | P4 | $268.80 | $598 | Save $329 | Every SBLA wand |

**Tier hierarchy (for upsell escalation):**
- P1 = Single-zone hero SKUs (highest margin focus)
- P2 = Neck single SKUs
- P3 = Mid-tier bundles (Power Trio, Essential, Duo)
- P4 = Entry lip / aspirational full sets

---

---

## PART 5 — COPY FLAG SYSTEM

The copy flag is set by Q5 and shown as a sub-headline on the result page. It does not affect product routing.

| Flag Value | Q5 Answer | Result Sub-Headline Copy |
|---|---|---|
| `mechanism_differentiator` | Creams/serums, no results | "You've tried the creams. Here's why the wand delivery system is clinically different." |
| `upgrade_positioning` | Used tools/devices before | "You've used tools before. SBLA's wand technology is the targeted evolution." |
| `education_first` | New to targeted treatments | "New to targeted treatments? Here's exactly what this formula addresses — and why it works." |
| `loyalty` | Used SBLA before | "You know SBLA. Here's your next step in the routine." |

If no copy flag is set (edge case), the sub-headline is hidden entirely.

---

---

## PART 6 — EXAMPLE SCORING PATHS

### Path A — Clean Eye Focus, No Bundle

| Step | Answer | Score Delta | Running Scores |
|---|---|---|---|
| Q1 | Eye | `eye +50` | eye=50, face=0, neck=0, lip=0 |
| Q2 | No, focused on eyes | No change | eye=50 |
| Q3B | Puffiness in the morning | `eye +10` | eye=60 |
| Q4 | Consistent routine | `eye +5` | eye=65 |
| Q5 | Used creams, no results | Copy flag only | — |
| Q6 | Rested, refreshed eyes | `eye +10` | eye=75 |

Gap between eye (75) and face (0) = 75 pts → no bundle upsell. No escalation flag.
→ **Result: State 1 — Eye Lift Wand**

---

### Path B — Eye + Face Competing, High-Intent Shopper

| Step | Answer | Score Delta | Running Scores |
|---|---|---|---|
| Q1 | Eye | `eye +50` | eye=50 |
| Q2 | Face | `face +25` | eye=50, face=25 |
| Q3B | Dry/tight skin | `face +10` | eye=50, face=35 |
| Q4 | Serious investment | ALL `+5` | eye=55, face=40, neck=5, lip=5 |
| Q5 | Used tools before | Copy flag | — |
| Q6 | Firmer skin | `face +10` | eye=55, face=50 |

Gap: eye=55, face=50 → gap=5 ≤ 10 → `bundleUpsellFlag = true`
`escalationFlag = true` (Q4 = high_intent)
Multi-zone check: bundleUpsell + escalation + gap ≤ 10 → **State 6**
→ **Result: State 6 — The Essential Collection (hero), Ultimate Sculpting Set (upsell)**

---

### Path C — Neck XL, Single Zone

| Step | Answer | Score Delta | Running Scores |
|---|---|---|---|
| Q1 | Neck | `neck +50` | neck=50 |
| Q2 | No, focused on neck | No change | neck=50 |
| Q3A | Under-chin fullness (xl) | `neck +5`, `neckResolution='xl'` | neck=55 |
| Q4 | Consistent routine | `neck +5` (winning zone) | neck=60 |
| Q5 | New to targeted treatments | Copy flag | — |
| Q6 | Lifted neck and jaw | `neck +10` | neck=70 |

Gap: neck=70, all others=0 → no bundle upsell. neckResolution='xl'.
→ **Result: State 3 — Neck XL Wand**

---

### Path D — Hard Trigger via Q6

| Step | Answer | Notes |
|---|---|---|
| Q1 | Face | `face +50` |
| Q2 | Eyes | `eye +25` |
| Q3B | Tight skin | `face +10` |
| Q4 | Just starting | `face +5` |
| Q5 | New to treatments | Copy flag |
| Q6 | **Overall transformation** | `hardBundleTrigger = true` |

Regardless of face leading, the Q6 "transformation" answer fires the hard trigger.
No escalation flag → hero = Essential Collection.
→ **Result: State 7 — The Essential Collection**

---

---

## PART 7 — STATE & FLAG REFERENCE

### State Object (Full)

```javascript
state = {
  currentScreen: 'entry',       // active screen ID
  history: [],                   // navigation history (for goBack())
  scores: {                      // zone accumulation
    eye: 0,
    face: 0,
    neck: 0,
    lip: 0
  },
  q1: null,                      // 'eye' | 'face' | 'neck' | 'lip' | 'all'
  q2: null,                      // zone key | 'none'
  q3: null,                      // zone key (Q3B) | 'standard' | 'xl' | 'both' (Q3A)
  q3_neck_variant: null,         // 'standard' | 'xl' | 'both' — neck only
  q4: null,                      // 'entry' | 'engaged' | 'results_chaser' | 'high_intent'
  q5_copyFlag: null,             // copy flag key
  q6: null,                      // zone key | 'all'
  hardBundleTrigger: false,      // true → always State 7
  bundleUpsellFlag: false,       // true → show secondary bundle on result
  escalationFlag: false,         // true → premium tier preference
  neckResolution: 'xl',          // default XL bias; set to 'standard' by Q3A
  bundleConsiderationFlag: false // true when Q3A = 'both' (both neck concerns)
}
```

### Flag Summary

| Flag | Set By | Effect |
|---|---|---|
| `hardBundleTrigger` | Q1='all' OR Q6='all' | Bypasses all scoring → State 7 |
| `escalationFlag` | Q4='high_intent' | Escalates bundle tier; contributes to State 6 trigger |
| `bundleUpsellFlag` | Scoring engine (gap check) | Shows secondary bundle product on result page |
| `bundleConsiderationFlag` | Q3A='both' | Signals neck+secondary interest; contributes to escalation |
| `neckResolution` | Q3A | Determines State 3 (XL) vs State 4 (Standard) for neck winners |

### Result State Quick Reference

| State | Condition | Hero | Bundle Upsell |
|---|---|---|---|
| 1 | winZone = eye | Eye Lift Wand | Your Best Face Set (if flag) |
| 2 | winZone = face | Liquid Facelift Wand | Your Best Face Set (if flag) |
| 3 | winZone = neck, XL | Neck XL Wand | Power Trio or Duo Set (if flag) |
| 4 | winZone = neck, Standard | Neck Standard Wand | Duo Set (if flag) |
| 5 | winZone = lip, eligible | Double The Plump | Ultimate Set (if flag) |
| 6 | bundle+escalation+gap≤10 | Power Trio / Essential | Ultimate Set (if escalation) |
| 7 | hardBundleTrigger | Essential / Ultimate | None |

---

---

## PART 8 — UTM ATTRIBUTION

All result CTAs include UTM parameters for Shopify analytics attribution:

```
?utm_source=quiz
&utm_medium=result
&utm_campaign=sbla-quiz-v1
&utm_content=result-{resultState}
```

The `utm_content` value is the numeric result state (1–7), enabling per-state conversion tracking in analytics.

---

*End of document. For code questions, reference `quiz.html` lines 1502–2130 (the `<script>` block).*
