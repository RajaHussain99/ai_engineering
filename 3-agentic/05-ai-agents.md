# Level 5: AI agents

**Phase 3: Agentic** · Previous: [Level 4](../2-retrieval/04-rag.md) · Next: [Level 6](06-context-engineering.md)

**Goal:** build systems where the model chooses the next step, uses tools, keeps track of
its work, recovers from errors and knows when to stop. Also learn when not to build one.

**Skip this level if** you have written an agent loop by hand and run it on tasks longer
than ten steps.

---

## 1. What an agent is

An agent is **a model using tools in a loop**. It decides what to do, does it, looks at
what happened, and decides again, until the job is finished.

**Example task:** "Find out why yesterday's sales report is empty and fix it."

```
Think:  I should look at the report job's log.
Act:    read_file("logs/report.log")
See:    "ERROR: table sales_2025 not found"
Think:  The table name may have changed. Let me list the tables.
Act:    run_sql("SHOW TABLES")
See:    sales_2026, customers, products
Think:  The job still points at last year's table. I'll update the config.
Act:    edit_file("config.yaml", "sales_2025" → "sales_2026")
Act:    run_command("python make_report.py")
See:    "Report written: 1,204 rows"
Done:   "The report job was reading last year's table. I updated the config and re-ran it."
```

Nobody wrote those steps in advance. The model chose each one based on what it had just
seen. That is what makes it an agent.

**Workflow vs agent** (from level 2):

| | Workflow | Agent |
|---|---|---|
| Who decides the next step | Your code | The model |
| Steps known in advance | Yes | No |
| Cost and speed | Lower, predictable | Higher, varies |
| Debugging | Easier | Harder |
| Best for | Repeatable tasks | Open-ended tasks |

**When not to build an agent.** If you can draw the steps as a flowchart, build the
flowchart. "Classify the email, then draft a reply from the matching template" is a
workflow. Use an agent when the path depends on what is discovered along the way: debugging,
research, working through a messy set of files.

---

## 2. The loop, planning and reflection

**The basic loop** is called ReAct (reason + act). It is level 2's tool calling, repeated.
Write it yourself once; it is short.

```python
def run_agent(task: str, tools: list, tool_functions: dict, max_steps: int = 20) -> str:
    messages = [{"role": "user", "content": task}]

    for step in range(max_steps):
        response = client.messages.create(
            model="claude-sonnet-5-5", max_tokens=2000, tools=tools, messages=messages
        )
        messages.append({"role": "assistant", "content": response.content})

        if response.stop_reason != "tool_use":          # no tool requested → finished
            return response.content[0].text

        results = []
        for block in response.content:
            if block.type == "tool_use":
                try:
                    output = tool_functions[block.name](**block.input)
                except Exception as error:
                    output = f"Error: {error}"           # let the model see the failure
                results.append({"type": "tool_result", "tool_use_id": block.id,
                                "content": str(output)})
        messages.append({"role": "user", "content": results})

    return "Stopped: reached the step limit."
```

That is a working agent. Frameworks add conveniences on top of exactly this.

**Planning.** Two styles:

- *Step by step:* decide one action at a time. Flexible, good when you learn as you go.
- *Plan first:* write a numbered plan, then carry it out, revising when something
  unexpected happens. Better for long tasks, because the plan keeps the agent on track.

**Reflection.** After a failure or before finishing, the agent checks its own work: "Did
the tests pass? Does this answer the original question?" Cheap, and catches many mistakes.

**Stopping.** An agent with no brakes can run forever and spend a lot. Always have:

- A maximum number of steps (as above).
- A maximum spend.
- A check for repeats: if it has made the same call with the same inputs three times, stop
  or tell it to try something else.

---

## 3. Tools for agents

Everything from level 2, section 5 applies, and matters more, because an agent makes many
calls in a row and each result feeds the next decision.

- **Keep results short.** A tool that dumps a 10,000-line file fills the agent's context
  and pushes the task out of view. Return the first 100 lines and say "showing 100 of
  10,000; call again with `start=100` for more".
- **Make errors helpful.** `"File not found: report.txt. Files in this folder: report.csv,
  notes.md"` lets the agent recover in one step.
- **Fewer, better tools.** An agent with 60 tools picks worse than one with 8.
- **Mark the dangerous ones.** Divide tools into two groups:

| Safe to run freely | Ask a human first |
|---|---|
| Read a file | Delete a file |
| Search the web | Send an email |
| Query a database | Change a database |
| List calendar events | Make a payment |

---

## 4. Model Context Protocol (MCP)

**The problem.** Every AI app used to need its own custom connection to every service. Ten
apps and ten services meant a hundred connections to write.

**The solution.** MCP is a shared standard, often compared to USB. Build a connector for a
service once (an "MCP server"), and any app that speaks MCP can use it.

The three roles:

- **Host:** the AI app the person uses (a chat app, a coding assistant).
- **Client:** the part inside the host that talks to servers.
- **Server:** a small program that offers a service's abilities.

A server can offer three kinds of thing:

- **Tools:** actions the model can take ("create a calendar event").
- **Resources:** data the app can read ("the contents of this file").
- **Prompts:** ready-made instructions the user can pick.

**A minimal server:**

```python
# pip install mcp
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("notes")
notes: list[str] = []

@mcp.tool()
def add_note(text: str) -> str:
    """Save a short note for later."""
    notes.append(text)
    return f"Saved. You now have {len(notes)} notes."

@mcp.tool()
def list_notes() -> list[str]:
    """Return all saved notes."""
    return notes

if __name__ == "__main__":
    mcp.run()
```

Any MCP-capable app can now use these two tools. The function's description (the text in
triple quotes) is what the model reads to decide when to call it.

Treat MCP servers written by strangers like any software from strangers: they run with
whatever access you give them (more in level 8).

---

## 5. Memory

"Memory" for an agent comes in two kinds.

**Short-term memory** is the context window: the conversation and the tool results so far.
It is lost when the session ends, and it fills up on long tasks (level 6 deals with that).

**Long-term memory** is anything saved outside the window and brought back later.

*Example:* on Monday a user says, "I'm vegetarian and allergic to nuts." The agent saves a
note. On Friday, in a brand-new conversation, the user asks for dinner ideas. The agent
loads its notes first and suggests nut-free vegetarian meals.

The simplest form is a file the agent can read and write:

```
# memory/user_preferences.md
- Vegetarian
- Allergic to nuts
- Prefers recipes under 30 minutes
```

Larger systems store memories in a database and search them, the same way as RAG.

The hard question is what to keep. Save lasting facts and preferences. Do not save
passing details ("the user asked about pasta today"), or the memory fills with noise and
old notes start to mislead.

---

## 6. Multiple agents

Sometimes one agent hands parts of the job to others.

**Common shapes:**

- **Lead and helpers.** A lead agent splits the task, sends pieces to helper agents, and
  combines what comes back.
  *Example:* "Compare five laptop brands." The lead starts five helpers, one per brand,
  all working at once. Each returns a half-page summary.
- **Handoff.** A front-desk agent passes the conversation to a specialist.
  *Example:* a support agent sends billing questions to a billing agent that has billing
  tools and instructions.

**What you gain:**

- *Speed:* helpers work at the same time.
- *Clean context:* each helper reads 50 pages in its own window and hands back a short
  summary. The lead's window stays uncluttered.

**What it costs:**

- Several times the tokens.
- Helpers only know what the lead tells them. Vague instructions give useless results.
- More moving parts to debug.

Advice: start with one agent and good tools. Add helpers when the work clearly splits into
independent pieces or one context window is not enough.

---

## 7. Graph-based frameworks

LangGraph, a popular framework, describes an agent as a graph: boxes (steps) joined by
arrows (what can follow what), sharing a common record of state.

```
        ┌─────────┐      needs a tool      ┌───────────┐
start → │  model  │ ─────────────────────► │ run tools │
        └─────────┘ ◄───────────────────── └───────────┘
             │ no tool needed
             ▼
           finish
```

What it adds over the hand-written loop:

- **Checkpoints:** state is saved after each step, so a crashed run can resume.
- **Pauses for people:** the graph can stop, wait for a human to approve, then continue.
- **Explicit control:** you can force certain steps to happen in a set order while leaving
  others to the model.

Learn it after writing the loop by hand; then every part will map to something you
already understand.

---

## 8. Sandboxed execution

An agent that writes and runs code is very capable: it can analyse data, draw charts and
test its own work. It is also risky, because code can delete files or send data anywhere.

A **sandbox** is a sealed box to run that code in: usually a container (level 0) with

- no access to your real files,
- no passwords or keys inside,
- limited or no internet,
- limits on time and memory,
- and it is thrown away afterwards.

*Example:* a data-analysis agent gets a copy of a spreadsheet inside a sandbox. Its code
can do anything to that copy. If the code is bad, or the agent was tricked into something
harmful, the damage stays in the box.

Never run model-written code directly on your own machine or server with your own permissions.

---

## 9. Reliability, user experience and recovery

**Errors add up.** This is the central reliability fact about agents.

```
If each step works 95% of the time:
   5 steps  → 77% chance the whole task works
  20 steps  → 36%
  50 steps  →  8%
```

So long tasks need agents that notice mistakes and repair them, not agents that never err.
What helps:

- **Checks along the way.** After editing code, run the tests. After writing a file, read
  it back.
- **Checkpoints.** Save progress so a failure at step 40 resumes from step 39, not step 1.
- **Clear error messages from tools** (section 3).

**User experience:**

- Show what the agent is doing ("Searching orders… Reading 3 results…"). Silence for two
  minutes feels broken.
- Let the user stop or redirect it mid-task.
- Ask before doing anything that cannot be undone.
- Finish with a short summary of what was done and anything that did not work.

---

## 10. Other frameworks

There are many: the Claude Agent SDK, OpenAI Agents SDK, Pydantic AI, CrewAI, Google's
Agent Development Kit, and more. Their interfaces change often.

What stays the same is everything in this file: the loop, tools, state, stopping
conditions, approval steps. Learn those well and picking up any framework takes a day.

---

## Free resources

- "Building effective agents": <https://www.anthropic.com/engineering/building-effective-agents>
- ReAct paper: <https://arxiv.org/abs/2210.03629>
- Model Context Protocol docs: <https://modelcontextprotocol.io/>
- LangGraph docs: <https://langchain-ai.github.io/langgraph/>
- Hugging Face agents course: <https://huggingface.co/learn/agents-course>
- Pydantic AI: <https://ai.pydantic.dev/>
- Anthropic tool use docs: <https://docs.anthropic.com/en/docs/build-with-claude/tool-use/overview>

## Project: a production-style agent

Pick one: a research agent that searches, reads and writes a report with sources; an agent
built around an MCP server you write; or a data-analysis agent that writes and runs code in
a sandbox.

Requirements for all three:

- At least three tools, one of which changes something and needs human approval.
- A step limit and a spending limit.
- Recovery from a failing tool without the whole run crashing.
- A visible log of every step.
- First version with no framework; then rebuild it with one and note what changed.
- **Evaluation step:** 15 tasks with a clear definition of success. Report the success
  rate, average number of steps and average cost per task.

## Self-check

1. Give a task where an agent is the wrong choice, and say what you would build instead.
2. How does your agent know it is finished?
3. What does MCP standardise that plain tool calling does not?
4. When do several agents help, and what do they cost?
5. Your agent succeeds 90% of the time per step. What happens over 30 steps, and what do you do about it?
