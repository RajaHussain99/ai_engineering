# Level 7: LLM evaluations

**Phase 4: Evaluation & trust** · Previous: [Level 6](../3-agentic/06-context-engineering.md) · Next: [Level 8](08-ai-security.md)

**Goal:** measure whether an AI system is good, so that every change is a decision backed
by numbers.

**Skip this level if** your team blocks releases on an automated evaluation suite.

An evaluation ("eval") is a test for an AI system. You have been running small ones since
level 2. This level makes them systematic. It is the skill that most separates people who
ship reliable AI from people who ship demos.

---

## 1. Why evaluate

With ordinary code, `add(2, 2)` returns 4 every time and one test proves it. With a model,
the same input can give different outputs, there are many acceptable answers, and a prompt
change that fixes one case can quietly break five others.

**Without evals:**

> "I changed the prompt and tried three questions. Looks better. Ship it."
> A week later, customers report that refund questions now get wrong answers.

**With evals:**

> "I changed the prompt. Score on our 200 test cases went from 81% to 86%. Refund
> questions dropped from 95% to 70%, so I fixed that before shipping."

**Where to start: read your outputs.** This is the step people skip, and the most valuable.

1. Collect 50 to 100 real inputs and the system's outputs.
2. Read each one. Write a short note on anything wrong.
3. Group the notes into categories.

*Example result for a support bot:*

| Problem | Count |
|---|---|
| Answer too long | 14 |
| Invented a policy that does not exist | 9 |
| Did not answer the actual question | 6 |
| Wrong tone (too casual) | 3 |
| Fine | 68 |

Now you know what to measure and what to fix first. Starting from generic metrics, without
looking at your own data, measures things that may not matter for your product.

---

## 2. Methods

Three ways to grade an output. Use the cheapest one that works.

### Code checks

Plain code that returns pass or fail. Fast, free and consistent.

```python
def check_category(output: str) -> bool:
    return output.strip() in {"billing", "technical", "other"}

def check_length(output: str) -> bool:
    return len(output.split()) <= 80

def check_has_citation(output: str) -> bool:
    return "[" in output and "]" in output
```

Good for: exact answers, valid format, length limits, required or banned words, whether
the right tool was called.

### Model graders

For things code cannot judge ("is this answer polite?", "is it supported by the source?"),
ask a second model to grade. This is often called "LLM as judge".

```
You are grading a customer-support answer.

<question>{question}</question>
<policy_document>{policy}</policy_document>
<answer>{answer}</answer>

Does the answer state anything about company policy that is NOT supported by the policy
document? First explain your reasoning in one or two sentences. Then, on a new line,
write exactly PASS (nothing unsupported) or FAIL (something unsupported).
```

Tips:

- **Ask for pass or fail, not a score from 1 to 10.** Nobody, human or model, can
  consistently tell a 6 from a 7. A yes/no question with a clear standard gives steadier results.
- **One question per grader.** Separate graders for "is it accurate", "is it polite" and
  "is it short enough" beat one that judges everything.
- **Ask for the reasoning before the verdict.** It improves the verdict and lets you see why.

### Human review

People reading outputs. Slow and costly, but it is the truth the other methods are trying
to copy. Use it to set up and check the graders.

### Checking the grader

A model grader can itself be wrong, so test it:

1. Label 50 outputs yourself, pass or fail.
2. Run the grader on the same 50.
3. Count how often it agrees with you.

*Example:* it agrees on 46 of 50, which is 92%. Good enough to rely on. If it agreed on 33
of 50, you would rewrite the grader prompt before trusting any number it produces.

---

## 3. Offline and online

**Offline evaluation** happens before release, on a fixed set of test cases. It is like
testing a car on a track.

**Online evaluation** happens after release, on real use. It is like watching how the car
does on real roads.

Online signals:

- Thumbs up and down.
- Whether the user rephrased and asked again (a sign the first answer failed).
- Whether the user copied the answer or accepted the suggestion.
- Whether the conversation was passed to a human.
- A sample of real conversations run through your graders each day.
- **A/B tests:** half of users get version A, half get version B; compare.

**The loop that makes systems improve:**

```
Real failure found in production
        ↓
Add it to the offline test set
        ↓
Fix it, and confirm with the test set
        ↓
It can never silently come back
```

---

## 4. Model evaluation and benchmarks

Two different questions get confused:

- **"Which model is best in general?"** That is what public benchmarks try to answer.
- **"Does my product work?"** That is what your own evals answer.

Benchmarks are standard exams for models: sets of coding problems, maths questions,
general-knowledge questions, and so on. They are useful for a rough comparison. Their limits:

- **Saturation.** Once top models all score 95% or more, the exam no longer separates them.
- **Leakage.** Exam questions are public, so they end up in training data. The model may
  have seen the answers.
- **Mismatch.** A high score on a general exam says little about sorting *your* insurance claims.

Use benchmarks to pick two or three candidates. Use your own test set to choose between them.

---

## 5. How benchmarks have changed

Worth knowing as background, because it shows why no fixed test lasts.

- Early tests were multiple-choice knowledge and reading questions. Models passed human
  level within a few years.
- Harder ones followed: graduate-level science questions, competition maths.
- Then tests of *doing* instead of knowing: fixing real bugs in real code projects,
  completing tasks on a computer, finishing multi-hour assignments.

The pattern repeats: a test is released, models improve, the test is "solved", a harder one
replaces it. The lesson for you is to keep adding harder cases to your own test set as your
system improves. A test your system always passes tells you nothing new.

---

## 6. Building your own test set

A test set is a list of cases. Each case has an input and a way to judge the output.

```json
{"input": "I was charged twice this month", "expected_category": "billing"}
{"input": "app wont open after update!!", "expected_category": "technical"}
{"input": "", "expected_category": "other"}
```

**Where to get cases:**

- **Real usage.** The best source. Take real questions from logs (with personal details removed).
- **Known failures.** Every bug report becomes a case.
- **Edge cases you invent.** Empty input, very long input, another language, rude input,
  two questions at once, a question outside the system's job.
- **Generated variations.** Ask a model to write 20 different ways a customer might ask
  about a refund. Read them yourself before using them.

**How many?** Start with 30 to 50 and grow to a few hundred. With only 20 cases, one case
changing is a 5-point swing, which is too jumpy to trust.

**Balance.** If 40% of real questions are about billing, about 40% of the test set should
be. And make sure each problem category from section 1 has cases that would catch it.

---

## 7. Test suites and catching regressions

A **regression** is when something that used to work stops working. They are the main risk
when changing prompts.

Run the whole test set automatically whenever the prompt, the model or the code changes.
This uses the automatic test runs you set up in level 0.

*What a result looks like on a proposed change:*

```
Evaluation results (200 cases)
                    before    after
Overall              84%       88%    ▲
  billing            95%       70%    ▼  ← investigate
  technical          80%       93%    ▲
  other              78%       91%    ▲
Avg. response time   2.1s      2.3s
Avg. cost            $0.004    $0.004
```

The overall score rose, but billing fell badly. Without the breakdown by category you
would have shipped a regression.

Practical points:

- Keep a fast set (a few dozen cases, under a minute) for every change and a full set
  before release.
- Track scores over time on a chart.
- Because outputs vary from run to run, a 1-point move may be noise. Run twice if unsure.

---

## 8. Evaluating RAG

A RAG system (level 4) has two parts, and you measure each separately so you know which to fix.

**Part 1: did search find the right material?**

- **Recall@k:** of the questions, how many had the right chunk in the top k results?
- **Precision:** of the chunks returned, how many were actually relevant?

**Part 2: did the model answer well from it?**

- **Faithfulness:** is every claim in the answer supported by the retrieved chunks? (This
  catches invented facts.)
- **Relevance:** does the answer address the question that was asked?
- **Correctness:** does it match the known right answer?

**Diagnosing a wrong answer:**

| Right chunk retrieved? | Answer faithful to chunks? | The problem is |
|---|---|---|
| No | n/a | Search. Fix chunking, add hybrid search or reranking. |
| Yes | No | Generation. The model ignored or contradicted the source; fix the prompt. |
| Yes | Yes, but still wrong | The source document itself is wrong or outdated. |

Also test questions with **no answer in the documents**, and check the system says so.

---

## 9. Evaluating agents

Agents are harder to test because they take many steps and there are many valid routes to
the goal.

**Check the outcome, not the transcript.** An agent may *say* "I've booked the flight".
Look at the booking system: is there a booking? Judge the final state of the world.

*Example case:*

```
Task:     "Fix the failing test in the payments module."
Setup:    A code repository with one known failing test.
Success:  All tests pass AND no test files were deleted or edited to cheat.
```

That last condition matters. Agents sometimes "succeed" by removing the failing test.
Write success checks that close such loopholes.

**Also measure how it got there:**

- Did it choose sensible tools?
- Were the inputs to those tools correct?
- How many steps and how much money did it take?
- Did it do anything it should not have (touch unrelated files, skip an approval)?

**Run each task several times.** Agents vary. A task that passes 3 times in 5 is a 60%
task, not a passing one. Report rates, not single runs.

**Give partial credit where it makes sense.** An agent that completed four of five subtasks
is better than one that completed none, and your score should show it.

---

## Free resources

- Hamel Husain, "Your AI product needs evals": <https://hamel.dev/blog/posts/evals/>
- Eugene Yan on LLM patterns and evaluation: <https://eugeneyan.com/writing/llm-patterns/>
- Applied LLMs: <https://applied-llms.org/>
- Inspect, open-source evaluation framework: <https://inspect.aisi.org.uk/>
- promptfoo: <https://www.promptfoo.dev/>
- Ragas, RAG metrics: <https://docs.ragas.io/>
- Langfuse docs: <https://langfuse.com/docs>

## Project: an evaluation suite for your own systems

The original course outline has no project for this level. This one matters more than most.

- Take the level 4 RAG application and the level 5 agent.
- Read 60 outputs from each and write your list of problem categories.
- Build a test set of at least 100 cases per system.
- Write code checks plus two model graders. Label 40 cases by hand and report how often
  each grader agrees with you.
- Run the suite automatically so a proposed change shows the score difference.
- Make one improvement to each system and show before and after.

## Self-check

1. Why read raw outputs before choosing what to measure?
2. Why is pass/fail usually better than a 1-to-10 score for a model grader?
3. How do you know your grader can be trusted?
4. A RAG answer is wrong. Which two measurements tell you where to look?
5. Why run an agent task more than once?
