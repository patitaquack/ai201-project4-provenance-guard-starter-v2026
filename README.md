# Provenance Guard

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first.**
>
> Once `python test.py` passes, in one terminal:
>
> ```bash
> python app.py
> ```
>
> and in another:
>
> ```bash
> curl -X POST http://127.0.0.1:5000/ping \
>   -H "Content-Type: application/json" \
>   -d '{"message": "hello"}'
> ```
>
> That's a web service. Something sent it text and it sent something back.
> Everything this unit is filling in the middle.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     Unit 7 asks for the first five sections. Unit 8 adds the five below.

     Everything is pasted as TEXT. A pasted curl response gets full credit;
     a picture of your terminal gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 7 — THE BUILD ═══════════════════════ -->

## What This Does

<!-- The path a submission takes, from arriving to the label a reader sees.
     Six or seven steps — this is the trace you wrote in Milestone 1. -->

Provenance Guard is a Flask web service that examins writing and estimates wether it was AI- generated text or Human written. It has two detection signals and it allows writers to appeal a decision.

- The user submits their writing along with a creator ID through   the /submit endpoint.

- The application checks that the text is valid and makes sure it is not empty or too long.

- The GPT-2 model analyzes the text to see how predictable the writing is and gives it a score between 0 and 1.

- The stylometry function looks at things like sentence lengths, word variety and punctuation to calculate another score.

- Both scores are combined into one final score. I decided to give each method an equal weight of 50%.

- Based on  final score the application labels the writing as human, unsure, or AI. It also saves the results in an audit log and gives the submission a unique content ID.

- If someone disagrees with the result they can submit an appeal using their content ID. The appeal is saved in the audit log with an under_review status.

The application is meant to identify patterns in writing, but the results are not proof that someone used AI.



---

## Signals and Scoring

<!-- For each signal: what property it measures, why that might differ between
     human and AI writing, and WHAT IT CAN'T SEE.

     The blind spot is not optional. If you can't name it you don't understand
     the signal yet — and unit 8's attack set is built out of blind spots. -->

### Signal one — the model signal

**What it measures:**

looks at how predictable the writing is and give s it a score between 0  - 1. A higher score means the text has more patterns associated with AI- generated writing.

**Why that might differ between human and AI writing:**

AI- genreated writing can be a lot more predicatable due to language models using common words and sentence patterns. Human writing may be less predictable because people have difference writing styles and all use different choice of words in their writing.

**What it can't see:**

It can not know who wrote the text. Someone who writes simples or repeatitive sentences may receive a high AI score even when they wrote everything themselves. It can halso have trouble recognizinv AI generated writing that has been edited.
### Signal two — the style signal

**What it measures:**

It looks at how a text is written and measures differences in lenghts (in sentences), how many unique words were used, and how often punctuation appears.

**Why that might differ between human and AI writing:**
Humans tend to use a variation of lengths in their sentences,and different vocabulary, and punctuation. AI generated writing can sometimes follow more consistent patterns.

**What it can't see:**

Who wrote the text. A human might write short and repeatitive sentences, while AI will generate writing in difference sentence lengths and vocanulary. 

### The combining rule

<!-- Write the rule in words, then give the numbers. What happens when the two
     signals disagree? -->

**The rule:**

I  decided to give both the GPT-2 model and stylometry an equal weight because both have their limitations. I didn't want to depend on just one method to decide whether someone used AI. By combining both scores, the application can give an unsure result when the scores are different instead of assuming the writing was AI-generated.

**The numbers:**
- GPT-2 model: 50% (0.50)
- Stylometry: 50% (0.50)
- Human threshold: 0.30
- AI threshold: 0.80

**Where it lives:** `scoring.py::combine_signals`


<!-- ⚠️ The grader checks your code against that line. If your rule lives
     somewhere else, say where. -->

**A case where my two signals split, and what my rule does with it:**

When I tested a paragraph about a garden GPT-2 gave it a score of 0.6724 but stylometry gave it 0.3314. This showed me that both methods can look at the same writing and give different results.
When I combined the scores, I got 0.5019. Since that falls between my thresholds the application returned "unsure." I think this is a lot better than automatically labeling someone's writing as AI when the two methods don't agree.


---

## Label Variants

<!-- The exact text of all three labels, as a reader would see them, and the
     score range each covers.

     Write them for someone who has never heard the word "threshold". -->

| Label | Score range | The exact text a reader sees |
|---|---|---|
| high-confidence human |0.00 - 0.30| This writing appears more consistent with human-written text.|
| unsure |  + 0.30, and under 0.80 | We cannot confidently determine whether this writing was AI-generated or human-written. You can appeal this result. |
| high-confidence AI |0.80 - 1.00 | This writing shows patterns associated with AI-generated text, but this result is not proof of AI use. You can appeal this decision.|

**Why I worded the "unsure" one this way:**
<!-- It's the hardest of the three. It has to admit uncertainty without
     sounding like an accusation, because the person reading it may well have
     written every word themselves. -->

I wanted to ensure that when the user reads an unsure message, they dont feel like they are being accused of using AI. Honesty is necessary but my priority is to protect the writer, and give them a chance to appeal if they disagree.



---

## Sample Run

<!-- One submission and one appeal, pasted as text: the request, the response,
     and the log entries. -->

**A submission**

<!-- The response needs content_id, guess, confidence, label, AND both
     signal scores - model_score and style_score. Unit 8's tools read those
     two off the response, so leaving them out costs you next unit's
     diagnostics. -->



curl -X POST http://127.0.0.1:5001/submit \
  -H "Content-Type: application/json" \
  -d '{"text": "The garden was quiet. Birds sang softly in the trees. Suddenly, a strong wind swept through the courtyard, scattering leaves across the path and sending everyone running toward the house.", "creator_id": "writer123"}'


{
  "combined_score": 0.5018956442468022,
  "confidence": 0.003791288493604439,
  "content_id": "17eb24c4-3cf2-4764-a8b4-c2f99b0d7ae7",
  "creator_id": "writer123",
  "guess": "unsure",
  "label": "We cannot confidently determine whether this writing was AI-generated or human-written. You can appeal this result.",
  "model_score": 0.6724,
  "style_score": 0.3313912884936045,
  "text_length": 187
}





**An appeal**


curl -X POST http://127.0.0.1:5001/appeal \
  -H "Content-Type: application/json" \
  -d '{
    "content_id": "17eb24c4-3cf2-4764-a8b4-c2f99b0d7ae7",
    "reasoning": "I wrote this paragraph myself and would like the decision reviewed."
  }'

  {
  "content_id": "17eb24c4-3cf2-4764-a8b4-c2f99b0d7ae7",
  "message": "Your appeal has been received and is under review.",
  "status": "under_review"
}




**What the log shows afterwards**

curl "http://127.0.0.1:5001/log?limit=2"


{
  "count": 2,
  "entries": [
    {
      "combined_score": 0.5018956442468022,
      "content_id": "17eb24c4-3cf2-4764-a8b4-c2f99b0d7ae7",
      "creator_id": "writer123",
      "guess": "unsure",
      "label": "We cannot confidently determine whether this writing was AI-generated or human-written. You can appeal this result.",
      "model_score": 0.6724,
      "status": "decided",
      "style_score": 0.3313912884936045,
      "timestamp": "2026-11-16T04:21:40+00:00"
    },
    {
      "content_id": "17eb24c4-3cf2-4764-a8b4-c2f99b0d7ae7",
      "creator_id": "writer123",
      "event": "appeal",
      "reasoning": "I wrote this paragraph myself and would like the decision reviewed.",
      "status": "under_review",
      "timestamp": "2026-11-16T04:37:53+00:00"
    }
  ]
}





---

## How I Used AI

**Moment 1**

- *What I asked for:*
I asked GPT how I can combine the GPT-2 model score and the stylometry score
- *What came back:*
It explained how weighted averages work and showed me how giving each score 50% would affect the result
- *What I changed:*
I kept the weights equal and updated the file with combine both scores, and I tested the function to make sure it returned the correct number

**Moment 2**

- *What I asked for:*
To help me understand an error when dealing with Flask server
- *What came back:*
It explained that the port I was using was already being used and showed me how to stop and existing process and use a different port
- *What I changed:*
I checked what was using the port and stopped the old processes and changed my server to run a different port.


<!-- ═══════════════════════ UNIT 8 — THE TEST ═══════════════════════ -->

---

## Rate Limiting

**My limits:** 10 per minute, 200 per day


**Why those numbers and not others:**
<!-- Think about two people: a real writer submitting their own work a few
     times an hour, and a script sending a thousand variations to map your
     thresholds. Your numbers have to be liveable for the first and hostile to
     the second. -->

     I chose these limits because I want users to be ableto sumbit their work without constantly getting rejected. I also do not want hundres of requests to overload the server. 10 request is a reasonable number for a user that uses the program regularly.

**What a caller counts as:** <!-- per address, or per creator_id? They fail
differently — one script can look like a thousand callers, and one household
can look like one. -->

The program uses the creator_id to identify the caller, but when there is no valid createor_id it uses the caller IP address. So, multiple requests using the same creator ID count towards the same limit.

**The run of status codes when I pushed past it:**

<!-- python run_attacks.py --flood 15 -->

200
200
200
200
200
200
200
200
200
200
429
429
429
429
429


---

## Attack Run — Before

<!-- Every attack with its outcome and a held-or-broke mark.
     `run_attacks.py --label before` produces the table. -->



**Ten audit log entries from the run**

<!-- Choose entries that show the interesting failures, not the ten easiest. -->

```json

```

**The single worst outcome** — the one I'd least want to explain to a writer
whose work got caught by it:



---

## Run Log and Verdicts

<!-- Your five criteria across three trials. `run_eval.py --label before`. -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

**Real output from one trial**, pasted as text, naming the file and function
that produced it:

```json

```

**Diagnoses**

<!-- For each miss AND each broken attack: the stage and the mechanism.

     Your service has five stages: the route, signal one, signal two, the
     combining rule, and the label mapping.

     Not a diagnosis:  "The evasion attacks got through."
     A diagnosis:      "Four evasion inputs got through. All four had typos
                        added to AI-generated text. My second signal measures
                        word variety, and typos raise word variety — so it read
                        them as more human. My combining rule weights that
                        signal at 0.6, so it dragged the whole score down."

     Look for a pattern. Five failures on one signal is one problem, not five. -->



---

## The Improvement

**What I changed:**

**Which diagnosis pointed at it:**

### Attack Run — After

<!-- Same format. `run_attacks.py --label after` -->



**Did it help, and how do I know:**

<!-- Widening the unsure band usually fixes one problem and creates another.
     Reporting that trade honestly is worth full credit. -->



---

## What's Still Broken

<!-- For each attack still getting through and each criterion still missed:
     what you'd do, and why you stopped. -->



**The trade I'd make if this ran for real:**

<!-- Two sentences. Fewer wrong accusations means more AI text getting through.
     Say which way you'd go and WHO PAYS FOR IT. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 7

       [ ] criteria.md has five numbered criteria, each naming a target
       [ ] Each has a reason underneath
       [ ] POST /submit returns content_id, guess, confidence, label,
           model_score and style_score
       [ ] Two signals that measure DIFFERENT things, each with a named blind spot
       [ ] Three label variants, all reachable, written out in full
       [ ] POST /appeal changes status and writes to the log
       [ ] Signals and Scoring names scoring.py::combine_signals
       [ ] Sample Run: a submission AND an appeal, as pasted text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN

     SUBMISSION CHECKLIST — unit 8

       [ ] Rate limiting on /submit, with your numbers justified
       [ ] The pasted run of status codes showing 429
       [ ] Audit log with all nine fields, recording rejections too
       [ ] The attack set run IN FULL, every input with an outcome
       [ ] Ten audit log entries pasted as text
       [ ] Five criteria across three trials
       [ ] A diagnosis for every miss and every broken attack — stage AND mechanism
       [ ] One improvement, with Attack Run — After
       [ ] What's Still Broken, including the production trade
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
