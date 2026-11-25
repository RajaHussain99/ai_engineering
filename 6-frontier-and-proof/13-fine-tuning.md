# Level 13: Fine-tuning

**Phase 6: Frontier & proof** · Previous: [Level 12](12-multimodal.md) · Next: [Level 14](14-portfolio-projects.md)

**Goal:** fine-tune a model only when prompting, retrieval and workflow design have
measurably run out, and prove with numbers that the result was worth the data, training
and running costs.

**Skip or defer this level if** you have not yet hit a quality or cost ceiling that better
prompts and retrieval cannot move.

**Fine-tuning** means training an existing model a bit further on your own examples, so
its behaviour shifts in the direction you want.

Analogy: a prompt is giving a new employee written instructions each morning. Fine-tuning
is sending them on a training course, after which they just know how you do things.

---

## 1. Should you fine-tune?

Usually not yet. Try these first, in order, because each is cheaper and quicker to change:

```
1. Better prompt                 (minutes)
2. Add examples to the prompt    (minutes)
3. Add retrieval (RAG)           (days)
4. Break the task into steps     (days)
5. Fine-tune                     (weeks, plus upkeep forever)
```

**Good reasons:**

- **A style or format the prompt cannot hold.** Every reply must be in your company's
  exact voice, and despite a long prompt with examples it drifts 10% of the time.
- **Cost and speed at high volume.** A large model does the task at 2 cents a request, 5
  million times a day. A small model, fine-tuned for just this task, might match it at a
  twentieth of the price.
- **A narrow specialised task with lots of examples.** Sorting medical notes into 400
  billing codes, with 50,000 labelled notes available.
- **A shorter prompt.** If your prompt carries 3,000 tokens of instructions and examples,
  a fine-tuned model may need none of them.

**Bad reasons:**

- **"To teach it our company's facts."** Fine-tuning is poor at adding facts. The model
  half-learns them and blends them with invented ones. Use RAG (level 4). Fine-tuning
  changes *how* a model behaves; retrieval changes *what it knows*.
- **"Our answers aren't good"** with no measurement. You cannot improve what you have not measured.
- **"It sounds advanced."**

**Before starting you must have:**

1. A test set and a way to score it (level 7).
2. A **baseline**: the score of your best prompt on that test set.
3. A target: "we need 92%; the best prompt gives 84%."

Without these you cannot tell whether the fine-tune helped.

---

## 2. Preparing the data

The data decides the result. A fine-tuned model copies its examples, flaws included.

**What an example looks like.** Usually a short conversation with the ideal reply:

```json
{"messages": [
  {"role": "system", "content": "You write product descriptions for Lumen Outdoor."},
  {"role": "user", "content": "Tent, 2-person, 1.8kg, waterproof to 3000mm"},
  {"role": "assistant", "content": "Light enough to forget it's in your pack. This two-person tent weighs just 1.8kg and shrugs off heavy rain."}
]}
```

One example per line, in a file.

**Where examples come from:**

- **Your records:** real inputs with outputs that people approved. Best source.
- **Experts:** people writing ideal answers. Slow, high quality.
- **A larger model:** have a strong model produce outputs, then check them (see section 7).

**How many?** A few hundred good ones is a fair start for a narrow task. Quality matters
more than quantity: 300 clean, varied examples beat 5,000 sloppy ones.

**A cleaning checklist:**

- **Read a sample yourself.** Always. You will find surprises.
- **Remove duplicates** and near-duplicates.
- **Fix or drop wrong answers.** If 5% of your labels are wrong, the model learns to be
  wrong about 5% of the time.
- **Check variety.** If every example is about tents, it will struggle with boots.
- **Check balance.** If 90% of examples are "approve", it will lean towards "approve".
- **Remove personal data.**

**Split the data three ways:**

| Part | Share | Used for |
|---|---|---|
| Training | about 80% | The model learns from these |
| Validation | about 10% | Checked during training to spot problems |
| Test | about 10% | Looked at only at the end, for the honest score |

The test examples must never appear in training. If they do, the score is a memory test
and tells you nothing. This mistake is called **leakage**.

---

## 3. Supervised fine-tuning

"Supervised" means learning from examples that include the right answer. It is the basic
and most common kind.

**What happens:** the model sees an input, produces its guess, the guess is compared with
your ideal answer, and the model's internal numbers are nudged to make the right answer
more likely. This repeats across all examples, several times over.

**The settings that matter:**

| Setting | Plain meaning | If too low | If too high |
|---|---|---|---|
| **Epochs** | How many times it goes through all the examples | Has not learned enough | Memorises the examples |
| **Learning rate** | How big each nudge is | Learns very slowly | Becomes unstable, forgets what it knew |
| **Batch size** | How many examples it looks at per nudge | Slower, noisier | Needs more memory |

Start with the defaults your tool suggests and change one thing at a time.

**Overfitting** is the main danger. The model memorises the training examples instead of
learning the general pattern, like a student who memorises last year's exam answers and
then fails this year's.

**How to spot it:** watch two numbers during training. "Loss" is a measure of how wrong
the model is; lower is better.

```
Epoch   Training loss   Validation loss
  1         1.20             1.25
  2         0.80             0.90
  3         0.50             0.85     ← best point: stop here
  4         0.25             0.95     ← training keeps improving, validation gets worse
  5         0.10             1.10     ← memorising
```

When the validation number stops falling and starts rising, the model has begun
memorising. Use the version from the best point.

**The chat template.** Each model expects conversations laid out in a particular way with
special markers. Use the model's own template through your library; a mismatch quietly
ruins results.

**Two ways to do it:** through a provider's fine-tuning service (you upload a file; they
handle the rest), or yourself on an open-weight model with open-source tools.

---

## 4. LoRA and QLoRA

Training every number in a large model needs enormous memory and produces a full-size copy
each time. **LoRA** avoids this.

**The idea:** freeze the original model. Add a small set of new numbers alongside it, and
train only those. The small set is called an **adapter**.

Analogy: instead of reprinting a whole textbook to add your notes, you slip a few
transparent sheets over the relevant pages. The book is unchanged; the sheets hold what is
new.

**Why it is so popular:**

- **Tiny files.** The model might be 15 GB; the adapter 50 MB.
- **Far less memory and time** to train.
- **Swappable.** One base model, many adapters: one for legal writing, one for support,
  one for product descriptions.
- **Quality close to full fine-tuning** for most tasks.

**QLoRA** adds one trick: load the frozen model in a squeezed, lower-precision form
(quantised, from level 11) so it fits in much less memory. This is what lets you fine-tune
a decent model on one consumer graphics card, or on a free cloud notebook.

**Settings you will see:**

- **Rank (r):** the size of the adapter. Typical values 8 to 64. Higher can learn more and
  can overfit more.
- **Alpha:** how strongly the adapter's changes apply. Often set to the rank or double it.
- **Target modules:** which parts of the model get adapters. Use the tool's default to start.

```python
from peft import LoraConfig

config = LoraConfig(r=16, lora_alpha=32, lora_dropout=0.05, task_type="CAUSAL_LM")
```

Free notebooks from projects such as Unsloth walk through a full QLoRA run on a free GPU.

---

## 5. Preference tuning

Supervised fine-tuning needs "the right answer". Some qualities have no single right
answer, but people can still say which of two answers is better. Preference tuning learns
from those comparisons.

**An example:**

```json
{
  "prompt": "A customer says their order arrived broken.",
  "chosen": "I'm sorry that happened. I've arranged a replacement, which will arrive by Friday. You don't need to return the broken one.",
  "rejected": "Per our policy section 4.2, damaged items may be eligible for replacement subject to verification. Please submit form D-17."
}
```

Both are "correct". One is clearly better. The model learns to move towards the kind of
answer that was chosen and away from the kind that was rejected.

The commonly used method is **DPO** (Direct Preference Optimization). Its appeal is that it
is simple: it needs only these pairs, with no extra scoring model.

**Good for:** tone, helpfulness, brevity, when to decline, house style.

**Where pairs come from:** thumbs up/down in your product; an editor's rewrite (the
original is "rejected", the rewrite is "chosen"); two candidate answers with a person
picking one.

Usually done after supervised fine-tuning, as a polish.

---

## 6. Reinforcement fine-tuning

Here the model is not shown answers at all. It tries, gets a **score** from a grader, and
is adjusted towards whatever earned higher scores. Learning by trial and feedback, the way
you learn a video game.

```
Model attempts a task → grader gives a score → model is adjusted → repeat
```

**Good for:** tasks where a result can be scored automatically but there is no single
right way to get there. Writing code that must pass tests. Solving maths with a checkable
answer. Multi-step tasks with a clear success test.

**The catch: the grader is everything.** The model will find any way to score well,
including ways you did not intend.

*Example.* The grader rewards "all tests pass". The model learns to delete the failing
tests. The score is perfect and the result is worthless.

*Another.* The grader rewards longer answers, assuming they are more thorough. The model
learns to pad.

This is why a model's score can go up while its real usefulness goes down. So:

- Write the grader with great care and try to cheat it yourself first.
- Read actual outputs during training, not only the score.
- Keep a separate test that the grader does not cover.

This is the most advanced method here. It needs many attempts and is easy to get wrong.

---

## 7. Distillation

**Distillation** means training a small model to copy a large one. The large model is the
"teacher", the small one the "student".

**Steps:**

1. Collect many real inputs for your task (no answers needed).
2. Run them through the large, expensive model and save its outputs.
3. Check a sample for quality; drop bad ones.
4. Fine-tune a small model on those input-output pairs.
5. Compare the student with the teacher on your test set.

**Example.**

```
Task: sort support tickets into 30 categories.
Teacher (large model with a long prompt):  95% accurate,  1.5¢ each,  2 seconds
Step:  have the teacher label 20,000 real tickets (one-off cost ≈ $300)
Student (small model, fine-tuned):          93% accurate,  0.05¢ each, 0.3 seconds
```

At 2 million tickets a month, that is $30,000 a month down to $1,000, for two points of accuracy.

This is one of the most practical uses of fine-tuning: you need no hand-labelled data, and
the payoff in cost and speed is easy to measure.

**Limits:** the student is usually a little worse and cannot exceed the teacher on that
task. It is good at this one job only. And check the teacher model's terms of use, since
some providers restrict using outputs to train other models.

---

## 8. Evaluating a fine-tune

The question is never "is the fine-tuned model good?" It is "is it **better than the
alternatives**, and by enough to justify the trouble?"

**Always compare at least three things** on the held-back test set:

| | Accuracy | Cost per 1,000 | Speed |
|---|---|---|---|
| Small model + best prompt | 84% | $0.40 | 0.4s |
| Large model + best prompt | 93% | $9.00 | 2.0s |
| **Small model, fine-tuned** | **92%** | **$0.40** | **0.4s** |

Here the fine-tune clearly wins: nearly the large model's accuracy at the small model's price.

But the table might instead read 84%, 93%, **85%**. Then the fine-tune barely helped and
you should not ship it. That is a valid result. Report it honestly.

**Check for damage elsewhere.** Fine-tuning for one task can make the model worse at
others, sometimes called "catastrophic forgetting".

*Example:* after tuning for product descriptions, the model has become worse at following
a simple instruction like "reply in French". Keep a small general test and run it before
and after.

**Read the outputs.** Numbers hide things. Look at 50 results yourself, especially the failures.

---

## 9. Serving the adapter

After training you have a base model and an adapter. To use them:

**Option 1: merge.** Combine the adapter into the base model to make one new model file.
Simple to run and slightly faster. But you get a full-size file per fine-tune.

**Option 2: load at runtime.** Keep one copy of the base model in memory and attach
adapters as needed. Good when you have many adapters (say one per customer), since they
share the big base model.

**Shrinking for serving.** Quantising the final model (level 11) cuts memory and cost.
Re-run your test set afterwards, because quantising can cost a little quality.

**Where to run it:** your own server with software such as vLLM, or a hosting company that
accepts custom adapters. If you used a provider's fine-tuning service, they host it for you.

**The full cost, which people underestimate:**

| Cost | When |
|---|---|
| Preparing data | Once, and it is usually the biggest piece of work |
| Training runs | Each attempt; expect several |
| Hosting | Every month, whether traffic is high or low |
| **Re-doing it** | **Every time a better base model appears** |

That last row matters most. A prompt can be moved to a new model in an afternoon. A
fine-tune must be redone and re-tested. New base models arrive often, and sometimes the
new base model with a plain prompt beats your old fine-tune. Count that into the decision.

---

## Free resources

- LoRA paper: <https://arxiv.org/abs/2106.09685>
- QLoRA paper: <https://arxiv.org/abs/2305.14314>
- DPO paper: <https://arxiv.org/abs/2305.18290>
- Hugging Face PEFT: <https://huggingface.co/docs/peft>
- Hugging Face TRL: <https://huggingface.co/docs/trl>
- Hugging Face LLM course: <https://huggingface.co/learn/llm-course>
- Unsloth (free notebook examples): <https://github.com/unslothai/unsloth>
- Google Colab, for a free GPU: <https://colab.research.google.com/>
- vLLM: <https://docs.vllm.ai/>

## Project: a LoRA that beats a prompted baseline

- Choose a narrow task with a measurable answer, such as sorting into 20 categories or
  converting text into a strict format.
- Build 500 to 2,000 examples and hold back 200 for testing.
- Baseline 1: a small model with your best prompt and examples.
- Baseline 2: a large hosted model with your best prompt.
- Train a QLoRA adapter on the small model using a free notebook GPU.
- Report accuracy, speed and cost per 1,000 requests for all three.
- Conclude honestly. "The fine-tune was not worth it" is an acceptable result if the
  numbers say so.

## Self-check

1. Why is fine-tuning the wrong way to give a model new facts?
2. What does LoRA train, and why is the resulting file so small?
3. What must exist before you start a fine-tune?
4. How can a model's score rise during reinforcement fine-tuning while it gets worse in practice?
5. What ongoing costs does a fine-tuned model carry that a prompted one does not?
