# Level 2: Working with LLM APIs

**Phase 1: Foundations** · Previous: [Level 1](01-llm-101.md) · Next: [Level 3](03-prompt-engineering.md)

**Goal:** turn single model calls into dependable software: conversations, streaming,
structured output, tools, multi-step flows, and handling of failure and cost.

**Skip this level if** you have built a streaming, tool-calling application directly on a
provider's library and handled its rate limits in production.

The code below uses the Anthropic Python library. Other providers work the same way with
slightly different names. Model names change often; check your provider's documentation for
the current ones.

---

## 1. First calls and conversation state

Calling a model is an HTTP request (level 0) wrapped in a helper library.

```python
# pip install anthropic     and set the ANTHROPIC_API_KEY environment variable
import anthropic

client = anthropic.Anthropic()

response = client.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=500,
    system="You are a friendly cooking assistant.",
    messages=[{"role": "user", "content": "How long do I boil an egg?"}],
)
print(response.content[0].text)
print(response.usage)   # how many tokens went in and came out
```

There are three kinds of message:

- **system:** standing instructions for the whole conversation.
- **user:** what the person said.
- **assistant:** what the model said.

**The API remembers nothing** (level 1, section 8). To hold a conversation, you keep the
list and send all of it every time:

```python
history = []

def chat(user_text: str) -> str:
    history.append({"role": "user", "content": user_text})
    response = client.messages.create(
        model="claude-sonnet-5-5", max_tokens=500, messages=history
    )
    reply = response.content[0].text
    history.append({"role": "assistant", "content": reply})
    return reply

chat("My name is Priya.")
print(chat("What's my name?"))   # works, because the first exchange was resent
```

In a real app, `history` is saved in a database per user, not in a variable.

---

## 2. Streaming

Without streaming, the user stares at a blank screen until the whole answer is ready. With
streaming, words appear as they are produced. Total time is the same, but it feels far
quicker.

```python
with client.messages.stream(
    model="claude-sonnet-5-5",
    max_tokens=500,
    messages=[{"role": "user", "content": "Tell me a short story."}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

To pass a stream through your own API to a browser, FastAPI has `StreamingResponse`. The
usual format is called server-sent events: a long-lived response where the server keeps
writing small chunks.

Things that go wrong: the connection drops halfway (decide whether to show the partial
answer or retry), and some proxies hold back the chunks and deliver them all at once, which
defeats the point.

---

## 3. Structured output

Programs need data, not paragraphs. If you want to save an order to a database, "Sure! The
customer wants two large pizzas" is useless. You need this:

```json
{"item": "pizza", "size": "large", "quantity": 2}
```

**The approach:** describe the shape you want, ask for JSON, and check what comes back.

```python
from pydantic import BaseModel, ValidationError
import json

class Order(BaseModel):
    item: str
    size: str        # "small", "medium" or "large"
    quantity: int

prompt = """Extract the order from the message below.
Reply with only JSON, in this form: {"item": "...", "size": "small|medium|large", "quantity": 0}

Message: Can I get two large pizzas please"""

response = client.messages.create(
    model="claude-sonnet-5-5", max_tokens=200,
    messages=[{"role": "user", "content": prompt}],
)

try:
    order = Order(**json.loads(response.content[0].text))
except (json.JSONDecodeError, ValidationError) as error:
    # ask again, including the error, or fall back to a human
    ...
```

Most providers also have a built-in structured-output feature where you hand over the
schema and the output is guaranteed to match it. Use that when it is available; it is more
reliable than asking nicely.

Tips for schemas the model fills in well:

- Clear field names (`delivery_date`, not `dd`).
- A fixed list of allowed values wherever possible.
- A way to say "unknown", so the model is not pushed to invent a value.

---

## 4. Tool calling

Models cannot look up today's weather, query your database or send an email. **Tools** fix
that. A tool is a function in your code that the model can ask you to run.

The sequence, which people often get wrong:

1. You tell the model which tools exist and what each does.
2. The model replies, "please call `get_weather` with `city = Paris`". It does not run
   anything itself.
3. **Your code** runs the function.
4. You send the result back.
5. The model writes its answer using the result.

```python
tools = [{
    "name": "get_weather",
    "description": "Get the current weather for a city. Use when the user asks about weather.",
    "input_schema": {
        "type": "object",
        "properties": {"city": {"type": "string", "description": "City name, e.g. Paris"}},
        "required": ["city"],
    },
}]

def get_weather(city: str) -> str:
    return f"18°C and cloudy in {city}"      # a real version would call a weather service

messages = [{"role": "user", "content": "Do I need a jacket in Paris today?"}]
response = client.messages.create(
    model="claude-sonnet-5-5", max_tokens=500, tools=tools, messages=messages
)

if response.stop_reason == "tool_use":
    request = next(block for block in response.content if block.type == "tool_use")
    result = get_weather(**request.input)                     # step 3: YOU run it

    messages.append({"role": "assistant", "content": response.content})
    messages.append({"role": "user", "content": [{
        "type": "tool_result", "tool_use_id": request.id, "content": result,
    }]})
    final = client.messages.create(
        model="claude-sonnet-5-5", max_tokens=500, tools=tools, messages=messages
    )
    print(final.content[0].text)   # "It's 18°C and cloudy, so a light jacket is a good idea."
```

If your function fails, send the error back as the result. The model can often fix its own
mistake, for example by correcting a misspelt city.

---

## 5. Designing good tools

The model chooses tools by reading their names and descriptions. Those are instructions,
so write them with care.

| Weak | Strong |
|---|---|
| `name: "search"` | `name: "search_orders"` |
| `description: "Searches."` | `description: "Find a customer's past orders by email address. Returns up to 10 most recent orders with date, items and status. Use this before answering any question about an existing order."` |
| Returns 5,000 lines of raw data | Returns the 10 most relevant rows, only the useful fields |
| Error: `"Error 500"` | Error: `"No customer found with that email. Ask the user to confirm the address."` |

More rules:

- **One clear job per tool.** `get_order` and `cancel_order`, not `manage_order` with a mode setting.
- **Keep results small.** Everything a tool returns takes space in the context window.
- **Separate reading from changing.** Tools that only look things up are safe to run freely.
  Tools that send, delete or pay should ask the user to confirm first.

---

## 6. Multi-step workflows

One prompt doing five jobs does each of them poorly. Splitting the work into steps, each
with its own focused prompt, is more reliable and easier to fix.

**Four common shapes:**

- **Chain:** one step feeds the next.
  *Example:* write a product description → translate it to Spanish → check the translation.
- **Router:** first classify the input, then send it down the matching path.
  *Example:* a support message is labelled "billing", "technical" or "other", and each
  label has its own prompt and tools. Easy ones can go to a cheaper model.
- **Parallel:** run several calls at once and combine them.
  *Example:* summarise 20 documents at the same time, then summarise the summaries.
- **Evaluator loop:** one call writes, another critiques, the first revises.
  *Example:* write code → review it for bugs → fix → repeat up to three times.

In all of these, **your code decides the order of steps**. That makes them *workflows*. An
*agent* (level 5) is different: there the model decides what to do next. Workflows are
cheaper, faster and more predictable, so try one first.

---

## 7. Async, concurrency and batching

Model calls are slow, so doing them one after another wastes time.

*Example:* summarising 100 reviews at 3 seconds each takes 5 minutes in sequence. Run 10 at
a time and it takes 30 seconds.

```python
import asyncio, anthropic

client = anthropic.AsyncAnthropic()
limit = asyncio.Semaphore(10)        # never more than 10 calls at once

async def summarise(review: str) -> str:
    async with limit:
        response = await client.messages.create(
            model="claude-haiku-5-5", max_tokens=100,
            messages=[{"role": "user", "content": f"Summarise in one sentence: {review}"}],
        )
        return response.content[0].text

async def main(reviews: list[str]):
    return await asyncio.gather(*(summarise(r) for r in reviews))
```

The semaphore matters. Providers set **rate limits** (requests and tokens per minute). Send
500 requests at once and most come back as 429 errors.

**Batch processing:** if you do not need answers right away, providers let you submit a
large file of requests and collect results later, at a reduced price. Good for overnight
jobs such as labelling a whole dataset.

---

## 8. Robustness

Model providers have outages, slow moments and rate limits. Code that assumes every call
succeeds will fail in front of users.

**Timeouts.** Never wait forever. Set a limit and decide what happens when it passes.

**Retries with backoff.** If a call fails with 429 or 5xx, wait and try again, waiting
longer each time, with a little randomness so thousands of clients do not all retry at the
same instant.

```python
import random, time, anthropic

def call_with_retry(make_call, attempts: int = 4):
    for attempt in range(attempts):
        try:
            return make_call()
        except (anthropic.RateLimitError, anthropic.InternalServerError):
            if attempt == attempts - 1:
                raise
            time.sleep(2 ** attempt + random.random())   # about 1s, 2s, 4s
```

The official libraries retry a couple of times on their own; know what yours does before
adding more.

**Do not retry 400-type errors.** The request is wrong and will be wrong again.

**Fallbacks.** If the main model is unavailable, try a second one, perhaps from another provider.

**Idempotency.** A big word for "safe to do twice". If a retry could send the same email
twice or charge a card twice, give each action a unique ID and have the receiving side
ignore repeats.

---

## 9. Cost and observability

You cannot fix what you cannot see. From the first day, record every model call.

**What to log, and the question each answers:**

| Field | Question |
|---|---|
| Model name | Which model did this? |
| Input and output tokens | What did it cost? |
| Time taken | Was it slow? |
| Success or error | Did it work? |
| A request ID | Which user request did this belong to? |
| Prompt version | Which wording produced this? |

**Working out cost:**

```
cost = (input tokens × input price) + (output tokens × output price)

Example: 2,000 tokens in, 500 out, at $3 and $15 per million tokens
       = (2,000 × $3 / 1,000,000) + (500 × $15 / 1,000,000)
       = $0.006 + $0.0075  =  about 1.4 cents
```

At 50,000 requests a day that is about $675 a day. Do this sum before launch, not after the bill.

**Tracing** records every step of a multi-step request as a tree, so you can see that step
3 of 6 took eight seconds or returned nonsense. Open-source tools such as Langfuse do this.

---

## 10. Frameworks

Libraries such as LangChain, LlamaIndex and Pydantic AI wrap the patterns above so you
write less code.

- **Benefit:** ready-made pieces for common tasks, and one interface over many providers.
- **Cost:** when something goes wrong, the actual prompt and request are hidden under
  layers, which makes debugging harder. Their interfaces also change often.

Advice: build each thing once by hand, as in this file, so you know what is happening. Then
use a framework where it saves real work. Being able to read framework code is useful for
jobs, since many teams use one.

---

## Free resources

- Anthropic API docs: <https://docs.anthropic.com/>
- OpenAI API docs: <https://platform.openai.com/docs>
- "Building effective agents" (covers the workflow shapes): <https://www.anthropic.com/engineering/building-effective-agents>
- Pydantic: <https://docs.pydantic.dev/>
- LangChain docs: <https://python.langchain.com/>
- LiteLLM, one interface to many providers: <https://docs.litellm.ai/>
- Langfuse, open-source tracing: <https://langfuse.com/docs>

Most providers give new accounts a small amount of free credit, and some offer free tiers
with low limits. Small models cost fractions of a cent per call, so the whole of this
level can be done for a few dollars.

## Project: a streaming assistant that uses tools

- A FastAPI endpoint that streams responses to a simple web or terminal client.
- Three tools, at least one of which calls a real outside service.
- One endpoint that returns checked, structured data instead of prose.
- Timeouts, retries with backoff, and a fallback model.
- A log line per call with tokens, time and cost.
- **Evaluation step:** 15 test inputs, each with the tool and inputs you expect. Report how
  many the assistant gets right, and re-run after every change.

## Self-check

1. Who actually runs a tool, and what does the model see afterwards?
2. Your JSON output is invalid 2% of the time. What are three ways to handle it?
3. What is the difference between a workflow and an agent?
4. How do you stop 500 simultaneous requests from hitting a rate limit?
5. Which fields would you log on every call, and what question does each answer?
