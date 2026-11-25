# Level 6: Context engineering

**Phase 3: Agentic** · Previous: [Level 5](05-ai-agents.md) · Next: [Level 7](../4-evaluation-and-trust/07-llm-evaluations.md)

**Goal:** manage what the model sees. Decide what goes into the context window, what stays,
what gets summarised and what gets stored elsewhere, so an agent keeps working well through
a long task.

**Skip this level if** you have kept an agent on track across several hundred tool calls.

## How this differs from prompt engineering

Prompt engineering is about wording one request well. Context engineering is about
everything the model sees over a whole job: instructions, tool descriptions, documents,
conversation history and tool results, and how that set changes step by step.

Think of the context window as a desk. A prompt is one sheet of paper on it. Context
engineering is keeping the desk tidy for an eight-hour working day, so the right papers
are in front of you and the rest are filed where you can find them.

---

## 1. The context window as a limited resource

Two limits apply.

**The hard limit.** The window has a maximum size. Go over and the request fails.

**The soft limit, which matters more.** Long before the window is full, quality drops. The
model's attention is spread across everything in view, so each irrelevant item makes the
relevant ones a little easier to miss. More context is not always better.

**Four ways a context goes bad:**

| Problem | Example |
|---|---|
| A mistake sticks | Early on the agent wrongly concludes "the bug is in the login code". That note stays in history and every later step builds on it. |
| The task gets buried | After 80 tool results, the original instruction "do not change the public API" is 100,000 tokens back, and the agent changes the public API. |
| Noise misleads | A search returns ten pages, nine irrelevant. The agent starts chasing something from page 6. |
| Contradictions | An old document says the price is $10; a new one says $12. Both are in view and the agent picks the wrong one. |

Most of the techniques below are ways to prevent these four.

---

## 2. Choosing and arranging what goes in

**Order.** A dependable layout:

```
1. Instructions that never change      (role, rules)
2. Tool descriptions
3. Reference material                   (documents, examples)
4. Conversation history and tool results
5. The current request
```

Stable things at the top, changing things at the bottom. This also makes caching work
(section 7).

**Fetch just in time.** Do not load everything at the start in case it is needed.

*Loaded up front:* paste all 40 files of a codebase into the prompt. Most are irrelevant
to today's bug, they cost money on every step, and they bury the three that matter.

*Just in time:* give the agent a list of file names and a `read_file` tool. It opens the
three it needs.

Give the model a map and the means to look things up, not the whole territory. This is how
a person works too: you do not memorise the filing cabinet; you know where things are.

---

## 3. Compression and compaction

On a long task the history keeps growing. Eventually you must shrink it.

**Cheapest first: clear old tool results.** A file the agent read 50 steps ago, and has
since edited, is stale. Replace its contents in the history with a short placeholder like
`[file contents removed; read again if needed]`. This frees a great deal of space and
loses almost nothing, since the agent can re-read the file.

**Then: summarise (compaction).** Replace a long stretch of history with a short summary
and continue from there.

*Before:* 150,000 tokens of back-and-forth.

*After:*

```
## Summary of work so far
Task: add CSV export to the reports page. Do not change the public API.

Done:
- Added export_csv() in reports/export.py
- Added an "Export" button in ReportsPage.tsx

Decisions:
- Using the built-in csv module, because the user asked for no new dependencies

Still to do:
- Dates are written in US format; the user wants day-month-year
- test_export_large_file is failing with a timeout

Files touched: reports/export.py, ReportsPage.tsx, tests/test_export.py
```

**What a summary must keep:** the goal, the rules and constraints, decisions and the
reasons for them, what is unfinished, known problems, and the names of files or items being
worked on.

**What it can drop:** raw tool output, dead ends, repeated attempts.

**The classic bug:** the summary drops the one rule that mattered ("do not change the
public API"), and the agent breaks it ten steps later. Test your summaries: after
compacting, can the agent still state the original constraints?

---

## 4. Memory and storing state outside the window

Instead of keeping everything in the window, have the agent write things down and read
them back. Files survive compaction, crashes and new sessions.

**A to-do list:**

```
# todo.md
- [x] Read the existing export code
- [x] Add export_csv function
- [ ] Fix date format
- [ ] Fix failing test
- [ ] Update the documentation
```

The agent updates this as it works and re-reads it after every compaction. It cannot lose
its place.

**A notes file:**

```
# notes.md
- Tests are run with `make test`, not pytest directly
- The reports page uses the old date library; do not upgrade it (user's request)
- Large files time out above about 50,000 rows
```

These are things learned the hard way that would otherwise be rediscovered, at a cost,
after each compaction.

This is exactly what people do on long projects: we keep lists and notes because we cannot
hold it all in our heads.

---

## 5. Strategies for long contexts

**A big window does not remove the need to choose.** Even with room for a million tokens,
recall of specific details falls as the input grows, cost rises with every step, and
responses slow down.

When you do use a long context:

- Put the long material at the top and the question at the end.
- Ask the model to first pull out the relevant quotes, then answer from those quotes. This
  focuses it on the parts that matter.

**Helpers as a way to keep context clean.** From level 5, section 6. A helper agent gets
its own fresh window, does a messy job (reading 30 web pages, searching a large codebase),
and returns a 300-word summary. The lead agent's window takes in 300 words instead of
30 pages.

*Example:* "Where in this codebase is the tax rate set?" A helper opens 25 files and
replies: "It is set in `billing/config.py` line 14 and overridden for the EU in
`billing/regions.py` line 80." The lead never sees the other 23 files.

---

## 6. Managing tool context

Tools use context in two places.

**Their descriptions.** Every tool's description is sent on every request. Fifty tools with
long descriptions can cost tens of thousands of tokens before any work happens, and make
the model worse at picking the right one.

- Give an agent only the tools its job needs.
- For large sets, let the agent search for tools and load the ones it wants, instead of
  listing all of them up front.

**Their results.** This is where windows usually fill.

| Wasteful | Careful |
|---|---|
| Return the whole 5,000-row table | Return 20 rows and the total count; let the agent ask for more or filter |
| Return a whole web page with menus and adverts | Return only the main article text |
| Return every field of a record | Return the five fields the agent asked for |

Trim at the tool, before the result ever reaches the model. It is far cheaper than
cleaning up afterwards.

---

## 7. Caching and efficiency

On a long agent run, each step resends the entire history. At step 50 you are paying to
process the same early content for the fiftieth time. **Prompt caching** lets the provider
reuse its work on the unchanged beginning and charge a fraction of the normal price for it.

```python
response = client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=1000,
    system=[{
        "type": "text",
        "text": long_instructions_and_reference_material,
        "cache_control": {"type": "ephemeral"},     # "remember everything up to here"
    }],
    messages=messages,
)
print(response.usage)   # shows how many tokens were read from the cache
```

**The rule:** a cache works only if the beginning is exactly the same, character for
character, as last time. Anything that changes breaks it from that point on.

Things that silently break caching:

- The current time written at the top of the prompt.
- Reordering tools between requests.
- Editing or deleting an earlier message.

So: keep the top fixed, put changing things at the end, and **add to the history, never
rewrite it** (except when you deliberately compact).

Watch the cache figures in your logs. On a well-built agent, most input tokens should be
cache reads. If they are not, something near the top is changing.

---

## 8. Harness engineering

The **harness** is all the code around the model: the loop, the tools, the permission
checks, the compaction, the memory files, the recovery from errors.

A useful comparison: the model is an engine; the harness is the rest of the car. A great
engine in a car with poor brakes and no steering is not a good car.

What a harness is responsible for:

- Running the loop and calling tools.
- Deciding what goes into the context at each step.
- Compacting when the window is filling.
- Asking the user for approval when needed.
- Saving checkpoints and resuming.
- Stopping on limits of steps, time and money.
- Recording everything for debugging.

Two agents built on the same model can perform very differently because of harness
choices. Much of the practical skill in building agents lies here, not in the choice of model.

---

## Free resources

- "Effective context engineering for AI agents": <https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents>
- Prompt caching docs: <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>
- "Lost in the Middle" paper: <https://arxiv.org/abs/2307.03172>
- "What we learned from a year of building with LLMs": <https://applied-llms.org/>
- LangGraph memory concepts: <https://langchain-ai.github.io/langgraph/concepts/memory/>

## Project: a long-running agent with managed context

Extend the level 5 agent, or write a new one, to finish a task that needs at least 100 tool
calls. Examples: tidy up a medium-sized code repository, or research and write a 20-page report.

- Automatic compaction when the window passes a set size.
- A notes file and a to-do list that the agent keeps and re-reads.
- Tool results capped and split into pages.
- Prompt caching turned on, with the share of cached tokens logged.
- Resume from a checkpoint after you stop the program halfway.
- **Evaluation step:** run the same task with and without each technique. Report whether
  it finished, total tokens and cost for each setup.

## Self-check

1. Why can adding more context make answers worse?
2. What must a compaction summary always keep?
3. What breaks a prompt cache, and how do you design around it?
4. How do helper agents help with context, apart from speed?
5. Name three jobs of an agent harness.
