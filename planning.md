1. Someone sends a text to your API and you give it an ID and save it.
2. You run two checks on the text producing a number between 0 and 1 detecting AI.
3. Combine two numbers into one confidence score, then convert that score into a plain-English label like "likely human," "uncertain," or "likely AI."
4. Save everything (the text, both scores, the final label, who submitted it, when) to a log, then send the label and score back to the user.

The appeal states whether you got it wrong and you mark the submission as appealed in the log and save their reason. A human can review it later.

Your two detectors need to answer:

What am I actually measuring in the text?
Why would AI text score differently than human text on this?
When would I get it wrong?

For example: one detector might measure how predictable the word choices are. AI text tends to be very predictable. Human text is messier. But a careful human editor also produces predictable text — that's the blind spot.

The false positive problem in one sentence:
When you wrongly flag a human as AI, your confidence score should show the uncertainty (0.84, not 1.0), your label should say "likely" not "definitely," and the appeal endpoint lets them push back so a human can look at the raw scores and decide.

Your API is four endpoints:

POST /submit — takes text, returns a label and score
GET /submission/{id} — look up a past result
POST /appeal — flag a result as wrong
GET /audit-log — see all past submissions

## Signal 1: Perplexity (how predictable is the text?)##

What it measures: AI text picks the most appropriate next word at each step, so a language model
finds it very predictable (low perplexity). 

Output: A float in [0, 1]. 1 = very predictable = likely AI.
0 = very unpredictable = likely human. 

Blind spot: Human editors write language similar to AI and technical text
scores as predictable regardless of who wrote it. Texts under 50 words produce
unreliable estimates.


## Signal 2: Predictability ##

What it measures: AI text is predictable while human writing varies.

Output: A float in [0, 1]. 1 = low variance = likely AI.
0 = high variance = likely human. 

Blind spot: Fewer than ~5 sentences produces unreliable variance. A
consistent human writer shows low
variance. This signal is weaker than Signal 1 on short texts.


## Combining into a confidence score ##

combined_score = 0.6 × signal_1 + 0.4 × signal_2

Signal 1 is weighted higher because perplexity is more reliable on short text.
Combined score is in [0, 1]. This is the number that drives the label.
0.6 is a weak signal shows that AI is likely.

## Architecture ##
SUBMISSION FLOW
===============

POST /submit
  { text, creator_id }
        |
        v
  Submission Handler
  - assigns submission_id
  - validates input
        |
        v
  Signal 1: Perplexity Extractor
  { raw text }  -->  { signal_1_score: float [0,1] }
        |
        v
  Signal 2: Burstiness Extractor
  { raw text }  -->  { signal_2_score: float [0,1] }
        |
        v
  Confidence Scorer
  { signal_1_score, signal_2_score }
  -->  combined_score = 0.6 x s1 + 0.4 x s2
        |
        v
  Label Generator
  { combined_score }  -->  { label: string }
        |
        v
  Audit Logger
  writes: { submission_id, creator_id, text_hash,
            signal_1_score, signal_2_score,
            combined_score, label, timestamp,
            status: "complete" }
        |
        v
  Response
  { submission_id, label, confidence,
    signal_scores: { signal_1, signal_2 } }


APPEAL FLOW
===========

POST /appeal
  { submission_id, creator_id, reason }
        |
        v
  Appeal Handler
  - looks up submission_id in audit log
  - verifies creator_id matches
  - checks status == "complete"
  - updates status --> "under_appeal"
  - appends appeal_reason, appeal_timestamp
        |
        v
  Response
  { submission_id, status: "under_appeal", message }

## AI Tool Plan ##
M3 — Submission endpoint + Signal 1

I will ask for a minimal Flask app skeleton with POST /submit wired up.
The extract_perplexity_score(text: str) -> float function returning
a normalised score in [0, 1].

Ask to call extract_perplexity_score directly on three inputs: a clearly
AI-generated paragraph, a clearly human-written paragraph, and a 5-word
sentence.
Check that the AI paragraph scores higher than the human paragraph.
Check that the short text returns a score near 0.5 with a warning flag.
Do not wire into the endpoint until these three checks pass.


M4 — Signal 2 + Confidence scoring

i will ask for the extract_burstiness_score(text: str) -> float function.
The compute_confidence(s1: float, s2: float) -> float function
implementing 0.6 x s1 + 0.4 x s2.

Ask to run both signals on the same three test inputs from M3.
Confirm Signal 2 is directionally consistent with Signal 1 on clear cases.
Confirm compute_confidence output is always in [0, 1].
Confirm the AI paragraph combined score exceeds 0.75 and the human
paragraph falls below 0.40, or adjust thresholds if not.



M5 — Production layer (labels + appeals)

i will ask to generate_label(score: float) -> dict returning label text and tier.
POST /appeal endpoint.
GET /submission/{id} endpoint.
GET /audit-log endpoint with optional creator_id filter.

Ask to call generate_label with 0.2, 0.55, and 0.85 — confirm all three label
variants are reachable and match the exact text in this spec.
Submit a test record, appeal it, call GET /submission/{id}, confirm status
changed to "under_appeal" and appeal_reason is present.
Appeal with a mismatched creator_id — confirm 403.
Appeal a record already "under_appeal" — confirm 409.
