# Class 4 — 29th Nov — Vector DB (Introduction) — Class Notes

## Recap & Context
- Previous classes covered **SQL** (NeonDB/Postgres) and **NoSQL** (MongoDB) databases, plus how to expose database operations as FastAPI endpoints.
- Today's topic: **Vector Databases** — why they exist, what problem they solve, and a first hands-on look at embeddings.
- This is described as one of the most important topics in the whole course — almost every real-world Generative AI application depends on vector databases and embeddings, so it gets a multi-class deep dive (today is just the conceptual foundation).

---

## 1. Why Do We Need a Vector Database? (The Core Problem)

### The building blocks: everything is a neural network
- All LLMs (ChatGPT, Gemini, LLaMA, Mistral, DeepSeek, etc.) are **generative models** — you give them a prompt (natural language), they generate an output.
- Under the hood, these models are **neural networks** (specifically, **transformer** architectures). A neural network is fundamentally a mathematical/computational system.
- **The core problem:** computers don't understand natural language (English, Hindi, Tamil, etc.) — they only understand numbers, and at the lowest level, binary (0s and 1s).
- So before any sentence can be processed by a neural network, it must first be **converted into a numerical representation**.

> 🔎 **Analogy:** Think of a neural network as a calculator that only accepts numbers as input. If you hand it the sentence "My name is Sudhanshu," it has no idea what to do with it — you first have to translate every word into a number it can compute with.

### Vectors: what "numerical representation" actually means
- This numerical representation is called a **vector**, and a vector has a certain number of **dimensions** (coordinates).
- **2D example:** A point on a graph needs two coordinates — an X value and a Y value — to be located.
- **3D example:** A point in physical space (like where you're sitting in a room) needs three coordinates — X, Y, and Z.
- **Higher dimensions:** You can't visualize a 100-dimension or 1,536-dimension point, but mathematically it works exactly the same way — it's just a list of that many numbers, each one a "coordinate" in a very high-dimensional space.
- **Vector = magnitude + direction.** A single number like "1 meter" is just a **scalar** (magnitude only, no direction). A vector needs coordinates that together define both a value *and* a position/direction in space.

> 🔎 **Analogy:** Giving someone directions with just "one" (one what? one step? one mile?) is meaningless on its own — that's a scalar. But saying "go 3 meters east, 4 meters north" pins down an exact point — that's a vector.

### Why higher dimensions matter for language
- A sentence isn't just a string of words — it encodes grammar, word order, relationships between nouns/verbs/adjectives, semantic meaning, and syntactic structure all at once.
- To capture *all* of that richness, you need many coordinates — hence why real embedding models produce vectors with hundreds or thousands of dimensions (in this class, 1,536 dimensions, from OpenAI's `text-embedding-3-small` model).
- **Trade-off:** higher dimensions = richer representation, but also more computational overhead. This is a deliberate design choice made when building/selecting an embedding model.
- **Embedding vs. Vector — same thing, different name:** "Vector" is the general mathematical term (magnitude + direction). "**Embedding**" is what we call it specifically when a piece of natural language (or an image, video, etc.) has been converted into that numerical/vector form.

---

## 2. Why Not Just Use SQL or MongoDB? (The Actual Problem Being Solved)

This was explored as a live discussion in class — here's the reasoning, step by step:

1. **You can already chat with ChatGPT/Gemini without storing anything.** So what's the point of storing embeddings in a database at all?
2. **The catch: LLMs know nothing about *your* private data.** Ask ChatGPT "What is my company's HR leave policy?" — it has no access to your internal HR documents, so it can't answer. Same for an insurance company's chatbot being asked about a specific policy document, or asking about a specific organization's course catalog.
3. **So the fix seems simple: store your private documents in a database, and let the system search them when a question comes in.** This is exactly what SQL and NoSQL databases already do — and we've been doing this for decades.
4. **But here's the real problem:** to query a SQL database, you must write **SQL syntax**. To query MongoDB, you write **MongoDB query syntax**. Every database has its own defined query language.
   - With LLMs, though, people don't write structured queries — they ask questions in their own **natural language**, with their own wording, grammar (even with mistakes), and phrasing. Two different people asking about the same policy will phrase their question completely differently.
5. **So the real requirement is:** a database that can take a question phrased in plain natural language, compare it against everything stored inside it, and return whichever stored piece of information is *closest in meaning* — without requiring any predefined query syntax.

> 🔎 **Analogy:** SQL/MongoDB are like a librarian who only responds to requests written in strict call-number format ("Fetch Dewey 641.5, Shelf 3"). A vector database is like a librarian who understands "hey, do you have anything about easy dinner recipes?" and finds the closest matching book — even though you never typed the exact title.

This is exactly the gap a **vector database** fills: it stores data as embeddings and retrieves the *closest matching* record to a natural-language query — not an exact syntactic match.

---

## 3. How "Closest Match" Actually Works — Distance Between Vectors

### The 2D coordinate example from class
Given four points on a simple 2D plane:
- **A** = (5, 6) — represents a **customer query**
- **B** = (1, 2)
- **C** = (0, 2)
- **D** = (1, 4)

Using the **Euclidean distance** formula (the same one from school geometry):

```
distance = √[(x1 − x2)² + (y1 − y2)²]
```

- Distance(A, D) = √[(5−1)² + (6−4)²] = √[16 + 4] = √20
- Distance(A, C) = √[(5−0)² + (6−2)²] = √[25 + 16] = √41

Since √20 < √41, **D is closer to A than C is**. If B, C, and D represent sentences stored in a database and A is the user's query, the system would return **D** as the best match — because it has the *smallest distance* to the query.

> 🔎 **Analogy:** Imagine dropping pins on a map for every FAQ answer you have, then dropping one more pin for the customer's question. Whichever existing pin is physically closest to the new one is your best-matching answer.

- This is the **same underlying math**, just extended from 2 dimensions to hundreds/thousands of dimensions when working with real sentence embeddings.
- Euclidean distance is just one method mentioned — others include **Manhattan distance**, **cosine similarity**, and **IP (inner product/dot product)**. Different similarity/distance methods suit different use cases; this will be covered in more depth in a future class.
- The number of "closest" results returned is often called **top-k** (e.g., top 3, top 5) — how many nearest matches to retrieve, depending on configuration.

---

## 4. Do We Still Need LLMs If Vector DB Can Already Retrieve the Answer?

- A vector DB search returns the **raw closest chunk of stored text** — but this might be a rough, broken, or overly terse fragment. It's not guaranteed to read like a polished, human-friendly answer.
- **This is exactly where the LLM comes back in:** it takes (a) the user's original query and (b) the raw data retrieved from the vector DB, and **combines** them to generate a smooth, well-phrased, final response.
- **Why pass in the original query too, and not just the retrieved data?** Because without knowing *what was asked*, the LLM (or a human) can't properly interpret or contextualize the raw answer.

> 🔎 **Analogy used in class:** If someone tells you a very specific detail about their night — "I was working on a client project, some white-labeling work, and a new offering called work-and-learn" — without knowing the *question* that prompted this answer, you have no way to make sense of it or summarize why it was said. Context (the query) is essential to interpreting the retrieved content.

### This combined pattern is called RAG
**RAG = Retrieval-Augmented Generation**
- **Retrieve** → get relevant data from the vector database based on the user's query.
- **Augment** → combine that retrieved data with the original user query.
- **Generate** → feed both into an LLM, which produces the final, polished, human-readable answer.

```
User query ──┬──────────────────────► Vector DB ──► retrieved data ──┐
             │                                                       │
             └───────────────────────────────────────────────────►  LLM ──► Final Answer
```

- RAG is one of the most widely used patterns across the industry for building applications that answer questions from an organization's **own private data** (HR policies, insurance documents, internal course catalogs, product manuals, etc.).
- A more advanced variant, **Agentic RAG**, will be covered a few months into the course.
- Real example shown in class: the "Euron AI" WhatsApp assistant answering "what is the latest course launched?" — a question no general-purpose LLM could answer correctly without access to Euron's private, up-to-date data.

---

## 5. Why Not Just Upload Everything Into an LLM's Chat Window?
- LLMs (ChatGPT, Claude, etc.) do let you upload documents and ask questions about them — that also technically works for small amounts of data.
- **But real organizational data is far too large.** You can't upload gigabytes, terabytes, or petabytes of documents into a single chat prompt — there are strict upload/token limits.
- You also can't expect every customer to manually upload your entire policy document set every time they open a chat.
- **This is why you need a proper database layer (vector DB) sitting behind the LLM** — it can hold arbitrarily large amounts of data efficiently, retrieve only the small relevant piece needed for a specific query, and hand just that piece to the LLM. This keeps the system fast and scalable, no matter how much total data you have stored.

---

## 6. Practical Demo — Generating Embeddings via the Euron API

The instructor generated embeddings using Euron's hosted `text-embedding-3-small` (OpenAI) model, then measured distances between them using **NumPy**, in a Jupyter notebook (`test.ipynb`).

### Step 1 — Install dependencies
```python
pip install requests
pip install numpy
```

### Step 2 — Generate an embedding for a sentence ("My name is Sudhanshu")
```python
import requests

url = "https://api.euron.one/api/v1/euri/embeddings"
token = "euri-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

payload = {
    "input": "my name is sudhanshu",
    "model": "text-embedding-3-small"
}

response = requests.post(url, json=payload, headers={"Authorization": f"Bearer {token}"})

print(response.json())
```
**What this does:**
- Sends the sentence `"my name is sudhanshu"` to the embeddings endpoint.
- Specifies the model to use: `text-embedding-3-small` (one of OpenAI's most popular embedding models, made available through the Euron API).
- The API responds with a JSON object containing the embedding — a long list of floating-point numbers — plus token-usage metadata.

### Step 3 — Extract and inspect the embedding
```python
result = response.json()
len(result['data'][0]['embedding'])   # -> 1536
a = result['data'][0]['embedding']
```
- `result['data'][0]['embedding']` pulls out just the numeric vector.
- **`len(...)` = 1536** — confirming this model represents every sentence as a **1,536-dimension vector**, regardless of how short or long the input sentence is. This dimension count is fixed by the model you choose, not something you control per-request.

### Step 4 — Generate embeddings for two more sentences (B and C)
```python
import requests

url = "https://api.euron.one/api/v1/euri/embeddings"
token = "euri-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

payload = {
    "input": "sudhanshu use to work for euron",
    "model": "text-embedding-3-small"
}
response = requests.post(url, json=payload, headers={"Authorization": f"Bearer {token}"})
b = response.json()['data'][0]['embedding']
```
```python
import requests

url = "https://api.euron.one/api/v1/euri/embeddings"
token = "euri-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

payload = {
    "input": "rakesh a gen ai batch student",
    "model": "text-embedding-3-small"
}
response = requests.post(url, json=payload, headers={"Authorization": f"Bearer {token}"})
c = response.json()['data'][0]['embedding']
```
- **a** = embedding for *"my name is sudhanshu"*
- **b** = embedding for *"sudhanshu use to work for euron"*
- **c** = embedding for *"rakesh a gen ai batch student"*

### Step 5 — Generate an embedding for the "query" sentence
```python
import requests

url = "https://api.euron.one/api/v1/euri/embeddings"
token = "euri-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

payload = {
    "input": "what sudhanshu does",
    "model": "text-embedding-3-small"
}
response = requests.post(url, json=payload, headers={"Authorization": f"Bearer {token}"})
inp = response.json()['data'][0]['embedding']
```
- **inp** = embedding for the incoming query: *"what sudhanshu does"* — this plays the same role as the customer query in the earlier "customer support" example.

### Step 6 — Convert to NumPy arrays and measure distance
```python
import numpy as np

a = np.array(a)
b = np.array(b)
c = np.array(c)
inp = np.array(inp)

dist_inp_a = np.linalg.norm(inp - a)
dist_inp_b = np.linalg.norm(inp - b)
dist_inp_c = np.linalg.norm(inp - c)

dist_inp_a, dist_inp_b, dist_inp_c
```
**What this does:**
- The raw embeddings come back as plain Python **lists** — `np.array(...)` converts each one into a NumPy array so vector math (subtraction, norm/magnitude) can be performed efficiently.
- `np.linalg.norm(inp - a)` computes the **Euclidean distance** between two 1,536-dimension vectors — this is the exact same formula from the 2D example earlier (`√[Σ(xᵢ − yᵢ)²]`), just extended to 1,536 dimensions instead of 2.

### Result from class
```
dist_inp_a = 0.7505   (query vs. "my name is sudhanshu")
dist_inp_b = 0.8504   (query vs. "sudhanshu use to work for euron")
dist_inp_c = 1.1831   (query vs. "rakesh a gen ai batch student")
```
- **Smallest distance → sentence A** ("my name is sudhanshu") is mathematically the "closest" to the query "what sudhanshu does."
- The instructor noted this result is a *bit* counter-intuitive to a human reader (many would expect sentence B, about what Sudhanshu actually *does for work*, to be the closest match) — and explained why:
  - The test sentences were **very short** (just a few words each), giving the model little context to work with. Real-world pipelines typically use much larger text **chunks** with **overlap** between them, which produces richer, more accurate embeddings.
  - **Euclidean distance** is described as a simple/"novice" similarity method — cosine similarity, hybrid search, or other techniques often give results that better match human intuition, and will be covered later.
- **Key takeaway from the demo:** this small NumPy exercise is a hands-on proof of the exact theory covered earlier — no vector database software was even used yet; it's just raw embeddings + basic distance math, showing what a vector DB does under the hood.

---

## 7. Choosing an Embedding Model
- The `text-embedding-3-small` model used here is one of OpenAI's most advanced embedding models — but it is **not the only option**.
- **Hugging Face** (a major open-source model repository) alone lists **3,000+ embedding models** — e.g., Qwen embeddings, Google's Gemma embeddings, Jina embeddings — each with different parameter sizes (from a few hundred million to several billion parameters) and different **output dimensions** (e.g., 768 vs. 1,536).
- **How to choose a model — the main criterion:** check what data the model was trained on, and match it to your domain.
  - Example: for an HR-focused chatbot, prefer a model that has seen HR-style language during training.
  - Example: for a clinical/healthcare use case, prefer a model trained on medical/clinical text.
- **Dimension size is not a simple "bigger is better" choice** — very high dimensions increase representational detail but also increase computational cost for similarity search. It should match the model's training data and your use case, not be picked purely by size.
- **Critical rule — consistency:** whichever embedding model you use to store your data, you **must use that exact same model** to embed incoming queries against that same data/database. Mixing embedding models within one pipeline breaks the comparison (the numbers wouldn't be on the same "scale" or meaning-space).
- You **can** use different embedding models for different, *separate* databases/pipelines (e.g., one model for an HR knowledge base, a different model for a product knowledge base) — as long as each individual pipeline stays internally consistent.
- Any embedding model (OpenAI, Google, open-source, etc.) can be paired with **any** vector database (Qdrant, Pinecone, Chroma, FAISS, pgvector/Postgres, MongoDB Atlas Vector Search, etc.) — model choice and database choice are independent decisions.

---

## 8. Q&A Highlights from Class
- **Can embeddings from different models be stored in the same vector DB?** Technically yes, but for a single querying pipeline, input and output embeddings must come from the same model for the comparison to be meaningful.
- **Can we convert existing SQL/RDBMS data into vector DB data?** Yes — but there's no universal rule for *what* to convert or *how*; it depends entirely on the specific problem being solved (which columns/rows matter, why, what the use case needs).
- **Should image/video data use higher-dimension embeddings?** Generally yes — images (and video, which is just a sequence of image frames) carry more complex information than plain text, so higher-dimension embeddings are typically used, often via **multimodal models**.
- **How do you handle a vector DB when source documents (e.g., HR/insurance PDFs) get updated?** Same as any database — you perform insert/update/delete operations on the relevant records/index entries as the underlying documents change. This will be covered hands-on in an upcoming class.
- **Security of vector databases:** depends on multiple factors — primarily *where* it's hosted (e.g., an EC2 instance lets you control traffic rules, add certificates, add an authentication layer, etc.), same as securing any other hosted service.

---

## Key Takeaways
- Neural networks (and LLMs) only understand numbers — natural language must first be converted into a numerical **vector**, called an **embedding**, before any model can process it.
- Higher-dimension vectors capture more relationships/meaning in language, at the cost of more computation.
- SQL/NoSQL databases require structured query syntax and return **exact** matches — they can't handle open-ended natural-language questions well.
- **Vector databases** store data as embeddings and retrieve the **closest matching** record to a query using distance/similarity math (Euclidean distance, cosine similarity, etc.) — this is what makes natural-language search over private data possible.
- **RAG (Retrieval-Augmented Generation)** = Vector DB retrieval + LLM generation, combined — this is the standard pattern for building chatbots/assistants that can answer questions using an organization's own private data.
- Embedding model choice matters — pick a model trained on data similar to your domain, and **always use the same model for both storing and querying** within one pipeline.

---

> ⚠️ **Security note:** The notebook code shared in class (`test.ipynb`) contains a live Euron API token hardcoded directly in the source (`token = "euri-..."`). As with the database credentials in earlier classes, if this is your own token, treat it as a secret: avoid hardcoding it in files you share, rotate it if it's already been exposed, and load it from an environment variable (e.g., via `python-dotenv`) instead.
