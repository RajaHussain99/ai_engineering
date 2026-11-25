# Level 4: Retrieval-augmented generation (RAG)

**Phase 2: Retrieval** · Previous: [Level 3](../1-foundations/03-prompt-engineering.md) · Next: [Level 5](../3-agentic/05-ai-agents.md)

**Goal:** give a model access to knowledge it was not trained on, by building a search
system and then writing answers based on what the search finds.

**Skip this level if** you have built a retrieval system and measured how often it finds
the right documents.

## The idea in one picture

A model knows nothing about your company's handbook, yesterday's news or a customer's
order history. RAG is an open-book exam: before answering, look up the relevant pages and
hand them to the model along with the question.

```
Question: "How many days of parental leave do we get?"
     │
     ▼
 1. SEARCH your documents  ──►  finds: "HR Handbook, section 4.2: Employees receive
     │                                   16 weeks of paid parental leave..."
     ▼
 2. BUILD a prompt:  "Using only the text below, answer the question. <text>…</text>"
     │
     ▼
 3. The model answers: "You get 16 weeks of paid parental leave (HR Handbook, 4.2)."
```

"Retrieval" is step 1. "Generation" is step 3. Most bad RAG answers come from step 1:
if the search hands over the wrong pages, no model can answer well. Spend your effort there.

---

# Part A: the search layer

## 1. Ingestion and document processing

**Ingestion** means getting your documents into a form you can search: reading the files
and turning them into clean text.

Easy cases: plain text, Markdown, simple web pages.

Hard cases, and what goes wrong:

| Source | Typical problem |
|---|---|
| PDF with two columns | Text from both columns gets mixed line by line |
| Tables | Rows and columns flatten into a meaningless stream of numbers |
| Scanned pages | There is no text, only a picture; you need OCR (text recognition) |
| Headers and footers | "Page 7 of 90 · Confidential" repeats in every chunk |

Always open the extracted text and read some. Garbage in at this step cannot be fixed later.

**Keep metadata** with each document: where it came from, its title, section, date, and who
is allowed to see it. You will use it to filter searches ("only 2025 policies") and to show
sources in answers.

---

## 2. Chunking

A 200-page handbook is too big and too mixed to search as one item. You cut it into
**chunks**: passages of a few paragraphs each. Search then returns the few chunks that matter.

**Ways to cut:**

- **Fixed size:** every 500 words, with some overlap so sentences on a boundary are not lost.
  Simple, but it cuts through the middle of ideas.
- **By structure:** split at headings and paragraphs. Usually better, because each chunk is
  one topic.
- **By meaning:** start a new chunk when the subject changes. More work, sometimes worth it.

**The test for a good chunk:** does it make sense on its own?

*Bad chunk:*
```
It increases to 20 days after five years. Unused days do not carry over.
```
What increases? This chunk is useless alone.

*Better chunk, with context added at the top:*
```
[HR Handbook > Leave > Annual leave]
Full-time employees get 15 days of annual leave. It increases to 20 days after five
years. Unused days do not carry over.
```

Adding the document title and section heading to every chunk is a cheap trick that often
helps more than fiddling with chunk size.

**Size trade-off:** small chunks are precise but lose surrounding context. Large chunks
keep context but bury the answer in unrelated text. A few hundred words is a common
starting point; test on your own documents.

---

## 3. Embeddings

An **embedding** turns a piece of text into a list of numbers (a vector) that captures its
meaning. Texts that mean similar things get similar numbers.

Picture a map where every sentence has a location:

```
"How do I reset my password?"      ●
"I forgot my login details"        ●   ← close together: similar meaning
                                        
"Best pizza in Naples"                           ●   ← far away
```

The two password sentences share almost no words, yet they land next to each other. That
is the power of embeddings: search by meaning, not by matching words.

```python
# pip install sentence-transformers    (runs on your own computer, free)
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("all-MiniLM-L6-v2")
vectors = model.encode([
    "How do I reset my password?",
    "I forgot my login details",
    "Best pizza in Naples",
])
print(vectors.shape)   # (3, 384): three texts, 384 numbers each
```

Things to know when choosing an embedding model:

- Bigger models are usually more accurate, and slower.
- Each has a maximum input length. Text past it is cut off silently.
- You must use the **same model** for your documents and your questions.
- If you change model later, you must re-embed everything.

---

## 4. Similarity and semantic search

Once everything is a vector, "find relevant text" becomes "find the nearest vectors".
Closeness is usually measured with **cosine similarity**: a score where 1 means the same
direction (same meaning) and 0 means unrelated.

```python
from sentence_transformers import util

question = model.encode("I can't log in")
scores = util.cos_sim(question, vectors)[0]
# roughly: [0.62, 0.71, 0.05]  → the two login sentences score high, pizza scores low
```

**A complete tiny search engine:**

```python
chunks = ["...chunk 1...", "...chunk 2...", "..."]          # your documents
chunk_vectors = model.encode(chunks)

def search(question: str, k: int = 3) -> list[str]:
    scores = util.cos_sim(model.encode(question), chunk_vectors)[0]
    best = scores.argsort(descending=True)[:k]
    return [chunks[i] for i in best]
```

This compares the question to every chunk, which is fine up to tens of thousands of chunks.
You do not need a special database to start.

---

## 5. Vector databases and fast search

With millions of chunks, comparing against every one is too slow. A **vector database**
solves this with an index that finds *almost* the nearest vectors very quickly. This is
called approximate nearest neighbour search.

Analogy: to find the closest café, you do not measure the distance to every café in the
country. You look in your own neighbourhood. You might occasionally miss one that is
slightly closer across the boundary, but you are thousands of times faster.

The names you will see for index types are HNSW and IVF. You rarely need the internals,
only the trade: a little accuracy for a lot of speed.

Vector databases also give you:

- **Filtering:** "nearest chunks where department = 'HR' and year = 2025".
- **Storage** of the text and metadata alongside the vectors.
- **Updates** without rebuilding everything.

Options: Chroma (simple, runs locally), pgvector (adds vectors to PostgreSQL, good if you
already use it), Qdrant, and others. Start with the simplest thing that works.

```python
# pip install chromadb
import chromadb
db = chromadb.Client()
collection = db.create_collection("handbook")
collection.add(ids=["c1", "c2"], documents=["chunk one text", "chunk two text"],
               metadatas=[{"section": "leave"}, {"section": "pay"}])
results = collection.query(query_texts=["How much holiday do I get?"], n_results=2)
```

---

## 6. Keyword search and hybrid search

Meaning-based search has a blind spot: exact terms.

*Example:* a user searches for error code `ERR_4471`. To an embedding model that looks much
like `ERR_4417` or any other code. It may return the wrong one. The same happens with
product numbers, people's names and rare jargon.

**Keyword search** matches actual words. The standard method is called **BM25**. It scores
a document higher when it contains the search words, more so when those words are rare.

| Query | Meaning search | Keyword search |
|---|---|---|
| "I can't get into my account" | Finds "password reset" article | Misses it (no shared words) |
| "ERR_4471" | May return a different error code | Finds the exact code |

Neither wins every time, so use both. That is **hybrid search**: run the two searches and
merge the lists. A simple merging method is reciprocal rank fusion: a chunk gets points for
ranking high in either list, and the points are added.

```
Meaning search ranks:  A, B, C, D
Keyword search ranks:  C, E, A, F
Merged:                A and C come first, because both searches liked them
```

Hybrid search is one of the most dependable upgrades you can make.

---

# Part B: answering on top of search

## 7. When to use RAG

Use RAG when the knowledge is:

- **Too big** to paste into every prompt (thousands of documents).
- **Changing** (prices, policies, tickets).
- **Private** (not in any model's training).

Do not use it when:

- Everything fits in the context window. If your whole knowledge base is a 20-page
  document, just include it. Simpler and often more accurate.
- The question needs the *whole* collection ("what are the five most common complaints
  across all 10,000 reviews?"). Search returns a handful of chunks, not a full count.

---

## 8. A basic pipeline from scratch

Put the pieces together:

```python
def answer(question: str) -> str:
    found = search(question, k=5)                      # from section 4
    sources = "\n\n".join(f"<source id='{i+1}'>\n{c}\n</source>" for i, c in enumerate(found))

    prompt = f"""{sources}

Answer the question using only the sources above.
After each fact, cite the source like [1].
If the sources do not contain the answer, say "I don't know based on the documents I have."

Question: {question}"""

    response = client.messages.create(
        model="claude-sonnet-5-5", max_tokens=500,
        messages=[{"role": "user", "content": prompt}],
    )
    return response.content[0].text
```

That is a complete RAG system. Build this version yourself before touching a framework, so
you understand each part. Everything below improves on it.

---

## 9. Reranking

First-pass search is fast but rough. A **reranker** is a slower, more careful model that
reads the question and each candidate chunk *together* and scores how well they match.

The standard two-stage pattern:

```
1. Fast search returns 50 candidates   (cheap, catches most of the good ones)
2. Reranker scores all 50 carefully    (slower, much more accurate)
3. Keep the top 5 for the prompt
```

Analogy: a recruiter skims 500 CVs to pick 50, then a hiring manager reads those 50
properly to pick 5.

Reranking often moves the right chunk from position 12 up to position 1, which is the
difference between a wrong and a right answer.

---

## 10. Rewriting the question

Users ask questions that search poorly. Fix the question before searching.

**Follow-ups.** In a conversation:

```
User: What's the parental leave policy?
Bot:  16 weeks paid.
User: Does it apply to contractors?
```

Searching for "Does it apply to contractors?" finds nothing useful, because "it" means
nothing alone. Have the model rewrite it first: **"Does the parental leave policy apply to
contractors?"**

**Several versions.** One question can be phrased many ways. Generate a few and search
with all of them:

```
Original:  "laptop won't charge"
Variants:  "laptop battery not charging"
           "power adapter not working"
           "computer does not turn on when plugged in"
```

Merge the results as in section 6. This technique is sometimes called RAG-Fusion.

**Splitting.** "Compare our leave policy in the UK and Germany" is two searches: one for
each country.

---

## 11. Grounding, citations and saying "I don't know"

**Grounding** means the answer comes from the supplied sources, not the model's general memory.

- **Ask for citations.** Each claim points to a source. Users can check, and you can spot
  made-up claims automatically.
- **Give a way out.** Tell the model what to say when the sources lack the answer.
  Otherwise it will fill the gap from memory, which is how you get confident wrong answers.

*Without a way out:*
```
Q: What is our policy on bringing pets to the office?
(sources contain nothing about pets)
A: Pets are generally allowed on Fridays with manager approval.     ← invented
```

*With one:*
```
A: I don't know based on the documents I have. The handbook sections I found cover
   leave and remote work, but not pets.
```

Test this on purpose: include questions your documents cannot answer and check that the
system declines.

---

## 12. Advanced patterns

Know these exist; reach for them when the basic version falls short.

- **Agentic retrieval.** Instead of one fixed search, the model gets a search tool and
  decides what to look up, reads the results, and searches again if needed. Good for
  questions that need several hops ("Who manages the person who wrote this policy?").
- **Parent-document retrieval.** Search over small, precise chunks, but hand the model the
  larger section each chunk came from, so it gets the surrounding context.
- **Contextual chunks.** Before embedding, have a model write a one-line note for each
  chunk saying where it sits in the document. This is the "add context at the top" trick
  from section 2, automated.
- **Graph-based retrieval.** Extract people, products and their relationships into a
  network, and search that. Useful when answers depend on connections between things.

---

## 13. Frameworks

LlamaIndex and LangChain package loaders, chunkers, vector-store connections and pipelines.
They save time once you know what each piece does. Use them after your from-scratch version
works, and keep your measurements (below) so you can tell whether a framework's defaults
help or hurt.

---

## Measuring search quality

Do this early. It is the only way to know whether a change helped.

1. Write 50 questions. For each, note which chunk or document holds the answer.
2. Run your search. For each question, check whether the right chunk is in the top 5.
3. **Recall@5** = the share of questions where it was.

*Example:* the right chunk appeared in the top 5 for 38 of 50 questions. Recall@5 = 76%.
You add hybrid search and it becomes 44 of 50 = 88%. Now you know hybrid search helped,
with a number.

When an answer is wrong, check this first. If the right chunk was not retrieved, fix
search. If it was retrieved and the answer is still wrong, fix the prompt.

---

## Free resources

- Original RAG paper: <https://arxiv.org/abs/2005.11401>
- Sentence Transformers: <https://sbert.net/>
- MTEB embedding leaderboard: <https://huggingface.co/spaces/mteb/leaderboard>
- pgvector: <https://github.com/pgvector/pgvector>
- Chroma docs: <https://docs.trychroma.com/>
- Qdrant docs: <https://qdrant.tech/documentation/>
- FAISS: <https://github.com/facebookresearch/faiss>
- `rank_bm25`: <https://github.com/dorianbrown/rank_bm25>
- Pinecone learning centre: <https://www.pinecone.io/learn/>
- Anthropic on contextual retrieval: <https://www.anthropic.com/news/contextual-retrieval>
- LlamaIndex docs: <https://docs.llamaindex.ai/>

## Project 1: a hybrid search engine

- Index at least 500 real documents (Wikipedia articles on a topic, a product manual, your
  own notes).
- Build keyword search, meaning search and merged hybrid search behind one API.
- Write 50 questions, each labelled with the document that should come back.
- Report recall@5 and recall@20 for all three, then again with a reranker.

## Project 2: a deployed RAG application

- Add answering with citations on top of project 1.
- Handle follow-up questions by rewriting them.
- Include 10 questions the documents cannot answer and confirm the system declines.
- **Evaluation step:** score 30 answers for correctness and for whether each claim is
  backed by a cited chunk.
- Deploy it using what you learned in level 0.

## Self-check

1. A user searches for an exact error code and gets a different one. What is missing?
2. Why does a reranker improve results that were already ranked?
3. How do you tell a search failure from an answering failure?
4. What does recall@5 measure, and why check it before answer quality?
5. When is RAG the wrong tool?
