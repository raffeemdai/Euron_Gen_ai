# Class 5 — 30th Nov — FAISS Practical — Class Notes

## Recap
- Previous class (29th Nov) covered vector database **theory**: why natural language must be converted into numeric embeddings, why SQL/NoSQL can't do natural-language similarity search, and RAG (Retrieval-Augmented Generation).
- Today's class is the first **hands-on, end-to-end practical**: install a real vector database, chunk real text data, generate embeddings via the Euron API, store them, and run a similarity search query — all in Python.
- The vector database used today: **FAISS** — **F**acebook **AI** **S**imilarity **S**earch.
- Other vector databases mentioned as upcoming topics: **ChromaDB**, **Qdrant**, **Pinecone**, **Weaviate** — each will get its own dedicated class.

---

## 1. What Is FAISS?
- FAISS = **Facebook AI Similarity Search**, a library (originally built by Meta/Facebook AI) purpose-built for storing vectors and running fast similarity searches over them.
- Conceptually, it plays the same role that a SQL "table" or MongoDB "collection" played in earlier classes — except instead of storing rows or documents, it stores **vectors** (embeddings) and is optimized specifically for "which stored vector is closest to this query vector?" type searches.
- In FAISS, the equivalent of a "table" or "collection" is called an **index**.

> 🔎 **Analogy:** Think of a FAISS index like a specialized filing cabinet built *only* for holding numbered coordinates and instantly telling you "which coordinate is nearest to this one?" — unlike a regular filing cabinet (SQL/NoSQL), it isn't built for keyword lookups or exact-match filters; its entire design is optimized around distance/nearness calculations.

### Installation
```bash
pip install faiss-cpu
```
- Most learners will use the CPU version (`faiss-cpu`) since a GPU isn't required for this scale of data.

### Import and version check
```python
import faiss
faiss.__version__
```

---

## 2. Preparing the Raw Data
- A real, messy, **unstructured** block of text was used: a biography/"founder story" about Sudhanshu Kumar (the instructor), copied as a single large text blob (no predefined structure — just paragraphs).
- This mirrors real-world scenarios: your source data could just as easily come from a PDF, Word document, `.txt` file, or JSON — the key point is you first **read/extract** the raw text, however it's stored.

```python
bio_text = """
Making an Impact
Helping Millions of Students Succeed
Sudhanshu's commitment to affordable education wasn't just a business strategy—it was his life's mission. Over the years, iNeuron has helped over 1.5 million students from 34+ countries, providing them with the skills they need to succeed in today's competitive job market. Many of these students, like Sudhanshu himself, came from disadvantaged backgrounds. They saw iNeuron as a lifeline—an opportunity to rise above their circumstances.

In 2022, iNeuron was acquired by PhysicsWallah in a deal worth ₹250 crore. While this acquisition was a significant milestone, Sudhanshu remained focused on his mission. Even after the acquisition, iNeuron continued to offer some of the most affordable and accessible tech courses in the world.

The Entrepreneur and Teacher: Sudhanshu's Dual Legacy
Sudhanshu's journey isn't just one of entrepreneurial success; it's also a story of dedication to teaching. Throughout his career, he has remained a passionate educator, constantly looking for ways to empower others through knowledge. Whether teaching courses in Big Data, Data Science, or programming, Sudhanshu has always sought to make complex subjects accessible to learners at all levels.

His commitment to affordable education has earned him the respect and admiration of countless students. Many credit Sudhanshu with changing their lives, helping them secure jobs, improve their skills, and break free from the limitations of their backgrounds.

The Journey

Early Life and the Power of Education

Rising Through the Ranks of the Tech World

The Birth of iNeuron: Democratizing Education

The Foundation of Euron: Expanding the Mission

Early Life and the Power of Education
Sudhanshu Kumar's life is a story of triumph over adversity, driven by the belief in the transformative power of education. Born in Jamshedpur, Jharkhand, India, to a family of very modest means, Sudhanshu's early years were marked by financial hardship. His surroundings offered little opportunity, and resources were limited, yet he understood from a young age that education could be his ticket out of poverty.

While many would have been daunted by the lack of support and opportunity, Sudhanshu was relentless in his pursuit of knowledge. He knew that education had the power to change lives, and he was determined to leverage it to create a better future for himself and his family. Despite the numerous challenges along the way, Sudhanshu excelled academically, eventually earning a degree in Computer Science and Engineering (CSE).
"""
```

---

## 3. Why We Chunk the Data Before Embedding

**The key question asked in class:** should the entire text be converted into *one single* embedding, or broken into smaller pieces first?

- If the whole biography became **one single embedding**, then any search query (e.g., "who is Sudhanshu Kumar?") would only ever get back **the entire blob** as the result — there's nothing smaller to compare against, so there's no way to pinpoint the *specific* relevant part of a large document.
- This completely defeats the purpose of a search system. Imagine a real use case: a 50-page or 1,000-page PDF — you clearly don't want one giant embedding for the whole document; you want to search *within* it and get back a small, relevant snippet.
- **The solution: chunking.** Break the large text into many small, overlapping pieces, and create **one embedding per chunk**. This is what actually enables meaningful search.

> 🔎 **Analogy:** Imagine trying to find one sentence in a book by asking "does this book contain something similar to my question?" — you'd only ever get a yes/no about the *entire book*. But if you first cut the book into paragraphs and compare your question against each paragraph individually, you can point to the *exact* paragraph most relevant to your question. Chunking is that "cutting into paragraphs" step.

### Chunk size and overlap
- **Chunk size** — how many words/characters go into each chunk. Class recommendation: somewhere around **500–1,200/1,300** characters generally works well, but it's not a fixed rule — it's tuned experimentally per data set and use case (today's demo used a much smaller **10-word** chunk size, since the sample text itself was short).
- **Overlap** — a small amount of shared content between consecutive chunks, so that context isn't abruptly cut off at chunk boundaries. Class recommendation: roughly **50–100 characters** of overlap for larger chunk sizes (here, an overlap of **2 words** was used for the 10-word chunks).
- This overlap technique is called the **sliding window** concept — each new chunk's starting point "slides forward" by `chunk_size − overlap`, rather than starting exactly where the previous chunk ended, which is what creates the shared overlapping text between chunks.

> 🔎 **Analogy:** Picture reading a long scroll through a small window that only shows a few words at a time. Instead of moving the window forward by its *full* width each time (which could awkwardly cut a sentence in half with no shared context), you slide it forward just a *bit less* than its full width — so the end of one "view" and the start of the next share a few words in common. That shared strip is the overlap.

### The chunking function
```python
def chunk_text(text, chunk_size=10, overlap=2):
    words = text.split()
    chunks = []
    start = 0
    while start < len(words):
        end = min(start + chunk_size, len(words))
        chunk = ' '.join(words[start:end])
        chunks.append(chunk)
        start += chunk_size - overlap
    return chunks
```

**Explanation, step by step:**
- `text.split()` — splits the raw text into a list of individual words (by whitespace).
- `start = 0` — begin at the very first word.
- **The `while` loop** keeps producing chunks as long as `start` hasn't reached the end of the word list yet.
- `end = min(start + chunk_size, len(words))` — the chunk's end point is either `start + chunk_size` words ahead, **or** the very end of the list — whichever comes first (this `min()` prevents trying to read past the last word).
- `chunk = ' '.join(words[start:end])` — slices out the words from `start` to `end` and joins them back into a single string — this is one chunk.
- `chunks.append(chunk)` — saves this chunk into the running list.
- `start += chunk_size - overlap` — moves the starting point forward for the *next* chunk, but by `chunk_size - overlap` words instead of the full `chunk_size` — this is exactly what creates the overlapping words between consecutive chunks (the sliding window).
- Returns the final list of all chunks once `start` has moved past the end of the word list.

### Running it on the bio text
```python
chunks = chunk_text(bio_text)
```
- With the default `chunk_size=10, overlap=2`, the ~500-word biography was broken into **49 chunks**, each roughly 10 words long, each sharing 2 words with the chunk before it.
- Example chunks produced (showing the overlap in action):
  - `"Making an Impact Helping Millions of Students Succeed Sudhanshu's commitment"`
  - `"Sudhanshu's commitment to affordable education wasn't just a business strategy—it"` *(note "Sudhanshu's commitment" repeats from the end of the previous chunk — that's the 2-word overlap)*
  - `"from 34+ countries, providing them with the skills they need"`
  - ...and so on, through 49 total chunks.

**A live debugging note from class:** the instructor initially tried a bigger `chunk_size` (like 500) on this particular short sample text, and it looked like chunking "wasn't working" — everything came back in a single chunk. The reason: the sample text simply didn't have 500 words to begin with, so the function correctly wrapped the entire (short) text into one chunk. There was no bug in the function — the chunk size just needs to be sized appropriately relative to how much text you actually have.

---

## 4. Converting Chunks Into Embeddings (via the Euron API)

### Setup
```python
import requests
import os
import json
import numpy as np
```

```python
API_URL = "https://api.euron.one/api/v1/euri/embeddings"
API_KEY = "euri-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"  # Replace with your actual API key
MODEL_NAME = "text-embedding-3-small"
```
- Same three ingredients needed for **any** API call, reused from earlier classes: a **URL**, an **API key**, and (here) a **model name**.
- `text-embedding-3-small` is the same OpenAI embedding model used in the previous class — it returns a **1,536-dimension** vector for any input text.

```python
header = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {API_KEY}"
}
```
- Standard header for any authenticated JSON API call: declares the payload format and passes the API key as a Bearer token.

### Embedding every chunk in a loop
```python
all_embeddings = []
for i, chunk in enumerate(chunks):
    payload = {
        "model": MODEL_NAME,
        "input": chunk
    }
    response = requests.post(API_URL, headers=header, data=json.dumps(payload))
    data = response.json()
    embeddings = data['data'][0]['embedding']
    all_embeddings.append(embeddings)
```

**Explanation:**
- `enumerate(chunks)` — a built-in Python function that walks through the `chunks` list and also gives you the **index** of each item, so `i` = the position (0, 1, 2, ...) and `chunk` = the actual text at that position.
- For **each individual chunk**, a separate API request is sent — one chunk in, one embedding out. This is the "one by one" loop explicitly emphasized in class: every chunk gets its **own** independent embedding call.
- `data['data'][0]['embedding']` — pulls the actual numeric vector out of the API's JSON response for that one chunk.
- **Key debugging moment from class:** the instructor initially forgot to `.append()` each result inside the loop — this meant only the *last* chunk's embedding ever got kept, overwriting all the previous ones. The fix was simply appending every result to the `all_embeddings` list inside the loop, so nothing gets lost.

### Verifying the result
```python
len(all_embeddings)   # -> 49
len(chunks)            # -> 49
type(all_embeddings)   # -> list
```
- Confirms a **1-to-1 match**: 49 chunks in, 49 embeddings out — nothing missing, nothing duplicated.

---

## 5. Converting Embeddings Into the Format FAISS Needs

- FAISS requires embeddings as a **NumPy array of `float32` values** — a plain Python list of lists won't work directly.

```python
embeddings_array = np.array(all_embeddings, dtype='float32')
```
- This produces a 2D array of **shape `(49, 1536)`** — 49 rows (one per chunk), each with 1,536 columns (the embedding dimension from the model). This is described in class as similar to a spreadsheet: 49 records, each with 1,536 "columns" of numeric data.
- `dtype='float32'` is simply a data-type conversion — nothing conceptually different from any other conversion, just the specific numeric precision FAISS expects internally.

---

## 6. Creating a FAISS Index and Storing the Data

```python
base_index = faiss.IndexFlatL2(1536)
```
**Breaking this down:**
- `faiss.IndexFlatL2(...)` — creates a new FAISS index. Think of this as creating a new empty "database"/"table" specifically for vectors.
- `1536` — the dimension every vector stored in this index **must** match (since that's what the embedding model returns). **All vectors inside one FAISS index must share the same dimension** — mixing dimensions inside the same index isn't allowed.
- **`L2` = Euclidean distance** — the exact same distance formula covered in the previous class (`√[Σ(xᵢ − yᵢ)²]`). This tells FAISS: "whenever a search happens on this index, measure closeness using Euclidean distance."
- **Why specify the distance method (`L2`) at *creation* time, before any searching happens?** Because vector databases build their internal indexing/optimization structures around the chosen distance metric right from the moment data is stored — not just at search time. This is a general pattern across *all* vector databases, not unique to FAISS.
- Other common distance/similarity options (mentioned, to be covered in more depth later): **IP** (inner product — related to cosine similarity) and others used by Pinecone, Qdrant, Weaviate, ChromaDB, etc.

### Adding the data
```python
base_index.add(embeddings_array)
```
- One single line stores **all 49 vectors** into the index at once.
- Emphasized in class: this whole process is really just **two lines of code** — (1) create the index, (2) add the data.

### Persisting the index to disk
```python
faiss.write_index(base_index, "faiss_index.faiss")
```
- Without this step, the index only exists in memory (RAM) and disappears once the notebook/session ends.
- `faiss.write_index(...)` **serializes** the index into a binary file on disk — so it can be reloaded later, or handed to someone else, without needing to redo any of the embedding/indexing work.
- Opening the `.faiss` file directly in a text editor shows unreadable binary/encrypted-looking content — this is expected; it's a serialized binary format meant to be read back only via the FAISS library itself (`faiss.read_index(...)`).

---

## 7. Querying the Index — Semantic Search in Action

### Preparing the query
```python
query_test = "tell me about kumar early life"
```
- This plays the same role as the "customer query" from the previous class's theory discussion.

### A reusable embedding function for queries
```python
def embedding_text(text):
    payload = {
        "model": MODEL_NAME,
        "input": text
    }
    response = requests.post(API_URL, headers=header, data=json.dumps(payload))
    data = response.json()
    embeddings = data['data'][0]['embedding']
    emb = np.array(embeddings, dtype="float32").reshape(1, -1)
    return emb
```
- This function wraps the exact same embedding logic used earlier for the chunks, so it's reusable for **any** new piece of text — in this case, the incoming search query.
- **Critical rule reinforced from class:** the query must be embedded using the **same model** (`text-embedding-3-small`) used to embed the stored chunks. Mixing embedding models between storage and querying would make the comparison meaningless.
- `.reshape(1, -1)` — FAISS's `search()` function expects a 2D array (even for a single query vector), so this reshapes the flat embedding into a `(1, 1536)` shaped array — "1 row, however many columns are needed."

```python
query_test_emb = embedding_text(query_test)
```

### Running the similarity search
```python
base_index.search(query_test_emb, 3)
```
**Explanation:**
- `search(query_vector, k)` — searches the index for the **top-k** nearest vectors to the given query vector. Here, `k=3` — asking for the **3 closest matches**.
- **Returns two arrays:**
  1. The **distances** to each of the top-k matches (smaller = closer/more relevant).
  2. The **indexes** (positions) of those matching chunks inside the original `chunks` list — so you can look up exactly which chunk of text each match corresponds to.

**Class result for `"tell me about kumar early life"`:**
```
(array([[1.0286922, 1.2296525, 1.3058555]], dtype=float32),
 array([[32, 27, 36]]))
```
- Distances: `1.03`, `1.23`, `1.31` (in increasing order — first result is the closest).
- Matching chunk indexes: `32`, `27`, `36`.

### Inspecting the actual matched text
```python
chunks[4]
```
```
"from 34+ countries, providing them with the skills they need"
```
*(Note: in an earlier run in class with a different chunk size, index 4 happened to be the top match — chunk indexes shift depending on the chunk_size/overlap settings used at the time.)*

```python
chunks[8]
```
```
"it to create a better future for himself and his family. Despite the numerous challenges along the way, Sudhanshu excelled academically, eventually earning a degree in Computer Science and Engineering (CSE)."
```

```python
chunks[6]
```
```
"the belief in the transformative power of education. Born in Jamshedpur, Jharkhand, India, to a family of very modest means, Sudhanshu's early years were marked by financial hardship. His surroundings offered little opportunity, and resources were limited, yet he understood from a young age that education could be his ticket"
```

- These retrieved chunks **do** talk about "early life," "born in Jamshedpur," "financial hardship," "family of modest means" — directly relevant to the query "tell me about kumar early life," even though the query never used those exact words. This demonstrates **semantic** (meaning-based) matching rather than exact keyword matching.
- **Robustness to typos and rewording:** the instructor also tried the query with typos and reduced wording (e.g., removing "kumar" entirely) — FAISS still returned relevant, similar chunks, just with slightly different rankings/indexes. This is expected: since matching happens on the *meaning* captured by the embedding, not on exact string matching, small wording changes shift results slightly but don't break the search entirely.
- **Changing chunk size changes results:** re-running the whole pipeline with a different `chunk_size`/`overlap` combination naturally produces a different set of chunks (and a different total chunk count — from a handful up to 49 in this session), which shifts which chunk indexes come back as the "top 3" for the same query. Chunking strategy is described as a tunable **hyperparameter** — there's no single universally correct chunk size; it depends on the data and the use case, and is set through experimentation.

---

## 8. Key Q&A Points from Class
- **"Isn't checking too many chunk-size variants futile?"** No — chunk size and overlap materially affect search quality, and tuning them (along with model choice and distance/similarity metric) is a normal, expected part of building a good retrieval pipeline. This will be explored further with other databases (cosine similarity, hybrid search, evaluation).
- **Does the embedding function care about HTML tags/punctuation in the text?** No — the underlying embedding model (OpenAI's, in this case) has been trained on huge amounts of real-world text and can handle punctuation/tags/etc. just fine. However, if a *cleaner*, more polished dataset is desired for your own reasons, pre-processing/cleaning before chunking is still a reasonable step — that's a data-quality choice, not a hard technical requirement.
- **Do different vector databases (Chroma, Pinecone, etc.) use the same functions/API?** No — each library has its own API/function names, but the **underlying concept is the same** across all of them: they're all designed to store vectors and perform similarity/nearest-neighbor search, just with different implementations, defaults, and extra features. A direct comparison across databases (strengths/weaknesses/use cases) is planned for upcoming classes.
- **More parameters ≠ a better neural network/model** (a tangential question asked in class): a model's *architecture* matters far more than raw parameter count. This is why the 2017 "Attention Is All You Need" paper (the origin of the transformer architecture) was such a turning point — before it, larger neural networks existed, but the architecture wasn't capable of what transformers unlocked.

---

## Key Takeaways
- **FAISS** = a vector database/library (from Meta/Facebook AI) purpose-built for fast similarity search over embeddings; its storage unit is called an **index** (comparable to a SQL table or Mongo collection).
- **Never embed an entire large document as one single vector** — always **chunk** it first (with a sensible chunk size + overlap) so search can pinpoint specific relevant sections, not just return the whole document.
- **Chunking = splitting text into overlapping pieces (sliding window)**; overlap preserves context across chunk boundaries.
- The exact same 3-part API pattern (URL + API key + model name) from earlier classes applies to generating embeddings here — just applied in a loop, once per chunk.
- Embeddings must be converted to a NumPy `float32` array before FAISS can store them; all vectors in a single FAISS index must share the same dimension.
- **`IndexFlatL2`** creates an index that uses **Euclidean distance (L2)** for similarity search — the distance metric must be chosen at index-creation time, not at search time.
- **The same embedding model used to store data must be used to embed the search query** — this is a hard rule, not a suggestion.
- `index.search(query_vector, k)` returns the **top-k nearest chunks** (as distances + indexes) to a query — this is the practical mechanism that makes natural-language, meaning-based search possible over your own private data.

---

> ⚠️ **Security note:** The notebook code shared in class (`faiss.ipynb`) contains a live Euron API key hardcoded directly in the source (`API_KEY = "euri-..."`). As with earlier classes, treat this as a secret if it's your own key — avoid hardcoding it in files you share, rotate it if already exposed, and prefer loading it from an environment variable (e.g., via `python-dotenv`) instead. (Note: the instructor mentioned deactivating this particular key at the end of the live session.)
