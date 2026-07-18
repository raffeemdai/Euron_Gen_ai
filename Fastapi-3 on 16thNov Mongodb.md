# FastAPI - 3 (16th Nov) — Class Notes: FastAPI with MongoDB

**Reference code (GitHub):** https://github.com/euronone/eurno_mongo_api/blob/main/main.py

---

## Recap
- Previous class (15th Nov) covered connecting FastAPI to a **SQL** database (NeonDB/Postgres) using `psycopg2`.
- Today's goal: do the same set of operations, but with a **NoSQL** database — **MongoDB** — and then actually **deploy** the API to the cloud (not just tunnel it with ngrok).

---

## 1. What is MongoDB?
- MongoDB is a **NoSQL** database — "NoSQL" means *not only SQL*.
- Besides SQL and NoSQL, the industry also uses vector databases, graph databases, etc. Databases + APIs are described as the two "backbones" of any real-time application.
- Key structural difference from SQL:
  - SQL: **table** with a fixed schema (columns must be defined ahead of time — e.g., yesterday's `student` table with `id`, `name`, `age`).
  - MongoDB: **collection** — roughly equivalent to a table, but **schema-less**. Data is stored as flexible key-value documents, so no schema needs to be pre-defined at the database level (validation instead happens at the API layer via Pydantic).

---

## 2. MongoDB Atlas Setup
1. Go to **mongodb.com** → sign in (Google sign-in works for a quick account).
2. Click **Create Cluster** → choose the **free tier** (M0 / free/shared instance — no card required for the free tier; a paid "flex"/dedicated tier does require a card).
3. Pick a cloud provider (AWS/Azure/GCP) and a region close to you (e.g., Mumbai).
4. Click **Create Deployment** — cluster provisioning takes a few minutes.
5. Once ready, click **Connect** on the cluster:
   - Choose **Driver** (not Compass/Shell/VS Code) → select **Python** as the driver language.
   - Copy the **connection string** (`mongodb+srv://<username>:<password>@...`).
6. Under **Security → Database Access**, create a **database user** (username + password). Assign it a role:
   - **Admin** — full access.
   - **Read/write** — can insert/update/delete/read.
   - **Read-only** — can only fetch data.
7. Substitute your created username/password into the connection string in place of the placeholders.
8. Under the cluster, click **Browse Collections** → **Create Database**:
   - Database name (example used in class): `euron`
   - Collection name (example used in class): `euron_coll`
   - Collection = the NoSQL analogue of a SQL table, but without a fixed schema.

> **Why expose this as an API instead of sharing the DB directly?** Anyone who needs to insert/update/read data just calls your API endpoint — they never see your MongoDB credentials or connection string. This is exactly how backend teams work: the backend exposes APIs; consumers don't need to know anything about the underlying database.

---

## 3. Project Setup (Local)

### Folder & files
- New project folder, e.g. `mongo db api`.
- Main file: `main.py` (name can be anything, as always — just update the `uvicorn` run command to match).

### `requirements.txt`
Instead of installing libraries one at a time, list them all in one file and install in a single command:
```
fastapi
uvicorn
motor
python-dotenv
pymongo[srv]
```
Install everything in one go:
```bash
pip install -r requirements.txt
```

### Virtual environment (best practice for every project)
```bash
python --version              # check available Python version
python -m venv myenv          # create a virtual environment folder called "myenv"
myenv\Scripts\activate        # activate it (Windows) — folder/Scripts/activate
```
- A virtual environment is just a folder holding only the libraries needed for *this* project — avoids bloating your global Python install and avoids version conflicts across projects.
- Once activated, `pip install -r requirements.txt` installs dependencies only inside that environment.
- Standard practice: **create a new virtual environment for every new project.**

### `.env` file — storing secrets properly
- Create a file named **`.env`** to hold secrets (API keys, DB credentials) instead of hardcoding them in your source file (unlike yesterday's class, where the Postgres URL was hardcoded directly in `test.py` — acknowledged in class as *not* the ideal/standard approach).
  ```
  MONGO_URI="mongodb+srv://<user>:<password>@<cluster-url>/?retryWrites=true&w=majority"
  ```
- When later pushing code to GitHub, `.env` should **never** be pushed (see `.gitignore` section below) — otherwise anyone with repo access could see your DB credentials and get full access to your cluster.

---

## 4. The Code (`main.py`)

> Full working version, as pushed to GitHub: https://github.com/euronone/eurno_mongo_api/blob/main/main.py

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from motor.motor_asyncio import AsyncIOMotorClient
from bson import ObjectId
import os
from dotenv import load_dotenv

load_dotenv()
MONGO_URI = os.getenv("MONGO_URI")
client = AsyncIOMotorClient(MONGO_URI)
db = client["euron"]
euron_data = db["euron_coll"]

app = FastAPI()


class eurondata(BaseModel):
    name: str
    phone: int
    city: str
    course: str


@app.post("/euron/insert")
async def euron_data_insert_helper(data: eurondata):
    result = await euron_data.insert_one(data.dict())
    return str(result.inserted_id)


def euron_helper(doc):
    doc["id"] = str(doc["_id"])
    del doc["_id"]
    return doc


@app.get("/euron/getdata")
async def get_euron_data():
    iterms = []
    cursor = euron_data.find({})
    async for document in cursor:
        iterms.append(euron_helper(document))
    return iterms


@app.get("/euron/showdata")
async def show_euron_data():
    iterms = []
    cursor = euron_data.find({})
    async for document in cursor:
        iterms.append(euron_helper(document))
    return iterms
```

### Explanation, block by block

- **Imports:**
  - `FastAPI`, `HTTPException` — app object + a way to raise structured HTTP errors (not used directly in these particular endpoints, but imported for later use).
  - `BaseModel` (Pydantic) — request body validation, same role as in the SQL class.
  - `AsyncIOMotorClient` (from `motor.motor_asyncio`) — **Motor** is MongoDB's official **asynchronous** Python driver. Async drivers are the standard choice for production-grade APIs because they don't block other requests while waiting on I/O (see the "async/await" explanation below).
  - `ObjectId` (from `bson`) — MongoDB's native unique ID type (imported for potential use, e.g. converting/looking up documents by ID).
  - `os` + `load_dotenv` (`python-dotenv`) — read the `.env` file into environment variables at runtime, instead of hardcoding secrets.
- `load_dotenv()` — loads the key/value pairs from `.env` into the process environment.
- `MONGO_URI = os.getenv("MONGO_URI")` — reads the connection string from the environment (falls back to `None` if not set/`.env` missing).
- `client = AsyncIOMotorClient(MONGO_URI)` — opens an async client connection to the Atlas cluster using the URI.
- `db = client["euron"]` — selects the **database** named `euron` (created earlier in Atlas's UI).
- `euron_data = db["euron_coll"]` — selects the **collection** named `euron_coll` inside that database. This is the object every insert/find operation below actually targets.
- `class eurondata(BaseModel)` — Pydantic schema for incoming data: `name` (str), `phone` (int), `city` (str), `course` (str). Even though MongoDB itself doesn't enforce a schema, the **API layer still validates the shape of incoming data** before it's inserted.
- **`POST /euron/insert`** → `euron_data_insert_helper(data: eurondata)`:
  - `await euron_data.insert_one(data.dict())` — MongoDB's built-in `insert_one()` function inserts a single document. `data.dict()` converts the validated Pydantic object into a plain dictionary (JSON-like) that MongoDB can store directly — no manual field-by-field mapping needed, unlike the SQL `INSERT` statement from yesterday's class.
  - Returns `str(result.inserted_id)` — MongoDB auto-generates a unique `_id` (an `ObjectId`) for every inserted document; this converts it to a plain string for the JSON response.
  - **Note:** the function is declared `async def` and uses `await` because Motor's operations are asynchronous — this was a bug fix made live in class (see "Live debugging" section below).
- `euron_helper(doc)` — a small utility function that reshapes a raw MongoDB document before returning it via the API:
  - Copies `_id` into a new field `"id"` as a string, then deletes the original `"_id"` key.
  - **Why:** MongoDB's `_id` field is an `ObjectId` object, not plain JSON-serializable data — trying to return it directly (or iterate over it as if it were a normal value) caused a `"ObjectId object is not iterable"` error during the live demo. Stripping/renaming it avoids this.
- **`GET /euron/getdata`** → `get_euron_data()`:
  - `cursor = euron_data.find({})` — `find({})` with an empty filter means **"match everything"** (you could instead pass filter criteria like `{"city": "BLR"}` to search on a specific field).
  - `async for document in cursor:` — iterates over the async cursor one document at a time (non-blocking).
  - Each document is passed through `euron_helper()` and appended to `iterms`, which is returned as the final JSON array of all records.
- **`GET /euron/showdata`** — identical logic to `/euron/getdata`; added later in class purely to demonstrate that pushing a code change to GitHub **auto-redeploys** on Render (see CI/CD section below) without any manual redeploy step.

---

## 5. Async / Await — Why It Matters
- **`async def`** marks a function as asynchronous — i.e., **non-blocking**.
- **`await`** is used when calling another asynchronous operation (like a Motor database call) — it tells Python "pause here until this completes, but let other tasks run in the meantime."
- Analogy used in class: driving on a road and hearing an ambulance siren — you make way for the ambulance without leaving the road entirely. Similarly, a non-blocking function yields resources when needed instead of freezing all other incoming requests while it waits.
- This matters most under **high traffic** — many simultaneous requests hitting the same insert/fetch endpoint. A blocking (synchronous) driver would process them strictly one at a time; an async driver lets the server keep handling other requests while waiting on the database.
- **Motor** is MongoDB's official async driver, built specifically for this async/await pattern. **PyMongo** is the older/standard (synchronous) driver — still valid, but not async by default.

---

## 6. Live Debugging Recap (bugs fixed during the session)
| Symptom | Root Cause | Fix |
|---|---|---|
| `No module named 'motor.motoro'` | Typo: `motor.motoro` instead of `motor.motor_asyncio` | Corrected the import path |
| `There is no current event loop in thread 'AnyIO worker thread'` | Function wasn't declared `async` | Added `async` before `def` |
| `document must be an instance of dict, bson.son.SON, ...` | Passed `data` object directly instead of converting it | Used `data.dict()` when calling `insert_one()` |
| `'coroutine' object has no attribute 'inserted_id'` | Missing `await` before the insert call | Added `await` before `euron_data.insert_one(...)` |
| `ObjectId object is not iterable` when fetching all records | Mongo's raw `_id` (an `ObjectId`) isn't directly JSON-serializable/iterable in a loop | Added the `euron_helper()` function to convert `_id` → string `id` and remove the original `_id` key |

---

## 7. Testing
- Same three methods as the SQL class: **Swagger UI** (`/docs`), **Postman**, **curl** — all work identically against MongoDB-backed endpoints.
- Example flow: `POST /euron/insert` with a JSON body → 200 response returns the inserted document's ID → `GET /euron/getdata` (or `/euron/showdata`) returns the full list of stored documents.

---

## 8. Deploying to the Cloud (Render + GitHub) — Real Production Hosting
Unlike yesterday's ngrok tunnel (which only exposes your *local* machine temporarily), today's class did a **real deployment** so the API runs permanently on a server, independent of your laptop.

### Step 1 — Install Git (if not already installed)
- Download from the official Git website for Windows/Mac/Linux, install with default options.
- Git is an open-source **version control** system; GitHub/GitLab/Bitbucket are companies that built hosted products on top of it.

### Step 2 — Create a `.gitignore` file
Prevents secret/unnecessary files from being pushed to GitHub:
```
.env
__pycache__
myenv/
```
- `.env` — keeps your Mongo URI/credentials out of the public repo (GitHub also actively blocks some known secret patterns by default).
- `__pycache__`, the virtual-environment folder (`myenv/`) — not needed in the repo since `requirements.txt` will recreate them on any machine.

### Step 3 — Push code to GitHub
```bash
git init
git add .
git commit -m "Euron API"
git branch main
git remote add origin <your-repo-url>
git push origin main
```
- Create the GitHub repo first (e.g., `euron-mongo-api`) via **GitHub → Repositories → New**.

### Step 4 — Deploy on Render (render.com)
1. Sign up / log in (Google login works).
2. Click **New → Web Service**.
3. Connect your GitHub account and select the repository.
4. Configure:
   - **Language:** Python (auto-detected)
   - **Branch:** `main`
   - **Root directory:** leave default (project root)
   - **Build command:** `pip install -r requirements.txt`
   - **Start command:** `uvicorn main:app --host 0.0.0.0 --port $PORT`
     - `0.0.0.0` = "listen on all network interfaces of this machine" (not `127.0.0.1`, which is local-only).
     - `$PORT` = Render's dynamically assigned port, referenced as an environment variable.
   - **Plan:** Free / Hobby tier.
5. Add **environment variables** in Render's dashboard (this is where the `.env` values go instead of the repo):
   - Key: `MONGO_URI`, Value: your full Mongo connection string (no quotes needed).
6. Click **Deploy**. Render will:
   - Pull the code from GitHub.
   - Run the build command (install dependencies).
   - Run the start command.
   - Once live, gives a public URL like:
     ```
     https://<your-app-name>.onrender.com
     ```
7. Test the same way as local: append `/docs` for Swagger UI, or hit endpoints directly via Postman/curl/browser.

### CI/CD — Automatic Redeployment
- Render + GitHub together form a simple **CI/CD (Continuous Integration / Continuous Deployment)** pipeline.
- Once connected, **any new `git push` to the `main` branch automatically triggers a new deployment** — no manual redeploy step needed.
- Demonstrated live in class: added the `/euron/showdata` endpoint, ran `git add` → `git commit` → `git push origin main`, and the new endpoint appeared live on Render shortly after, without touching Render's dashboard again.
- A manual/forced redeploy option is also available in Render's dashboard if needed.

### Notes & Limitations (Free Tier)
- Free-tier Render doesn't support a custom domain/DNS mapping — that requires a paid plan.
- GitHub Pages **cannot** be used for this — it only serves static sites and provides no compute (no ability to run Python/install dependencies), whereas an API needs an actual running process with CPU/RAM.

---

## 9. Homework (Assigned)
Extend the `euron` collection API with three more operations:
1. **Full update** — replace an entire document by ID.
2. **Partial update** — update only specific field(s) of a document by ID.
3. **Delete** — remove a document by ID.

Requirements:
- Deploy the finished API on **Render** (not ngrok this time — a real hosted URL is required).
- Share the **Render URL** (e.g., `https://your-app.onrender.com/docs`) in the batch group so classmates and the instructor can test it.

---

## Key Takeaways
- **Collections** (MongoDB) vs. **tables** (SQL): both organize data, but collections don't require a predefined schema — validation instead happens at the API layer via Pydantic.
- **Motor** = MongoDB's official async driver; pair it with `async def` / `await` for non-blocking, production-grade APIs — critical under high traffic.
- MongoDB's `_id` field is an `ObjectId`, not directly JSON-serializable — always convert/strip it (e.g., via a small helper function) before returning documents through an API.
- Secrets (DB URIs, passwords) belong in a `.env` file locally and in the hosting platform's **environment variables** panel in production — never hardcoded in source code or pushed to GitHub. Use `.gitignore` to guarantee `.env` never gets committed.
- **Virtual environments** + **`requirements.txt`** are the standard way to manage per-project dependencies cleanly, both locally and during cloud deployment.
- **Render + GitHub** gives a simple, free way to get a real, always-on public API with automatic redeployment on every `git push` — a genuine CI/CD workflow, unlike ngrok's temporary local tunnel.
