# Level 11: Managed AI platforms

**Phase 5: Production** · Previous: [Level 10](10-llmops.md) · Next: [Level 12](../6-frontier-and-proof/12-multimodal.md)

**Goal:** ship an AI application on the platforms that companies actually buy, and know
which parts to hand to a vendor and which to keep yourself.

**Skip or defer this level if** you are not yet working somewhere with cloud contracts,
compliance rules or requirements about where data is stored. Come back when a job needs it.

A "managed" service is one where another company runs the hard parts for you. This level
is about the business and practical side of getting access to models.

---

## 1. Three ways to use a model

| | A. The maker's own service | B. Through a cloud provider | C. Open-weight model |
|---|---|---|---|
| What it is | You call the company that built the model | The same model, offered inside AWS, Google Cloud or Microsoft Azure | A model whose files are public; you or a host run it |
| Setup | Minutes | Hours to days | Days to weeks if you run it yourself |
| Newest features | First | Sometimes later | Depends on the model |
| Billing | A new vendor | On your existing cloud bill | Hardware or a host's fee |
| Control | Least | Medium | Most |
| Work for you | Least | Medium | Most |

**Analogies:** A is buying coffee from the roaster's own café. B is buying the same beans
at the supermarket where you already shop. C is roasting your own.

**How people choose:**

- A startup that wants the newest features and quick setup usually picks A.
- A bank whose security team has already approved one cloud, and which needs a year to
  approve a new vendor, picks B.
- A company that cannot let data leave its own servers, or that has huge steady volume,
  looks at C.

---

## 2. What a cloud AI platform contains

The three big ones are Amazon Bedrock, Google Vertex AI and Microsoft Azure AI Foundry.
They differ in detail but offer the same kinds of thing:

- **A model catalogue:** models from several makers behind one account.
- **An inference service:** the way you call a model. ("Inference" means using a trained model.)
- **Guardrails:** ready-made input and output checks (level 8).
- **Knowledge bases:** a managed RAG pipeline; you point it at documents (level 4).
- **Agent services:** a managed agent loop with tools (level 5).
- **Evaluation tools:** ways to run test sets (level 7).
- **Fine-tuning** for some models (level 13).

**What to check before committing:**

- **Is the exact model version you want available?** Cloud platforms sometimes get new
  models and features later than the maker's own service.
- **Are all the features there?** A feature in the maker's service may be missing or work
  differently on the cloud platform.
- **Are the names the same?** Model names often differ between the maker and each cloud.

Never assume. Run your own test set (level 7) on the platform you will actually use.

---

## 3. Identity, networks and tenancy

These three topics are why large companies prefer option B.

**Identity: roles, not keys.** An API key is a password. If it leaks, anyone can use it
until someone notices. Clouds offer something safer: you give your *application* a role
("this service may call models, and nothing else"), and the cloud hands it short-lived
credentials automatically. No long-lived secret exists to steal.

**Networks: private connections.** By default your request travels over the public
internet (encrypted). Companies with strict rules want traffic to stay inside their
private cloud network. Clouds offer "private endpoints" for this. Think of an internal
corridor between two offices instead of walking along the public street.

**Tenancy: keeping customers apart.** A "tenant" is one of your customers. If you serve
many companies from one system, you must stop one affecting another.

*Example:* without limits per tenant, one customer runs a huge job, uses up your whole
rate limit, and every other customer gets errors. Fixes, from lightest to strongest:
separate usage limits per customer; separate keys; separate projects or accounts.

Also keep each tenant's data separate in search (level 8, section 5).

---

## 4. Regions, data location and compliance

**Regions.** Clouds run data centres in many parts of the world. You choose which region
handles your requests.

**Data residency** means rules about where data may be processed or stored. Some laws and
many contracts require that certain data stays inside a country or area.

*Example:* a German hospital's contract says patient data must be processed inside the
European Union. You must pick an EU region and confirm the model is available there.

**A trap: cross-region routing.** To handle busy periods, platforms can send requests to
other regions automatically. That is good for reliability and may break a residency
promise. Check what the setting does and which regions it can use.

**Compliance.** Standards and laws you will hear named: SOC 2 (security practices), ISO
27001 (security management), HIPAA (US health data), GDPR (EU personal data). Cloud
providers publish which of their services meet which standards.

**Questions to ask any provider:**

- Do you keep my requests and responses? For how long?
- Are they used to train models?
- Who at your company can see them?
- Can I get an agreement that covers my legal duties?

Get the answers in writing; they differ by provider and by plan.

---

## 5. Capacity, limits and cost

**Ways to pay:**

| Option | How it works | Good for |
|---|---|---|
| Pay as you go | Per token, shared capacity | Getting started; uneven traffic |
| Reserved capacity | A fixed fee for guaranteed throughput | Large, steady traffic that must not slow down |
| Batch | Submit a file of requests, results later, lower price | Jobs where nobody is waiting |

*Example:* a company has a live chat feature and a nightly job that labels 2 million
records. Chat goes on pay-as-you-go (or reserved, if busy). The nightly job goes through
batch at a discount.

**Limits (quotas).** Every account has caps on requests and tokens per minute. New
accounts start low.

*Classic mistake:* everything works in testing at 10 requests a minute. On launch day
traffic hits 2,000 a minute and most requests are refused. Raising a limit means asking
the provider, which can take days. **Ask well before launch.**

**Knowing where the money goes.** One bill for the whole company tells you nothing. Label
usage by team and by feature (clouds call these "tags", or use separate projects), so you
can say "the search feature costs $9,000 a month and the summariser $400".

---

## 6. Open-weight models

"Open-weight" means the model's files are published, so anyone can download and run them.

**Two ways to use them:**

- **A hosting company runs it.** You call it like any other model service. Easy, and often
  cheap per token.
- **You run it yourself** on rented or owned graphics cards (GPUs), using serving software
  such as vLLM, or Ollama for trying things on a laptop.

```bash
# trying an open-weight model on your own computer, free
ollama run llama3.2
```

**Why run your own:** data never leaves your machines; no per-token fee; you can modify
the model (level 13); no provider can retire it on you.

**Why not:** you now run servers. You handle capacity, updates, failures and scaling, and
you pay for the GPU whether it is busy or idle.

**Terms you will meet:**

- **GPU memory:** the model must fit in it. Bigger models need bigger, dearer cards.
- **Quantisation:** storing the model's numbers at lower precision so it fits in less
  memory and runs faster, at a small cost in quality. Like saving a photo at lower quality
  to make the file smaller.
- **Batching:** serving several requests at once on one GPU for efficiency.

**The break-even sum.** Work it out before buying hardware.

```
Renting one GPU: about $1.50 an hour ≈ $1,100 a month, running nonstop
Hosted service for a similar model: say $0.50 per million tokens

$1,100 ÷ $0.50 = 2,200 million tokens a month to break even
```

If you use 200 million tokens a month, the hosted service costs $100 and the GPU $1,100.
Self-hosting loses badly. At 5,000 million tokens it wins. Most teams are below break-even
and should not self-host for cost reasons alone. (Figures are illustrative; check current prices.)

---

## 7. Model routers and gateways

Level 9 introduced gateways. Here is the buying decision.

**A gateway or router** sits between your app and several model providers and gives you
one way to call them all.

```
                     ┌──► Provider A
Your app ──► Gateway ├──► Provider B
                     └──► Your own hosted model
```

What you get: automatic switch-over when a provider fails, spreading load, one log, one
place for limits and budgets, and changing models with a setting.

**Hosted router vs running your own:**

| | Hosted router | Your own gateway |
|---|---|---|
| Setup | Sign up | Deploy and maintain it |
| Data path | Passes through another company | Stays with you |
| Cost | A small added fee | Your time and a server |
| If it fails | You wait for them | You fix it |

Note the last row: a gateway is a single point that everything depends on. If it goes
down, all your AI features go down. Run more than one copy, or have a direct path as backup.

---

## 8. Managed building blocks: build or buy

Platforms sell ready-made versions of things you learned to build: search indexes,
knowledge bases, agent loops, guardrails.

**Buy when:**

- The piece is not what makes your product special.
- The vendor's limits fit your needs.
- You have a small team and need it working this month.

**Build when:**

- Quality in that piece is your product. If search quality is what customers pay for, you
  need to control chunking, hybrid search and reranking yourself.
- The ready-made version hides the controls you need.
- The price grows badly with your volume.

*Example.* An internal HR assistant over 200 documents: buy a managed knowledge base and
have it running in an afternoon. A legal research product where finding the right passage
is the entire value: build your own search.

A middle path that works well: start with the bought version to learn what you need, and
replace the parts that hold you back.

---

## 9. Portability, lock-in and moving

**Lock-in** means it has become expensive to leave a vendor. Some is fine; just know how
much you have.

**How to stay movable:**

**A thin layer of your own.** Do not scatter one provider's code all through your app.

```python
# Everything in your app calls this one function.
def ask_model(system: str, messages: list[dict], size: str = "medium") -> str:
    ...   # the only place that knows which provider is used
```

Switching provider then means changing one file.

**Your test set is your moving tool.** The scary part of switching is not the code; it is
not knowing whether quality held. With a level 7 test set you find out in an hour:

```
                 Current model    Candidate
Quality score         88%            86%
Typical speed         2.1s           1.4s
Cost per request      0.9¢           0.3¢
```

Now it is a clear business decision instead of a gamble.

**Prompts do not transfer unchanged.** A prompt tuned for one model often does worse on
another. Plan time to adjust and re-test.

**Models get retired.** Providers switch off old versions on published dates. Keep a list
of which versions you use and when each ends, so a retirement is a planned task and not an outage.

---

## Free resources

- Amazon Bedrock docs: <https://docs.aws.amazon.com/bedrock/>
- Vertex AI docs: <https://cloud.google.com/vertex-ai/docs>
- Azure AI Foundry docs: <https://learn.microsoft.com/azure/ai-foundry/>
- vLLM: <https://docs.vllm.ai/>
- Ollama, for local open-weight models: <https://ollama.com/>
- LiteLLM: <https://docs.litellm.ai/>
- OpenRouter docs: <https://openrouter.ai/docs>
- Hugging Face Inference Endpoints: <https://huggingface.co/docs/inference-endpoints>

## Project: deploy on a managed platform

- Move one of your applications to a cloud platform's model service, using a role instead
  of a stored key. Most clouds give new accounts free credit; set a budget alert first.
- Put a gateway in front with a fallback to a second provider.
- Run one open-weight model yourself, on your laptop with Ollama or on a rented GPU for an
  hour, and compare quality, speed and cost per million tokens with the managed option.
- Run your test set against all three and put the results in a table.
- Write one page: which would you choose for a customer with strict data rules, and why?

## Self-check

1. Why would a company use a model through its cloud provider instead of the maker's own service?
2. What can cross-region routing do to a promise about where data is processed?
3. At what point does running your own open-weight model become cheaper?
4. What does a gateway give you, and what new risk does it add?
5. How do you keep a change of provider from turning into a rewrite?
