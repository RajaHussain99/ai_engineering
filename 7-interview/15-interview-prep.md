# Level 15: Interview preparation

**Phase 7: Interview** · Previous: [Level 14](../6-frontier-and-proof/14-portfolio-projects.md)

**Goal:** turn everything from the earlier levels into interview performance. There are no
new concepts here. This level is practice at scenario questions, system design, debugging
cases, trade-offs and defending your own decisions.

Each section gives sample questions with an outline of a strong answer. Do not memorise
them. Notice the pattern they share:

1. **Ask or state what you need to know first.**
2. **Start with the simplest thing that could work.**
3. **Say how you would measure it.**
4. **Name the trade-offs.**

---

## 1. Strategy and positioning

**Know which job you are applying for.** Titles vary, but roles fall into rough groups:

| Role | Main work | Levels that matter most |
|---|---|---|
| Applied AI / AI engineer | Building product features on existing models | 2 to 10 |
| Backend engineer working on AI features | General software with some AI | 0, 2, 9, 10 |
| Machine learning engineer | Training and serving models | 1, 7, 13, plus maths |
| Forward-deployed / solutions engineer | Building with customers directly | 2 to 9, plus communication |

Read the job advert closely and prepare for what it lists.

**A one-minute introduction.** Build it around two projects with results.

> "I'm a software engineer who has spent the last year building applications on language
> models. My main project is a question-answering system over housing regulations: I got
> search accuracy from 74% to 91% by adding keyword search and reranking, and it's live
> with a few hundred users. I also built an agent that fixes failing tests in small code
> projects and measured it at 68% success over 50 tasks. I'm most interested in search and
> in measuring quality, which is why this role caught my eye."

Under a minute, specific, with numbers, ending on why this job.

**CV bullets: action plus number.**

| Weak | Strong |
|---|---|
| "Worked on a RAG chatbot" | "Built a document Q&A system; raised search recall from 74% to 91% with hybrid search and reranking" |
| "Used LLMs to improve support" | "Cut cost per support answer by 70% by routing simple questions to a smaller model, with no drop in measured quality" |

---

## 2. RAG scenarios

**Q: "Our RAG system's answers are wrong about 20% of the time. How do you find out why?"**

Outline:
- First, look at real failures. Take 50 wrong answers and read them.
- For each, ask: was the right passage found? This splits the problem in two (level 7, section 8).
- If it was not found: it is a search problem. Look at how documents are split, add keyword
  search, add a reranker, rewrite questions.
- If it was found but the answer is still wrong: it is an answering problem. Look at the
  prompt, at whether too many passages are drowning the right one, and at the model.
- If the passage itself is wrong or outdated: it is a data problem.
- Then fix the largest group first and re-measure.

The strong signal here is separating search from answering before proposing any fix.

**Q: "Search keeps missing exact product codes."**
Meaning-based search treats similar-looking codes as the same. Add keyword search and
combine the two (level 4, section 6).

**Q: "The document collection grows from 10,000 to 50 million. What breaks first?"**
- Comparing against every document becomes too slow; you need an approximate index (level 4, section 5).
- Memory and storage for the vectors.
- The time and cost to embed everything, and to redo it if you change embedding model.
- Quality: more documents means more near-misses, so reranking and filtering matter more.
- Keeping it up to date as documents change.

**Q: "How do you stop one customer seeing another's documents?"**
Filter by permission at search time, so forbidden documents never reach the model. Never
rely on telling the model to keep secrets (level 8, section 5).

---

## 3. Agent scenarios

**Q: "When would you not use an agent?"**

Outline: when the steps are known in advance. Then a fixed workflow is cheaper, quicker,
more predictable and easier to debug. Give an example: "Classify the email, then draft from
the matching template" is a workflow. Use an agent when the route depends on what is found
along the way, such as debugging or open-ended research. And say that you would try the
workflow first and move to an agent only if measurement showed it falling short.

Interviewers ask this to see whether you reach for agents by reflex. Showing restraint is
the right answer.

**Q: "The agent keeps making the same tool call over and over."**
- Read the trace: what does the tool return? Usually an unhelpful error or an empty result,
  so the agent sees no reason to change course.
- Fix the tool's message so it says what to do next.
- Add a check for repeated identical calls that stops or redirects the agent.
- Check whether the history has grown so long the agent has lost track (level 6).
- Add a step limit as a backstop.

**Q: "How do you evaluate an agent that takes 40 steps?"**
- Judge the end result by checking the actual state, not what the agent says.
- Run each task several times and report a success rate.
- Also track steps, cost and any forbidden actions.
- Give part marks for part-completed tasks.
- Close loopholes, such as "passing" by deleting the failing test (level 7, section 9).

**Q: "How do you make an agent with write access safe?"**
Smallest possible access; human approval for anything irreversible; a sandbox for running
code; limits on steps and spending; a log of every action; and assume it will sometimes be
tricked, so design for limited damage (level 8, section 4).

---

## 4. Fine-tuning and model adaptation

**Q: "Prompting, RAG or fine-tuning: how do you choose?"**

Outline:

| Need | Use |
|---|---|
| Better instructions or format | Prompting |
| Knowledge the model lacks, or that changes | RAG |
| A consistent style prompts cannot hold, or lower cost at high volume | Fine-tuning |

Start with prompting because it is quickest to change. Add RAG for knowledge. Consider
fine-tuning last, and only with a test set and a baseline, because it is slow to change and
must be redone when base models improve. They also combine: a fine-tuned model can still
use RAG.

A crisp line: "Fine-tuning changes how the model behaves. RAG changes what it knows."

**Q: "How much data do you need?"**
It depends on the task. A few hundred good examples can be enough for a narrow one. Quality
and variety matter more than count. Start small, measure, and add data where the model fails.

**Q: "How do you know the fine-tune worked?"**
Compare it on a held-back test set against the same small model with a good prompt and
against a larger model with a good prompt. Check quality, cost and speed. Also check it has
not got worse at other things.

---

## 5. Operations and production debugging

**Q: "Response times doubled overnight and nobody released anything. Where do you look?"**

Outline, working from outside in:
1. Is it everything or one feature? One region? One model?
2. Check the model provider's status page; the slowdown may be theirs.
3. Look at traces: which step got slower? The model, search, a tool, the database?
4. Did the *inputs* change? Longer documents or longer conversations mean more tokens and more time.
5. Did traffic rise, so requests are queuing or being rate-limited and retried?
6. Did a model name that floats to "latest" quietly move to a new version?

The point is a calm, ordered search that narrows the cause, not a guess.

**Q: "Costs tripled this month."**
- Break the bill down by feature, by model and by customer.
- More requests, or more tokens per request?
- Common culprits: conversation histories growing without limit; a retry loop; a new
  feature sending whole documents; caching that stopped working because something changed
  at the top of the prompt; one customer or a bug hammering the service.
- Then add the missing limit or alert so it is caught sooner next time.

**Q: "The provider updated the model and quality dropped. What should have caught it?"**
Pinning to a fixed model version so updates happen only when you choose; an evaluation
suite run before switching; and daily quality checks on a sample of live traffic (level 10).

---

## 6. AI system design

These are 45-minute open questions such as "Design a support assistant for an airline."
Use the framework from level 9 and say what you are doing as you go.

**A time plan:**

| Minutes | Do |
|---|---|
| 0–5 | Ask questions. Who are the users? How many requests? How fast? What data? What counts as success? |
| 5–10 | State the requirements and the limits back, with numbers. |
| 10–25 | Draw the simplest design that meets them. Explain each part. |
| 25–35 | Go deeper on the hardest part. Say how you would measure quality. |
| 35–45 | Failures, cost, security, scaling, and what you would build next. |

**Worked opening for "an airline support assistant":**

> "Before I draw anything: is this for customers or for staff? Should it take actions, like
> rebooking, or only answer questions? Roughly how many conversations a day? Which
> languages? And what happens today when a customer asks something?
>
> Assuming customers, answering questions plus looking up their booking, 50,000
> conversations a day: I'd start simple. A router sorts each message into policy
> questions, booking questions, or 'needs a human'. Policy questions go to search over the
> help articles. Booking questions call one read-only tool for the signed-in customer's
> booking. I would not let it change bookings in the first version, because a mistake there
> is costly and hard to undo; I'd add that later behind a confirmation step.
>
> To measure it: a test set of a few hundred real questions, graded for correctness and for
> whether the answer is supported by the source, plus the hand-over rate in production…"

**Common mistakes:**

- Starting to draw before asking anything.
- Jumping straight to many agents and a vector database when a simpler design would do.
- Never mentioning how quality will be measured.
- Never mentioning cost.
- Ignoring what happens when something fails.

Practise four under a timer: a support assistant, a code-review tool, data extraction from
documents at large volume, and a voice agent.

---

## 7. Staying current

The field moves quickly, and interviewers often ask "how do you keep up?" or "what do you
think of [new thing]?"

**A sensible answer to "how do you keep up?":**

> "I follow a small number of primary sources: the documentation and engineering blogs of
> the main model providers, and a few practitioners who publish measurements, not just
> opinions. When something new looks relevant, I try it on my own test set before forming
> a view."

**A framework for any new technique.** When asked about something you have only half
heard of, ask these aloud:

1. What problem is it meant to solve?
2. What did people do before, and why was that not enough?
3. What does it cost (money, speed, complexity)?
4. How would I test whether it helps on my task?

*Example:* "I haven't used that in production. As I understand it, it aims to solve X. I'd
want to see whether it beats my current approach on my test set before adopting it, since
it adds another moving part."

That is a far stronger answer than pretending to expertise, and stronger than "I don't
know" on its own. Being hard to hype is a quality employers want.

---

## 8. Defending your projects

Expect: "Tell me about a project", followed by 20 minutes of "why?".

**Prepare these for each project:**

- **The problem** and why it mattered, in two sentences.
- **The design**, which you can draw from memory in two minutes.
- **The hardest bug** and how you found it.
- **A decision you would change** now.
- **The numbers:** quality, speed, cost.

**Sample exchange:**

> **Q: Why did you split documents by heading instead of fixed size?**
> A: I tried both. Fixed size cut rules in half, so a passage would say "it increases to 20
> days" with no mention of what "it" was. Splitting by heading kept each rule whole. On my
> 120 test questions, recall went from 74% to 83% from that change alone.
>
> **Q: What would you do differently?**
> A: I built the search first and the test set second. I spent two weeks tuning by feel
> before I had any numbers. Next time the test set comes first.
>
> **Q: What's the biggest weakness?**
> A: Questions that need two different sections at once. It finds one and misses the
> other. I'd try splitting such questions into parts and searching for each.

Notice: every answer has a reason and a number, and the weaknesses are stated plainly.

**If you do not know something about your own project,** say so. "I used the library's
default there and didn't look into it; I should have." An honest gap is fine. A bluff that
gets caught is not.

---

## 9. Behavioural and communication questions

These ask about how you work with people: "Tell me about a time when…"

**Structure every story in three parts:**

- **Situation:** what was going on (two sentences).
- **Action:** what *you* did (most of the answer). Say "I", not "we".
- **Result:** what happened, with a number if possible, and what you learned.

**Example: "Tell me about a disagreement with a colleague."**

> *Situation:* A teammate wanted to switch our support assistant to the newest large
> model because it "felt better". I thought the cost increase wasn't justified.
>
> *Action:* Rather than argue, I suggested we test both on our 200 saved questions. I ran
> it that afternoon and shared the table.
>
> *Result:* The new model scored 2 points higher at six times the cost. We agreed to use
> it only for the hardest 15% of questions. Quality rose and the bill barely moved. What I
> took from it: put a measurement on the table and the disagreement usually settles itself.

**Prepare stories for:**

- A disagreement.
- A failure or mistake of your own.
- An unclear or changing requirement.
- Explaining something technical to a non-technical person.
- A time you had to learn something quickly.

**Explaining AI limits to non-technical people** comes up often in this job.

> **"Why can't we just make it never be wrong?"**
> "It works by predicting likely text, so it will sometimes produce something that sounds
> right and isn't. We can't remove that, but we can manage it: we give it our real
> documents to answer from, we have it show its sources, we measure how often it's wrong,
> and for anything risky a person checks. Right now it's wrong about 2% of the time on our
> test questions, and those cases go to a human."

Plain words, honest about the limit, clear about what is being done, with a number.

**Useful phrases:**

- "I don't know, but here is how I'd find out."
- "It depends on X. If X is high I'd do A; otherwise B."
- "I'd start with the simplest version and measure before adding more."

---

## Free resources

- Chip Huyen's blog: <https://huyenchip.com/blog/>
- Eugene Yan's writing: <https://eugeneyan.com/writing/>
- Applied LLMs: <https://applied-llms.org/>
- Tech Interview Handbook (free): <https://www.techinterviewhandbook.org/>
- System Design Primer: <https://github.com/donnemartin/system-design-primer>
- Pramp, free practice interviews with peers: <https://www.pramp.com/>
- Anthropic engineering blog: <https://www.anthropic.com/engineering>

## Project: a full practice interview loop

Run four sessions with a friend, a mentor, or an AI voice assistant playing interviewer.
Record each one.

1. **Project deep dive (45 min).** Present one project and take hard questions.
2. **System design (45 min).** A question you have not seen before.
3. **Debugging and scenarios (45 min).** Six quick scenarios from sections 2 to 5.
4. **Coding (60 min).** Build something small and real: a retry helper with backoff, a
   keyword-search scorer, a tool-calling loop, or a streaming endpoint.

After each, listen to the recording and write down three things to fix. Repeat your weakest
session a week later.

## Self-check

1. Can you draw each of your projects from memory and justify every part?
2. Do you start with the simplest design and add complexity only when you can say why?
3. Does every answer about quality say how you would measure it?
4. Can you produce a cost estimate, with the sums, on the spot?
5. Do you have a prepared, honest answer for "what went wrong on this project"?
