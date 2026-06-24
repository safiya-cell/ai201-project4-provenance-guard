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
