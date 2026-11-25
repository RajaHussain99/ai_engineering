# Level 9: AI system design

**Phase 5: Production** · Previous: [Level 8](../4-evaluation-and-trust/08-ai-security.md) · Next: [Level 10](10-llmops.md)

**Goal:** design a complete AI application and be able to explain each choice in terms of
quality, speed, cost, reliability and the user's experience.

**Skip this level if** you have written and defended a design document for an AI feature
that then went live.

System design is deciding how the pieces fit together before you build. Earlier levels
taught the pieces. This one is about choosing among them and justifying the choice.

---

## 1. A design framework

Go through the same five questions every time.

**1. Who is the user and what does success look like, in numbers?**
Not "a helpful chatbot" but "answers 70% of customer questions without a human, with fewer
than 2% wrong answers".

**2. What are the limits?**
How fast must it respond? What can each request cost? How many requests a day? How
sensitive is the data?

**3. What is the simplest design that could meet them?**
Climb this ladder only as far as you need:

```
One prompt
   ↓ not enough knowledge?
Prompt + documents pasted in
   ↓ too many documents?
RAG
   ↓ needs several fixed steps?
Workflow
   ↓ steps cannot be known in advance?
Agent
```

**4. How will you measure it?** Before and after launch (level 7).

**5. What happens when each part fails?**

**Worked example: support assistant for an online shop.**

| Question | Answer |
|---|---|
| User and success | Shoppers asking about orders and returns. Resolve 60% without a human; under 2% wrong answers. |
| Limits | First words within 2 seconds. Under 2 cents per conversation. 20,000 conversations a day. Order data is private. |
| Simplest design | Policy questions: RAG over 300 help articles. Order questions: one tool that looks up the logged-in customer's orders. A router decides which. No agent loop needed. |
| Measurement | 200 test questions; daily sample of live chats graded; track how often chats are passed to humans. |
| Failures | Model provider down → second provider. Order system down → "I can't check orders right now", offer a human. Unsure → hand over to a human. |

Notice the design is a router plus RAG plus one tool, not an agent. Starting simple is a
strength in design, not a weakness.

---

## 2. Where knowledge lives

Every fact the system needs must live somewhere. There are four places.

| Place | Good for | Example |
|---|---|---|
| **The prompt** | Small, stable, always needed | Company name, tone of voice, opening hours |
| **A search index (RAG)** | Large, changing, mostly static text | Help articles, manuals, policies |
| **A tool** | Live or per-user data | "Where is order #1234?", today's stock level |
| **Fine-tuned weights** | Style and behaviour, not facts | Always replying in the house style (level 13) |

For each source also decide:

- **Freshness.** How old may the answer be? Prices need to be live (tool). A returns policy
  that changes yearly can be indexed nightly (RAG).
- **Access.** Who may see it? (Level 8, section 5.)
- **Updating.** Who changes it, and how does the system pick up the change?

A common mistake is putting live data in a search index. The index says a product is in
stock; it sold out an hour ago.

---

## 3. Model routing and gateways

**Routing** means sending each request to the cheapest model that can handle it.

*Example:* a support assistant gets 20,000 conversations a day.

```
All to the large model:                           20,000 × 2.0¢ = $400 a day
Routed:  80% easy → small model   16,000 × 0.2¢ =  $32
         20% hard → large model    4,000 × 2.0¢ =  $80
                                                   $112 a day
```

A 72% saving, if your evals confirm the small model handles the easy 80% well.

Ways to route:

- **By type:** a quick classifier labels the request, and each label has an assigned model.
- **By escalation:** try the small model first; if it is unsure or a check fails, retry
  with the large one.

A **gateway** is one service that all your model calls pass through. Think of it as a
reception desk. It handles:

- Keys in one place, not scattered through your code.
- One log of every call and its cost.
- Rate limits and spending caps per team or per customer.
- Automatic fallback to another model or provider.
- Switching models by changing a setting, not rewriting code.

Open-source gateways exist (LiteLLM is a common one), and so do hosted ones.

---

## 4. Speed

Users notice delay. Start by writing a **time budget**: where does each second go?

*Example for a RAG answer:*

| Step | Time |
|---|---|
| Rewrite the question | 0.4s |
| Search | 0.1s |
| Rerank | 0.3s |
| Model starts answering | 0.6s |
| **Until the user sees something** | **1.4s** |
| Rest of the answer streams in | 3.0s |

Now you can see what to attack first.

**Ways to go faster, roughly from cheapest to most effort:**

1. **Stream.** The user starts reading at 1.4 seconds instead of waiting 4.4.
2. **Shorten the output.** Writing is the slow part. "Answer in under 100 words" can halve the time.
3. **Use a smaller model** where your evals say it is good enough.
4. **Run things at the same time.** Searching two sources one after another takes twice as
   long as searching both at once.
5. **Cut steps.** Does the question really need rewriting every time?
6. **Cache** (next section).
7. **Trim the input.** Five good chunks beat twenty mediocre ones.

**Perceived speed counts.** Showing "Searching your orders…" makes a three-second wait feel
fine; a blank screen for three seconds feels broken.

---

## 5. Caching

Caching means saving a result so you do not have to compute it again. Three kinds apply.

**Prompt caching.** The provider reuses its work on the unchanged start of your prompt
(level 6, section 7). Lower cost and faster replies. Almost always worth doing.

**Exact-match caching.** If the identical question arrives again, return the saved answer
without calling the model.

*Example:* "What are your opening hours?" is asked 3,000 times a day, word for word.
Answer once, save it, serve the copy.

**Semantic caching.** Return a saved answer when a new question *means* the same as an old
one, judged by embedding similarity (level 4).

```
"What time do you open?"  ≈  "When are you open?"   → reuse the answer
```

This is powerful and risky. Two questions can look alike and need different answers:

```
"Can I return shoes?"              → "Yes, within 30 days."
"Can I return shoes I've worn?"    → "No."
```

If the cache judges those the same, the second customer gets a wrong answer.

**Rules for caching safely:**

- Set the similarity bar high, and test with pairs like the one above.
- **Never share cached answers between users when the answer depends on who is asking.**
  "Where is my order?" must not return another customer's order.
- Give entries an expiry time, and clear them when the underlying facts change.

---

## 6. Reliability and failing gracefully

Assume every part will fail at some point and decide in advance what happens.

**The tools** (several from level 2):

- **Timeouts:** stop waiting after a set time.
- **Retries:** try again on temporary errors.
- **Fallback model:** a second provider when the first is down.
- **Circuit breaker:** if a service has failed ten times in a row, stop calling it for a
  minute instead of piling on more requests. Like a fuse: it trips to prevent further damage.

**Failing gracefully** means that when something breaks, the user still gets something
useful instead of an error page.

*Example ladder for a support assistant:*

```
Everything works       → full answer with order details
Order system down      → answers policy questions; says it can't check orders right now
Search down            → "I'm having trouble finding that. Here's our help centre link."
All models down        → "Our assistant is unavailable. Contact us at …"
```

Each rung is worse than the one above, but none is a crash.

**Guardrails** (level 8) belong here too: check what comes in and what goes out.

**Know when to hand over.** A system that says "I'm not sure, let me connect you to a
person" is more trustworthy than one that always answers.

---

## 7. Organising longer jobs

Not everything fits in one quick request.

**Immediate vs background.** A chat reply must come back in seconds. Summarising 500
documents may take ten minutes; no one should sit watching a spinner.

*The background pattern:*

```
1. User clicks "Analyse these 500 files"
2. Server replies at once: "Started. Job number 8841."
3. The job goes into a QUEUE
4. A WORKER program takes it from the queue and does the work
5. The user sees progress ("212 of 500 done") and is told when it finishes
```

A **queue** is a waiting line for jobs. A **worker** is a program that takes jobs from the
line. If jobs pile up, you add workers.

**Surviving restarts.** If the server restarts at file 300, the job should carry on from
300, not start again. That needs progress saved after each step. Tools built for this are
often called "durable execution" or workflow engines.

**Safe to repeat.** If a step runs twice because of a retry, nothing bad should happen
(the idempotency idea from level 2).

---

## 8. Monitoring and getting better

After launch, you need to see what is happening and feed it back.

**Four things to watch:**

| What | Examples |
|---|---|
| Quality | Share of sampled answers passing your graders; thumbs-down rate; hand-over rate |
| Speed | Typical and worst-case response times |
| Cost | Per request, per user, per day |
| Errors | Failed calls, timeouts, guardrail blocks |

For speed, look at the slow end, not only the average. "p95 = 6 seconds" means 5% of
requests take longer than six seconds. An average of 2 seconds can hide a lot of unhappy users.

**Collect feedback** cheaply: thumbs up/down, "was this helpful?", and indirect signs such
as the user asking the same thing again.

**The improvement loop** (the same one from level 7):

```
Watch production → find failures → understand the cause
       ↑                                    ↓
   release  ←  confirm with tests  ←  add to test set and fix
```

Teams that run this loop weekly end up with far better systems than teams that launch and
move on.

---

## 9. Case studies

Practise by designing these on paper with the five questions from section 1. Some pointers
on what makes each one interesting:

**A. Customer-support assistant over a help centre.** Worked through above. The core
tensions: accuracy vs coverage (answer more, or hand over more?), and keeping each
customer's data private.

**B. Coding assistant inside an editor.**
- Suggestions while typing must appear in a fraction of a second, so a small fast model.
  A "fix this bug" request can take a minute, so a large model with tools.
- The main challenge is context: which of 5,000 files matter right now?
- Easy to measure: did the developer accept the suggestion? Do the tests pass?

**C. Pulling data out of a million pages a day.**
- No one is waiting, so use batch processing at lower prices.
- Cost per page dominates: at a million pages, a tenth of a cent each is $1,000 a day.
- Send easy pages to a small model and hard ones (handwriting, tables) to a larger one.
- Low-confidence results go to a human review queue.

**D. Real-time voice agent.**
- People expect a reply within about a second, so every step is under pressure.
- Must cope with being interrupted mid-sentence.
- Answers must be short and spoken-style: no lists, no formatting.

For each, write down the design, then search for published engineering write-ups of
similar systems and compare.

---

## Estimating cost: a worked example

Being able to do this sum on the spot is a core design skill.

```
Feature: summarise each incoming support ticket.
Volume:  100,000 tickets a day
Input:   ~800 tokens of ticket + 200 of instructions = 1,000 tokens
Output:  ~100 tokens
Prices (illustrative): $1 per million input tokens, $5 per million output tokens

Input cost:   100,000 × 1,000 = 100 million tokens × $1  = $100
Output cost:  100,000 ×   100 =  10 million tokens × $5  =  $50
Per day:  $150         Per month:  about $4,500
```

Then ask what would change it: caching the 200 instruction tokens, batch pricing, a
smaller model, shorter summaries.

---

## Free resources

- Chip Huyen, "Building LLM applications for production": <https://huyenchip.com/2023/04/11/llm-engineering.html>
- Chip Huyen's blog: <https://huyenchip.com/blog/>
- Eugene Yan, LLM system patterns: <https://eugeneyan.com/writing/llm-patterns/>
- Applied LLMs: <https://applied-llms.org/>
- Google SRE book (free online): <https://sre.google/sre-book/table-of-contents/>
- LiteLLM, an example gateway: <https://docs.litellm.ai/>
- Anthropic engineering blog: <https://www.anthropic.com/engineering>

## Project: a design document

Write 6 to 10 pages designing a real product idea. No code needed.

- The problem, the users, and success measures with target numbers.
- Limits, including a time budget and a cost estimate with the sums shown.
- A diagram of the parts and how data flows between them.
- At least three decisions, each with the options you rejected and why.
- How you will evaluate it, the security risks, and what happens when each part fails.
- A launch plan and what you would watch in the first week.

Ask someone to pick holes in it for 30 minutes, then revise.

## Self-check

1. How do you decide between one prompt, a workflow, RAG and an agent?
2. Your slowest 5% of requests take twice the target time. List four things to try, cheapest first.
3. What is the danger in a semantic cache?
4. What does the user see when your model provider is down?
5. Estimate the monthly cost of a feature serving 100,000 requests a day. What do you need to know?
