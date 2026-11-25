# Level 14: Portfolio projects

**Phase 6: Frontier & proof** · Previous: [Level 13](13-fine-tuning.md) · Next: [Level 15](../7-interview/15-interview-prep.md)

**Goal:** turn what you have learned into three to five public, working projects that
combine earlier levels. For someone new to the field, this is the strongest evidence an
employer will see.

A **portfolio** is a collection of your work that others can look at. If you built the
project in each earlier level, you already have most of the material. Use this level to
combine, strengthen and present it.

---

## What makes a project believable

Someone reviewing applications may give yours two minutes. Five things make those minutes count.

**1. It is live.** A link they can click beats code they would have to download and set up.

**2. It has numbers.** Compare:

> "Built a chatbot that answers questions about documents."

> "Built a document Q&A system. Search finds the right passage in the top 5 results for
> 91% of 120 test questions (up from 74% with basic search). Answers are correct and
> supported by the source 88% of the time. Typical response time 2.3 seconds, cost 0.4
> cents per question."

The second shows you measure your work, which is rarer than building.

**3. It shows decisions.** "I tried three ways of splitting documents. Splitting by heading
scored 9 points higher than fixed-size pieces, so I used that." Anyone can follow a
tutorial; choices show understanding.

**4. It solves a real problem.** Ideally one you or someone you know has. "A tool that
answers questions about my university's 300-page regulations" is more interesting than
"a PDF chatbot".

**5. It is not a copy of a tutorial.** Reviewers have seen the standard demo many times.
Change the data, the field or the constraints.

---

## Project ideas

Build three to five. For each: what it is, a concrete example, which levels it uses, and
what makes it stand out.

### 1. Document question-answering platform

**What:** upload documents, ask questions, get answers that point to their sources.

**Example:** a tool for tenants that answers questions about their local housing rules,
with a link to the exact section.

**Uses:** level 0 (API, deployment), 4 (search and answering), 7 (measurement), 10 (running it).

**Must have:** hybrid search with reranking; a search test set with recall figures; answers
with citations; "I don't know" when the documents lack the answer.

**Stands out if:** each user sees only their own documents (level 8), and you show a
before/after table for each search improvement.

### 2. CV and job-description matching agent

**What:** reads a CV and a job advert, pulls out structured information from both, and
produces a match score with reasons.

**Example output:**

```json
{
  "match": 72,
  "meets": ["3+ years Python", "REST API experience"],
  "missing": ["Kubernetes", "team leadership"],
  "summary": "Strong on backend skills; no evidence of the required infrastructure experience."
}
```

**Uses:** level 2 (structured output), 3 (prompts), 7 (measurement).

**Must have:** output that always matches a fixed structure; a test set where you compare
the tool's ranking with a person's.

**Stands out if:** you run a fairness check. Change only the name on otherwise identical
CVs and show the score does not move. If it does move, say so and show how you fixed it.
This is a real concern in hiring tools (level 8, section 8), and addressing it shows maturity.

### 3. Multi-agent research report system

**What:** given a question, a lead agent plans the research, helper agents search and read
at the same time, and a writer produces a report with sources.

**Example:** "What are the trade-offs between heat pumps and gas boilers for a 1930s house
in the UK?" → a three-page report with twelve linked sources.

**Uses:** level 5 (agents), 6 (managing context), 7 (measurement).

**Must have:** a visible plan; helpers running in parallel; every claim linked to a source;
limits on steps and spending.

**Stands out if:** you report the cost and time per report, and compare one agent against
several on the same questions to show whether the extra agents were worth it.

### 4. An MCP server for a real tool

**What:** a connector that lets any MCP-capable AI app use a real service (level 5, section 4).

**Example:** a server for a to-do app with `list_tasks`, `add_task` and `complete_task`.

**Uses:** level 5 (MCP, tool design), 8 (security).

**Must have:** clear tool descriptions; helpful error messages; small, tidy results; a
confirmation step for anything that changes or deletes.

**Stands out if:** you publish it so others can install it, and the README explains the
access it needs and why. Real users, even five of them, are strong evidence.

### 5. A monitored production API

**What:** a small AI service where the interesting part is how well it is run.

**Example:** a text-summarising API that sends short inputs to a small model and long ones
to a large one, falls back to a second provider on failure, and caps daily spending.

**Uses:** level 9 (design), 10 (operations).

**Must have:** routing between models; fallback; spending limits; a trace for every
request; a dashboard; automated tests and quality checks on each change.

**Stands out if:** you include a screenshot of the dashboard and a short write-up of a
failure you caused on purpose and how the system coped. This project tells the story "I can
keep things running", which many portfolios lack.

### 6. A voice or image project

**What:** something that goes beyond text.

**Examples:** a voice assistant for booking appointments; a tool that reads photos of
receipts into a spreadsheet; search across your photo library by description.

**Uses:** level 12.

**Must have:** accuracy measured on real, messy inputs (blurry photos, background noise);
response time measured for voice.

**Stands out if:** you show honest failure examples and what you did about them.

---

## Presenting each project

### The README

The README is the front page of your project. Most visitors read nothing else. A structure
that works:

```markdown
# Tenant Rules Assistant

Answers questions about city housing rules, with a link to the exact section.

**Try it:** https://…        **2-minute demo:** https://…

![screenshot](docs/screenshot.png)

## What it does
Three or four sentences. Who it is for and what problem it solves.

## How it works
A diagram, then one paragraph.

## Results
| Measure                        | Basic version | Final |
|--------------------------------|---------------|-------|
| Right passage in top 5         | 74%           | 91%   |
| Answer correct and supported   | 71%           | 88%   |
| Typical response time          | 4.1s          | 2.3s  |

Tested on 120 questions I wrote from real tenant forum posts.

## Decisions and trade-offs
- **Split documents by heading, not fixed size.** Scored 9 points higher, because …
- **Added keyword search.** Rule numbers like "§14.2" were being missed by …
- **Chose the mid-size model.** The large one scored 2 points higher at 6× the cost.

## Limitations
- Only covers rules as of March 2026.
- Struggles with questions that span several sections.
- Not legal advice; the app says so on every answer.

## Run it yourself
Four or five commands that work on a clean machine.
```

### Why "Limitations" helps you

It may feel like admitting weakness. It reads as the opposite. Every real system has
limits; naming yours shows you tested enough to find them and are honest about your work.
Experienced reviewers distrust projects that claim to have none.

### A demo video

Two minutes, screen recording, your voice. Show the main use, one hard case it handles,
and one number. Many people will watch a short video who would never run your code.

### A diagram

Boxes and arrows showing the parts and how data moves. A free tool such as Excalidraw is
enough. A reviewer should understand the design in ten seconds.

### A short article

For each project, write 500 to 1,000 words on the hardest problem you hit and how you
solved it.

*Example title:* "Why my document search kept missing rule numbers, and the fix that took
recall from 74% to 91%."

Structure: the problem, what you tried that failed, what worked, the numbers, what you
would do next. Publish it anywhere public and link it from the README. This shows you can
explain technical work, which is a large part of the job.

---

## Positioning

**Positioning** means presenting yourself clearly for the kind of job you want.

- **Pick a direction.** "I build search and question-answering systems and measure them
  carefully" is easier to remember than "I do AI."
- **Put the best project first.** On your profile and CV, lead with the strongest one.
- **One line per project, with a number.**
  *"Document Q&A system: 91% search accuracy on 120 test questions; live at …"*
- **Tidy your public profile.** Pin the three to five projects. Remove or hide half-finished
  experiments and tutorial copies.
- **Make it easy to reach you.**

---

## Keeping public demos safe and cheap

A public demo backed by a paid model is an open tap unless you limit it.

- **Set a spending cap** on the key with your provider.
- **Limit requests** per visitor, for example 20 a day.
- **Limit input size** so nobody pastes a whole book.
- **Use a small model** for the demo where quality allows.
- **Keep the key on the server.** Never put it in code that runs in the visitor's browser,
  where anyone can read it.
- **Add a short notice** saying it is a demo and inputs should not include private information.

## Free hosting and tools

- GitHub, for code and READMEs: <https://github.com/>
- Hugging Face Spaces, free demo hosting: <https://huggingface.co/spaces>
- Streamlit Community Cloud: <https://streamlit.io/cloud>
- Gradio, for quick interfaces: <https://www.gradio.app/>
- Excalidraw, for diagrams: <https://excalidraw.com/>

## Self-check

1. Can a stranger use each project within one minute of opening the README?
2. Does each project show a number that improved, and what it was compared with?
3. Can you explain every design decision for five minutes without notes?
4. What is the weakest part of each project, and have you said so in writing?
5. What stops someone from running up your bill through the public demo?
