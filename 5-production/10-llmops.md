# Level 10: LLMOps

**Phase 5: Production** · Previous: [Level 9](09-ai-system-design.md) · Next: [Level 11](11-managed-ai-platforms.md)

**Goal:** take the RAG application and the agent you already built and run them for real
users: versioned, tested on every change, guarded, packaged, deployed, monitored, and
improving over time.

**Skip this level if** you already run an AI service with automated releases, quality
gates and alerts.

"Ops" is short for operations: everything involved in keeping software running well after
it is written. LLMOps is operations for applications built on language models.

---

## 1. What LLMOps adds

Ordinary operations covers deploying code and keeping servers healthy. AI applications add
four complications.

| Ordinary software | AI application |
|---|---|
| Behaviour changes only when code changes | Behaviour also changes when the **prompt** or **model** changes |
| Same input, same output | Same input can give different outputs |
| A test passes or fails | Quality is a percentage across many cases |
| You control all the parts | The model is run by someone else and may be updated or retired |

*Example of the last row:* nothing in your code changed, yet answers got worse on Tuesday.
Cause: you were using a model name that always points to "the newest version", and the
provider moved it to a new release. Section 2 shows how to avoid this.

So you now manage three things that can each change your product's behaviour: **code,
prompts and model versions**. All three need versioning, testing and a way to go back.

---

## 2. Getting an app ready for production

A demo runs on your laptop. A production app must survive real conditions. A checklist:

**Settings outside the code.**

```python
import os
MODEL = os.environ.get("MODEL_NAME", "claude-sonnet-5-5")
MAX_TOKENS = int(os.environ.get("MAX_TOKENS", "1000"))
```

Now you can change the model on the server without editing code.

**Secrets in a secrets manager,** never in the repository (level 0, section 5).

**Pin exact model versions.** Providers offer names that float to the latest release and
names that stay fixed to one release. Use the fixed kind in production, so behaviour
changes only when *you* decide, after running your tests.

**Structured logs.** Write logs as data, so they can be searched and counted.

```python
import json, logging
logging.info(json.dumps({
    "event": "model_call", "request_id": request_id, "model": MODEL,
    "prompt_version": "support_v7", "input_tokens": 1830, "output_tokens": 212,
    "seconds": 2.4, "status": "ok",
}))
```

**Health check** (level 0, section 7), **timeouts and retries** (level 2, section 8).

**Graceful shutdown.** When the server is told to stop for a new release, it should finish
the requests in progress first, not cut users off mid-answer.

---

## 3. Tracking experiments and prompt versions

While improving a system you try many things. Without records you forget what you tried
and cannot reproduce your best result.

**An experiment log** records, for each attempt:

| Run | Prompt | Model | Other settings | Test set | Score | Cost per request |
|---|---|---|---|---|---|---|
| 12 | support_v6 | small | 5 chunks | v3 (200 cases) | 81% | 0.3¢ |
| 13 | support_v7 | small | 5 chunks | v3 | 85% | 0.3¢ |
| 14 | support_v7 | small | 10 chunks + reranker | v3 | 89% | 0.5¢ |
| 15 | support_v7 | large | 10 chunks + reranker | v3 | 91% | 2.1¢ |

Now a decision is easy to defend: run 14 gets most of the quality of run 15 at a quarter
of the cost. A spreadsheet is enough to start. Tools such as MLflow and Langfuse do this
automatically.

**Prompt versions.** Two common approaches:

- **In the repository.** Prompts are files next to the code. Changing one goes through the
  same review and release process as code. Simple and safe.
- **In a prompt registry.** Prompts live in a separate service with version numbers and
  labels like "production" and "staging". Lets you change a prompt without releasing code,
  and lets non-programmers edit them.

Either way: every production answer should be traceable to the exact prompt version that
produced it.

---

## 4. Evaluation as a release gate

A **gate** is a check that must pass before a change can go out. Here it is your level 7
test suite, run automatically.

```
Developer proposes a change
        ↓
Automatic checks run:  code tests  +  evaluation suite
        ↓
Score at or above the bar?  ── no ──►  change is blocked
        ↓ yes
Change can be merged and released
```

*Example rule:* "Overall score must not fall more than 1 point, and no category may fall
more than 3 points."

Practical details:

- **Fast set and full set.** A 40-case set that runs in a minute on every change; the full
  500 cases before a release.
- **Cost.** An eval run uses real model calls. Budget for it, and use a cheaper grader where
  it is accurate enough.
- **Wobble.** Because outputs vary, set the bar with a little slack, or run twice and average.
- **Where the bar comes from.** Start from the current score. Raise it as the system improves.

This one practice prevents most "we changed the prompt and broke something" incidents.

---

## 5. Guardrails at runtime

Level 8 introduced guardrails. In production they run on every request, so they must be
fast and you must watch them.

**A typical line-up:**

```
Request in
  → Is the user within their usage limit?          (stops abuse and runaway cost)
  → Strip personal data that isn't needed
  → Is this on-topic and not an attack?
  → MODEL
  → Does the output match the required format?
  → Does it contain anything it shouldn't?
Response out
```

**Usage limits** deserve emphasis. Without them, one user or one bug can spend a month's
budget overnight.

*Example:* each free user gets 50 requests a day; each paying customer has a monthly token
allowance; and there is a hard daily cap on total spend that pages someone when reached.

**Watch how often each guardrail fires.** A jump from 0.5% to 8% blocked means either an
attack is under way or the check is wrongly blocking normal users. Both need attention.

**Mind the delay.** Every check adds time. Run independent checks at the same time, and
keep them light.

---

## 6. Serving and packaging

Package the app as a container (level 0, section 7). Some points special to AI apps:

**Most of the time is spent waiting.** Your server is not computing; it is waiting for the
model provider. With async code (level 0, section 1), one server process can handle
hundreds of waiting requests at once. With blocking code, each waiting request ties up a
whole worker, and you run out quickly.

**Streaming through middlemen.** Between your app and the user sit load balancers and
proxies. Many collect the whole response before passing it on ("buffering"), which silently
turns your streaming into one long wait.

*Symptom:* streaming works on your laptop, but in production the answer appears all at
once after ten seconds. *Fix:* turn off buffering for those routes and raise the idle
timeout so long responses are not cut off.

**Keep images small and repeatable.** List exact library versions, use a slim base image,
and do not bake secrets into the image.

---

## 7. CI/CD

Two abbreviations:

- **CI, continuous integration:** every change is automatically tested.
- **CD, continuous delivery:** changes that pass are automatically released.

Together they form a **pipeline**: a series of automatic steps from "code pushed" to
"running for users".

**A pipeline with GitHub Actions** (free for public repositories):

```yaml
# .github/workflows/ci.yml
name: ci
on: [pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -r requirements.txt
      - run: pytest                                  # ordinary code tests
      - run: python run_evals.py --set fast --min-score 0.85     # the quality gate
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

**Releasing safely:**

- **Staging first.** A copy of production where changes are tried before real users see them.
- **Canary release.** Send the new version to 5% of users, watch the numbers, then widen.
  Named after the canaries miners carried to detect bad air early.
- **Blue-green.** Run old and new side by side and switch traffic in one move. Switching
  back is just as quick.

---

## 8. Deploying to the cloud

The original course uses Amazon Web Services (AWS). The same building blocks exist under
different names on every cloud, so learn the concepts.

| Building block | What it does | AWS name (for reference) |
|---|---|---|
| Container service | Runs your container and restarts it if it dies | ECS, App Runner |
| Image registry | Stores your container images | ECR |
| Managed database | A database someone else keeps running | RDS |
| Secrets store | Holds keys and passwords safely | Secrets Manager |
| Load balancer | Spreads requests across copies of your app | ALB |
| Access roles | Says which service may do what | IAM |
| Logs and metrics | Collects output and numbers | CloudWatch |

**Autoscaling:** run more copies when busy, fewer when quiet.

**Infrastructure as code.** Instead of clicking through a web console to create these
pieces, describe them in files (with a tool such as Terraform). Benefits: the setup is in
version control, can be reviewed, and can be recreated exactly.

**Start small.** One container service, one database, secrets, logs. Simpler hosts that
run a container straight from a repository are fine for learning and for many real
products. Set a billing alert on day one.

---

## 9. Monitoring and observability

**Monitoring** tells you *that* something is wrong. **Observability** lets you find out *why*.

**Traces.** A trace records one request from start to finish as a tree of steps:

```
Request 8f3a  "Where's my order?"                    total 3.9s   $0.006
├─ check usage limit                                       0.01s
├─ classify question → "order"                             0.30s   small model
├─ tool: lookup_orders(customer 5521)                      2.40s   ← slow!
└─ write answer                                            1.20s   large model
```

At a glance: the delay is in the order lookup, not the model. Without a trace you would be
guessing.

**Dashboards.** One screen with the four groups from level 9: quality, speed, cost and errors.

**Alerts.** Automatic messages when a number crosses a line.

| Alert | Example line |
|---|---|
| Errors | More than 2% of requests failing over 5 minutes |
| Speed | Slowest 5% of requests above 10 seconds |
| Cost | Spend this hour more than three times the usual |
| Quality | Share of sampled answers passing the grader drops below 85% |

Good alerts are rare and always mean "act now". If an alert fires daily and people ignore
it, delete it or fix what it complains about.

**Quality monitoring** is the part that is new with AI. Each day, take a sample of real
conversations, run your level 7 graders on them, and chart the pass rate. That is how you
notice a slow slide that no error message will ever report.

---

## 10. Incidents, rollbacks and change management

An **incident** is when something is broken for users.

**The first move is nearly always to roll back:** return to the last version that worked,
then investigate calmly. This must be fast and practised.

Because behaviour depends on code, prompt and model, you need to be able to roll back each:

- Code: redeploy the previous image.
- Prompt: point "production" at the previous prompt version.
- Model: change the setting back to the previous pinned version.

**A runbook** is a written set of steps for a known problem, so whoever is on duty at 3am
does not need to think from scratch.

```
RUNBOOK: Model provider is failing
1. Check the provider's status page.
2. Confirm the fallback is taking traffic (dashboard: "requests by provider").
3. If the fallback is not working, switch on "maintenance message" mode.
4. Post an update in the team channel.
5. When the provider recovers, switch back and watch errors for 15 minutes.
```

**After the incident,** write a short review with no blame:

- What happened and for how long?
- How did we notice? Could we have noticed sooner?
- What was the cause?
- What will we change so it does not happen again? (Usually: a new test, a new alert, or
  a new guardrail.)

**Change management** just means making changes carefully: small ones, one at a time,
tested, released gradually, with a way back. Ten small releases are safer than one big one,
because when something breaks you know which change did it.

---

## Free resources

- GitHub Actions docs: <https://docs.github.com/en/actions>
- Docker docs: <https://docs.docker.com/>
- MLflow docs: <https://mlflow.org/docs/latest/>
- Langfuse (open-source tracing and prompt management): <https://langfuse.com/docs>
- OpenTelemetry: <https://opentelemetry.io/docs/>
- AWS documentation: <https://docs.aws.amazon.com/>
- Google SRE book: <https://sre.google/sre-book/table-of-contents/>
- Applied LLMs, operations section: <https://applied-llms.org/>

## Project: take what you built to production

Use the level 4 RAG application or the level 5 agent.

- Prompts and model names versioned and pinned.
- A pipeline that runs code tests and the evaluation suite, and blocks a change that lowers the score.
- Automatic release to staging; a manual step to promote to production.
- A trace for every request, and one dashboard covering speed, cost, errors and quality.
- Two alerts, and a runbook for each.
- A drill: release a deliberately bad prompt to staging, watch the gate or the alert catch
  it, and roll back. Write down how long each step took.

## Self-check

1. Why pin a model version, and what is the downside?
2. What is in your release gate, and where did its bar come from?
3. Streaming works on your laptop but not in production. What is the likely cause?
4. Which four numbers would you put on one dashboard?
5. How do you roll back a prompt change without releasing new code?
