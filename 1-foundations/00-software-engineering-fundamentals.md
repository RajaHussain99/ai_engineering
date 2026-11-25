# Level 0: Software engineering fundamentals

**Phase 1: Foundations** · Next: [Level 1](01-llm-101.md)

**Goal:** build, debug and deploy an ordinary web service before adding any AI to it.

**Skip this level if** you have shipped a backend service that other people used.

## Why this comes first

An AI product is about 90% ordinary software. The model call is one line. Everything around
it is the work: receiving a request, checking the input, saving data, handling errors, and
running on a server that stays up. When AI products break, the cause is usually something
plain, like a missing timeout or a password committed to the repository. Learn the plain
parts first and the AI parts become easy to add.

---

## 1. Python for this kind of work

Python is the main language of AI engineering because nearly every model provider and
library supports it first. You do not need all of Python. You need the parts below.

**Virtual environments.** Each project gets its own private folder of installed libraries,
so two projects that need different versions do not clash.

```bash
python -m venv .venv          # create the private folder
source .venv/bin/activate     # start using it
pip install fastapi           # installs into .venv only
pip freeze > requirements.txt # record exact versions
```

**Type hints.** Small labels that say what kind of value a variable holds. Python does not
enforce them, but your editor uses them to catch mistakes, and AI libraries use them to
describe data to the model.

```python
def total_price(prices: list[float], tax_rate: float) -> float:
    return sum(prices) * (1 + tax_rate)
```

**Exceptions.** When something goes wrong, Python raises an error. You catch the errors you
can handle and let the rest stop the program loudly.

```python
try:
    amount = float(user_input)
except ValueError:
    print("Please enter a number.")
```

**Async.** A way to wait for slow things (network calls, model responses) without freezing
the whole program. Think of a waiter who takes the next table's order while the kitchen
cooks, instead of standing at the kitchen door.

```python
import asyncio, httpx

async def fetch(url: str) -> int:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)   # "await" = let others work while I wait
        return response.status_code

async def main():
    # both requests run at the same time
    results = await asyncio.gather(fetch("https://example.com"), fetch("https://example.org"))
    print(results)

asyncio.run(main())
```

This matters later because a model call can take 10 seconds. Without async, one slow
request blocks everyone else.

**Testing.** A test is a small piece of code that checks another piece of code.

```python
# test_prices.py  (run with: pytest)
from prices import total_price

def test_total_price_adds_tax():
    assert total_price([10.0, 20.0], tax_rate=0.1) == 33.0
```

Tests let you change code without fear. If you break something, a test fails and tells you.

---

## 2. The web: HTTP, REST and JSON

Every AI model you will use is reached over the web. Your program sends a request and gets
a response. This exchange follows a set of rules called HTTP.

**A request has four parts:**

| Part | What it is | Example |
|---|---|---|
| Method | The kind of action | `GET` (read), `POST` (create), `PUT` (replace), `DELETE` |
| URL | Where to send it | `https://api.example.com/books/42` |
| Headers | Extra information | `Authorization: Bearer my-secret-key` |
| Body | The data, usually JSON | `{"title": "Dune"}` |

**A response has a status code** that tells you what happened:

| Code | Meaning | What your program should do |
|---|---|---|
| 200, 201 | Worked | Carry on |
| 400 | You sent something invalid | Fix the request; retrying will not help |
| 401 | Missing or wrong key | Check your credentials |
| 403 | Key is fine, but you are not allowed | Ask for access |
| 404 | Not found | Check the URL |
| 429 | Too many requests | Wait, then retry |
| 500, 503 | The server failed | Wait, then retry |

Knowing which errors to retry (429 and 5xx) and which not to (4xx) is one of the most
useful things in this whole level.

**JSON** is the text format used to send data. It looks like a Python dictionary.

```json
{ "title": "Dune", "pages": 412, "tags": ["sci-fi", "classic"], "read": false }
```

**REST** is a convention for organising URLs around things ("resources"):

```
GET    /books        list all books
POST   /books        add a book
GET    /books/42     get book 42
DELETE /books/42     remove book 42
```

**Try it:** `curl https://api.github.com/users/octocat` in a terminal sends a GET request
and prints the JSON response.

---

## 3. Building an API with FastAPI

An API is a program that other programs talk to. FastAPI is a Python library that makes
one with very little code.

```python
# main.py   (run with: uvicorn main:app --reload)
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()
books: dict[int, dict] = {}

class Book(BaseModel):          # describes what a valid book looks like
    title: str
    pages: int

@app.post("/books", status_code=201)
def add_book(book: Book):
    book_id = len(books) + 1
    books[book_id] = book.model_dump()
    return {"id": book_id, **books[book_id]}

@app.get("/books/{book_id}")
def get_book(book_id: int):
    if book_id not in books:
        raise HTTPException(status_code=404, detail="Book not found")
    return books[book_id]
```

Three things to notice:

- **Validation is automatic.** If someone sends `{"title": "Dune", "pages": "many"}`,
  FastAPI rejects it with a clear 422 error, because `pages` must be a number. The `Book`
  class does this. It comes from a library called Pydantic, which you will use again in
  level 2 to check what a model returns.
- **Documentation is automatic.** Open `http://localhost:8000/docs` and you get a page
  where you can try every endpoint.
- **Errors are explicit.** A missing book returns a proper 404, not a crash.

---

## 4. Data and persistence

The example above keeps books in memory, so they vanish when the program restarts. A
database keeps them on disk.

**SQL** is the language for talking to most databases. Data lives in tables, like
spreadsheets with strict columns.

```sql
CREATE TABLE books (
    id     INTEGER PRIMARY KEY,
    title  TEXT NOT NULL,
    pages  INTEGER
);

INSERT INTO books (title, pages) VALUES ('Dune', 412);
SELECT title FROM books WHERE pages > 300;
```

**SQLite** is a database stored in a single file. It needs no setup and ships with Python.

```python
import sqlite3
db = sqlite3.connect("library.db")
db.execute("INSERT INTO books (title, pages) VALUES (?, ?)", ("Dune", 412))
db.commit()
```

Always pass values with `?` placeholders as shown. Building SQL by gluing strings together
lets an attacker inject their own commands.

Terms worth knowing:

- **Index:** like the index in a book. It makes lookups fast on a chosen column.
- **Transaction:** a group of changes that either all happen or none do. Moving money
  between accounts is the classic case: you must never subtract without adding.
- **Migration:** a small script that changes the database structure (adds a column, say).
  You keep migrations in your repository so every copy of the database can be brought to
  the same shape.

When more than one program needs the database at once, move from SQLite to PostgreSQL.

---

## 5. Git and collaboration

Git records every version of your code, so you can see what changed, undo mistakes, and
work alongside other people.

```bash
git init                         # start tracking this folder
git add main.py                  # choose what to save
git commit -m "Add book endpoints"   # save a snapshot with a note
git switch -c add-search         # make a branch: a safe side-copy to work on
# ...make changes, commit them...
git switch main
git merge add-search             # bring the work back
git push                         # upload to GitHub
```

- **Branch:** a separate line of work. If the experiment fails, delete the branch and
  nothing else is affected.
- **Pull request:** a proposal on GitHub to merge your branch. Other people read the
  changes and comment before it goes in.
- **`.gitignore`:** a list of files Git must never save.

```
# .gitignore
.venv/
.env
*.db
```

**The most important rule:** never commit passwords or API keys. Git keeps history
forever, so deleting the file later does not remove the secret. Put keys in a `.env` file,
list `.env` in `.gitignore`, and read them from the environment:

```python
import os
api_key = os.environ["MY_API_KEY"]
```

---

## 6. AI-assisted development

Coding assistants can write, explain and fix code. Used well they make you faster. Used
carelessly they produce code you cannot maintain.

Good uses:

- "Explain what this function does, line by line."
- "Write pytest tests for `total_price`, including empty lists and negative numbers."
- "This test fails with this error. What is the likely cause?"

Habits that keep you safe:

- Read every line before you accept it. You are the author of whatever you commit.
- Ask for small changes, and run the tests after each one.
- If you could not explain the code to a colleague, do not merge it.

---

## 7. Containers and deployment

"It works on my machine" is a real problem: your laptop has a particular Python version and
particular libraries, and the server does not. A **container** packages your program with
everything it needs, so it runs the same everywhere. Docker is the usual tool.

```dockerfile
# Dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```bash
docker build -t library-api .                         # build the package
docker run -p 8000:8000 -e MY_API_KEY=abc library-api # run it, passing a setting
```

**Deployment** means running that container on a computer that is always on and reachable
from the internet. Many hosts will run a container for you from a GitHub repository, some
with a free tier.

Add a health-check endpoint so the host can tell whether your service is alive:

```python
@app.get("/health")
def health():
    return {"status": "ok"}
```

---

## Free resources

- Python tutorial: <https://docs.python.org/3/tutorial/>
- HTTP guide (MDN): <https://developer.mozilla.org/en-US/docs/Web/HTTP>
- FastAPI tutorial: <https://fastapi.tiangolo.com/tutorial/>
- Pydantic docs: <https://docs.pydantic.dev/>
- `sqlite3`: <https://docs.python.org/3/library/sqlite3.html>
- Pro Git (free book): <https://git-scm.com/book>
- Docker getting started: <https://docs.docker.com/get-started/>
- The Twelve-Factor App: <https://12factor.net/>

## Project: a REST service with no AI in it

Build a small API for something you would use, such as a reading list or expense log.

- Create, read, update and delete endpoints with validated input.
- Data stored in SQLite or PostgreSQL, with one migration.
- At least ten tests, run automatically on every push.
- Logs and a `/health` endpoint.
- A `Dockerfile`, and the service deployed at a public URL.
- A README that lets a stranger run it in under five minutes.

## Self-check

1. What is the difference between a 401, a 403 and a 429, and what should a client do on each?
2. Why does slow, blocking code inside an `async` function hurt every other request?
3. What happens to your data if the container restarts? How do you know?
4. How would you undo a bad deploy?
5. Where do your secrets live, and who can read them?
