# AuthReady

**AI-assisted prior authorization readiness check for BayCare Health System.**

A utilization-management nurse attaches documents from a patient's chart, selects the payer's published medical-necessity policy, and gets a criterion-by-criterion readiness verdict *before* the request is submitted. The point is to catch a missing document in ten seconds instead of discovering it in a denial letter two weeks later.

Single self-contained HTML file. No server, no build step, no dependencies.

---

## Quick start

1. Download `AuthReady-prototype.html`.
2. Open it in Chrome.
3. Click **AI service not connected** in the top-right, paste an OpenAI API key, press **Connect**.
4. Open a case from the worklist and press **Run readiness check**.

A guided walkthrough of the full demo flow is built in — sidebar → **Presentation guide**.

---

## The problem

Prior authorization is a documentation-completeness problem disguised as a clinical one. A request that reaches the payer missing one itemized therapy trial comes back as a denial or a records request, and the patient waits another one to two weeks while the appeal cycles. Nobody at the provider finds out what was missing until after it costs time.

An AAHKS survey found 71% of prior-auth denials for total joint procedures cite insufficient conservative-treatment documentation — either not attempted, or not attempted long enough. That is an infrastructure failure, not a clinical one, and it is fixable before submission.

## Why this is a gap and not a duplicate

BayCare's public AI work is documentation *capture* — ambient listening in the Oracle Health EHR, and a nurse voice-documentation pilot. Commercial AI prior-auth products (Availity AuthAI and similar) are built for the **payer** side: they help a health plan reach a determination faster.

AuthReady sits on the **provider** side of that wall, checking the packet against the payer's own published criteria before it is ever sent.

---

## How it works

```
Worklist  →  Case intake  →  Readiness report  →  Close gaps  →  Re-check  →  Submit
```

1. **Worklist** — pending authorization requests with per-row readiness status and the three success metrics.
2. **Case intake** — documents come from the patient's chart, not an upload. Tick what the check should read; the exact text the model receives is shown read-only.
3. **Policy selection** — one of four real, published payer policies. The criteria shown on screen are the criteria written into the system prompt, so the two can never drift.
4. **Readiness report** — per criterion: Met / Unmet / Unclear, the verbatim sentence of evidence relied on, and the reasoning. Plus denial-risk score, confidence, latency, and what the payer will ask for.
5. **Close the gaps** — attach a document the model flagged as missing, or add an addendum (filed separately; the original note is never altered), then re-run. The risk score moves.
6. **Submit** — simulated. No payer system is contacted.

### Prompt design

The system prompt is generated from whichever policy is selected. Its constraints are the interesting part:

- Use **only** the attached document text and the listed criteria — never outside knowledge of payer rules.
- Quote evidence **verbatim**. Never paraphrase a quote, never invent one.
- If a criterion isn't addressed, return `Unclear` and say what's missing. Do not infer.
- A generic "failed conservative treatment" does **not** satisfy a criterion requiring itemized modalities and dates — mark it Unmet and say why. (This mirrors CMS guidance, which states as much explicitly.)
- Return a fixed JSON object, one findings entry per criterion, in order.

The full prompt is viewable at runtime under **How this works**.

---

## The four payer policies are real and public

| Policy | Payer | Service |
|---|---|---|
| [CPB 0236](https://www.aetna.com/cpb/medical/data/200_299/0236.html) | Aetna | MRI and CT of the Spine |
| [CPB 0660](https://www.aetna.com/cpb/medical/data/600_699/0660.html) | Aetna | Knee Arthroplasty |
| [LCD L34220](https://www.cms.gov/medicare-coverage-database/view/lcd.aspx?LCDId=34220) | Medicare Part B | Lumbar MRI (with NCD 220.2) |
| [LCD L36575](https://www.cms.gov/medicare-coverage-database/view/lcd.aspx?LCDId=36575) | Medicare Part B | Total Knee Arthroplasty |

Criteria are **paraphrased in our own words** and cited by ID with a link to the source. The policy documents are copyrighted and are not reproduced.

---

## Edge cases handled

| Case | Behavior |
|---|---|
| Scanned fax with no text layer | Rejected **before** any API call is spent, with the reason stated |
| Empty or too-short attachment | Rejected client-side |
| No API key | Blocked with a prompt to connect |
| HTTP 401 | "API key rejected" + the provider's message; no results render |
| HTTP 429 | "Rate limited or out of quota" |
| HTTP 404 | "Model not available on this key" |
| Network failure | "Could not reach the API" |
| Reply isn't valid JSON | Caught at parse, re-run offered |
| Unexpected readiness label / no findings | Validated and rejected |

The unreadable-fax case is on the worklist as a patient (Kowalski) so it can be demonstrated, not just described.

---

## Try your own case

**Patient lookup** lets anyone build a test case: label it, pick any of the four policies, paste a note, and optionally state what you expect the verdict to be *before* running. The app then scores your prediction against the model's answer.

Four loadable sample notes exercise different branches — clean approval, one gap, a note too sparse to judge, and a document for an entirely different procedure.

---

## API key handling

No key is committed to this repository. The source carries the placeholder:

```js
API_KEY = "YOUR_KEY_HERE";
```

The key is supplied at runtime through the header panel, held in memory for the browser tab only, sent directly to `https://api.openai.com/v1/chat/completions`, and cleared on reload. Model is selectable (gpt-5 / gpt-4o / gpt-4o-mini).

**Do not commit a key.** In production this is a server-side credential, never a browser field.

---

## What's real vs. simulated

**Real** — the API call, the structured JSON parse, every finding and citation, latency measurement, denial-risk score, confidence, the re-check loop, and the input validation.

**Simulated** — the five patients, their charts, and "Submit to payer." All records are synthetic. **No real PHI is present anywhere in this repository.**

---

## What a production version would require

- **Server-side inference behind a signed BAA.** No PHI may transit a browser to a public API endpoint. This is the single largest gap between the prototype and a deployable system.
- **Policy ingestion with versioning and effective dates**, not four hand-summarized rulebooks. Payers revise criteria; a stale rulebook is worse than none.
- **EHR integration** (BayCare runs Oracle Health) to read the chart instead of fixtures.
- **Evaluation against a labelled set** of historically approved and denied requests, with monitoring for policy drift.
- **Audit logging** of every check, every citation, and the reviewing nurse's sign-off.

## Limits

Advisory only. Output is **not a coverage determination**, not a payer decision, and not medical advice. A licensed nurse reviews every submission. The tool is deliberately built not to help a user satisfy a criterion that the clinical facts don't support — where a gap can't be closed by documentation, it says so.

---

## Repository

```
AuthReady-prototype.html   ← the deliverable; open this in a browser
AuthReady.dc.html          ← authoring source
README.md
```

AI tools used: Claude (scaffolding and UI).
