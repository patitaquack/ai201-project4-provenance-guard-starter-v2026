# Milestone 1 — Trace a submission before building one

This is the plan. Nothing here is code yet. The README's **What This Does** and
**Signals and Scoring** sections come from this file.

---

## 1. The path: from arriving text to the label a reader sees

1. **`POST /submit` (app.py), the front door.** JSON comes in with `text` and `creator_id`.
   The route checks the input: text that is missing, empty, too short or enormous
   gets refused here with a 400 and a plain explanation, plus a rejection entry in the log.
   *Adds:* a known-good input, and a fresh `content_id` (uuid) that names this submission from now on.

2. **Signal one: `detector.model_signal(text)`.** gpt2 runs locally and measures how
   predictable the text is (perplexity), squashed onto 0–1.
   *Adds:* `model_score` (higher means more AI-like).

3. **Signal two: `stylometry.style_signal(text)`.** Measures the shape of the text:
   sentence-length spread, type-token ratio and punctuation density, each scaled and
   flipped so that higher means more AI-like, then blended.
   *Adds:* `style_score` (higher means more AI-like).

4. **Combining rule: `scoring.combine_signals(model, style)`.** Two numbers become one.
   *Adds:* `combined_score` (0 = confidently human, 1 = confidently AI).

5. **Label mapping: `scoring.score_to_label(score)`.** The score is checked against
   `HUMAN_THRESHOLD` and `AI_THRESHOLD` in config.py.
   *Adds:* `guess` (`human` / `unsure` / `ai`) and the sentence a person reads.

6. **Audit log: `audit.log_decision(...)`.** One JSON line in `logs/audit.jsonl` holding
   all nine fields, with `status: "decided"`.
   *Adds:* a permanent record that `/appeal` and next unit's diagnostics depend on.

7. **The response.** JSON goes back to the caller: the label, the content_id, and both
   signal scores. The reader sees the label sentence and the content_id they would need
   to appeal.

*(Side path)* **`POST /appeal`.** The writer sends the `content_id` and their reasoning. The route
checks that the id exists in the log, then `audit.log_appeal(...)` appends an `under_review`
entry. The original decision is never overwritten.

---

## 2. The two signals and their blind spots

### Signal one: model signal (`detector.py`, given)

- **Measures:** perplexity, meaning how surprised gpt2 is by each next word. Low perplexity gives a high score.
- **Why it might differ:** a language model writes by picking likely next words, so its
  output is text that a model finds easy to predict. Humans make odd, specific word choices.
- **What it can't see:** *who* wrote the text. It rewards common words in plain, ordinary
  sentences, so second-language writers, younger writers and anyone writing simply on
  purpose score as machine-like. The reverse also happens: ornate prose full of rare words, or AI text
  pushed toward unusual wording ("use obscure vocabulary"), scores as human. It also only
  reads the first 400 tokens, so whatever comes after that is invisible to it.

### Signal two: style signal (`stylometry.py`, mine to build)

- **Measures:** the shape of the text with no meaning involved. That covers variation in sentence length,
  vocabulary variety (unique words ÷ total words) and punctuation density and variety.
- **Why it might differ:** generated prose tends toward even, mid-length sentences, a
  narrower vocabulary and tidy punctuation. Human writing is lumpier, with a three-word
  sentence sitting next to a forty-word one.
- **What it can't see:** the meaning, and whether a change was made on purpose. Any
  surface edit that leaves the substance alone still moves the score: typos (which inflate
  vocabulary variety), splitting or merging sentences, added dashes or semicolons, a pasted quotation.
  All of these are cheap to do deliberately. It is also length-sensitive, because type-token ratio
  drops as text gets longer, and a single sentence has no spread at all.

**The important overlap:** both signals read plain, evenly paced, simple-vocabulary
writing as AI. On exactly the writers most at risk, the second signal does **not**
act as a check on the first. It agrees with it.

---

## 3. The false positive, traced

**Who:** a student writing in their second language submits a paragraph they wrote
themselves. Short sentences, words they're sure of, and some repetition.

| Step | What happens | Likely value |
|---|---|---|
| model signal | common words, predictable, so low perplexity | ~0.80 |
| style signal | even sentence lengths, repeated words, few commas | ~0.70 |
| combined (50/50 placeholder) | | ~0.75 |
| label (AI_THRESHOLD 0.65) | **high-confidence AI** | |

*(These are estimates. Check them in Milestone 4 by adding a plainly written human
paragraph to the calibration inputs.)*

- **What the label says to them:** with the placeholder text, a flat claim that their work is AI.
  That is an accusation they can't disprove, about the text they worked hardest on.
- **What they can do:** appeal with their `content_id`. That moves the item to
  `under_review` and nothing more. Nothing gets re-checked automatically. So the response
  **must** hand them the content_id and tell them, in plain words, that they can appeal and how.
- **What this decides for later (Milestone 4/5):**
  - The cost is one-sided. Missing AI text costs the site a little, while a wrong accusation costs this
    person a lot. So the bar for the "AI" label should be high (above the 0.65 placeholder).
  - Consider making the "AI" label require **both** signals to be high, rather than letting an
    average carry one strong signal over the line. Even so, this writer trips both, which is
    why the threshold and the wording matter more than the weights.
  - The "AI" and "unsure" labels should describe the *text*, not the person ("this reads
    like…", not "you used AI"), and both should point to the appeal.

> ✏️ **Your call:** the actual thresholds and weights. Write down the numbers you pick
> and the reason, which is this trace.

---

## 4. Endpoint contracts

### `POST /submit`

**Request**

| Field | Type | Required | Rules |
|---|---|---|---|
| `text` | string | yes | after trimming: not empty, at least ~3 sentences/20 words, at most ~10,000 characters |
| `creator_id` | string | yes | non-empty; identifies the writer for the log and the appeal |

**Response, 200**

| Field | Type | Meaning |
|---|---|---|
| `content_id` | string (uuid) | the id for this submission. Needed for `/appeal` |
| `guess` | `"human"` \| `"unsure"` \| `"ai"` | machine-readable verdict |
| `confidence` | float 0–1 | how far the score sits from a coin flip: `abs(combined_score − 0.5) × 2`. 0 = no idea, 1 = certain |
| `label` | string | the exact sentence shown to the reader |
| `model_score` | float 0–1 | signal one. Higher = more AI-like |
| `style_score` | float 0–1 | signal two. Higher = more AI-like |
| `combined_score` | float 0–1 | output of `combine_signals` |
| `status` | `"decided"` | lifecycle state |
| `appeal_hint` | string | tells the writer how to appeal using `content_id` |

**Errors**, all shaped as `{"error": "<code>", "message": "<plain sentence>"}` and all logged with `log_rejection`:

| Status | `error` | When |
|---|---|---|
| 400 | `invalid_json` | body isn't a JSON object |
| 400 | `missing_field` | `text` or `creator_id` absent, or not a string |
| 400 | `text_too_short` | empty, or too short to score |
| 413 | `text_too_long` | over the length cap |
| 503 | `detector_unavailable` | model couldn't load |
| 429 | `rate_limited` | unit 8 (already wired in app.py) |

### `POST /appeal`

**Request**

| Field | Type | Required | Rules |
|---|---|---|---|
| `content_id` | string | yes | must match a submission in the log |
| `reasoning` | string | yes | non-empty. Why the writer thinks the label is wrong |

**Response, 200**

| Field | Type | Meaning |
|---|---|---|
| `content_id` | string | echoed back |
| `status` | `"under_review"` | new state |
| `message` | string | plain confirmation, e.g. "Thanks, your appeal is recorded and a person will review it." |

`creator_id` isn't asked for. It's taken from the original decision in the log and passed to `log_appeal`.

**Errors**

| Status | `error` | When |
|---|---|---|
| 400 | `invalid_json` / `missing_field` | body isn't an object, or a field is missing or empty |
| 404 | `unknown_content_id` | no submission has that id |
| 409 | `already_under_review` | *(my choice: reject a duplicate appeal)* |

**Read-back check:** could someone build against this without asking me anything?
The questions still open are the exact length limits and whether a duplicate appeal
should be a 409 or a friendly 200. Decide both before Milestone 3.
