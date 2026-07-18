# FastAPI with Python — Class Notes & Practice Examples

https://euron.one/learn/86cf35e7-2458-4d84-a197-a9f1a77e2f69?tab=Overview&activeLectureId=36a2b33c-3233-43d8-8c70-960ff023fd48
9th NOv2025 class of Genai bootcamp2.0

*Based on a hands-on FastAPI session covering setup, GET/POST requests, path & query parameters, Pydantic validation, and exposing a local API to the internet with ngrok.*

---

## 1. What Problem Does an API Solve?

- You can call a Python function directly only from **Python code**.
- If someone writing Java, JavaScript, .NET, etc. wants to use your function, they can't call it directly — it's "Python-dependent."
- An **API** exposes your function over **HTTP**, a language-independent protocol. Anyone, from any programming language or tool, can now call it.
- This is exactly how real-world systems (banks, payment gateways, etc.) — all built in different tech stacks — are able to talk to each other.

> **Core idea:** API = a way to expose a function to the outside world so any system can execute it, regardless of the language it was written in.

---

## 2. Environment Setup

### Install FastAPI and Uvicorn
```bash
pip install fastapi
pip install uvicorn
```

- **FastAPI** — the web framework library used to build the API.
- **Uvicorn** — an ASGI server that actually runs/serves your FastAPI application.
- Packages are installed from **PyPI** (Python Package Index) — the most popular repository for Python libraries.

### Common installation issues & fixes
| Problem | Fix |
|---|---|
| `pip is not recognized` | Python/pip not installed properly, or not in PATH |
| `Access denied` / permission errors (common on office laptops) | Run VS Code / terminal "as Administrator" |
| Prefer isolation | Use a virtual environment (`venv` or `conda`) and install inside it |

---

## 3. Your First FastAPI App

**`main.py`**
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def test():
    return {"message": "Hello World"}
```

Key points:
- `from fastapi import FastAPI` — imports the FastAPI class.
- `app = FastAPI()` — creates an **instance/object** of the app; this object is used to register routes.
- `@app.get("/")` — a **decorator** that exposes the function below it as an API endpoint that responds to **GET** requests at the route `/`.
- The function no longer just `print`s — it **returns** a dictionary (JSON), which is what the caller receives.

### Running the server
```bash
uvicorn main:app
```
- `main` → the filename (`main.py`, without `.py`)
- `app` → the FastAPI object variable name inside that file

Auto-reload on code changes (very useful during development):
```bash
uvicorn main:app --reload
```
Without `--reload`, you must stop (`Ctrl + C`) and restart the server manually every time you edit code.

### Understanding the URL
```
http://127.0.0.1:8000/
```
- `127.0.0.1` (aka `localhost`) — your own machine.
- `8000` — the **port** the app is running on (a machine can run many processes, each on its own port).
- `/` — the **route / endpoint** — determines which function gets executed.

You can test this by pasting the URL into a browser (works for GET requests only).

---

## 4. Multiple Routes / Endpoints

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def test():
    return {"message": "Hello World"}

@app.get("/sudh_test_dummy")
def test_one():
    return {"message": "My name is Sudhanshu. I used to teach data."}
```

- Each function can be exposed at its own custom route — just like a directory structure.
- Every new route needs the server to reload (handled automatically with `--reload`).

---

## 5. Exposing Your Local API to the World — ngrok

Your API runs on `localhost`, so only your own machine can access it — unless you expose it.

**ngrok** is a free tunneling service: it takes something running on a local port and gives it a public HTTPS URL, so anyone on the internet can access it (without deploying to the cloud).

### Steps
1. Go to `ngrok.com`, sign up/login (Google login works).
2. Download & install ngrok (or use the provided install command with your auth token).
3. Authenticate once:
   ```bash
   ngrok config add-authtoken <your-token>
   ```
4. Expose your running app's port:
   ```bash
   ngrok http 8000
   ```
5. ngrok gives you a public URL (e.g. `https://xxxx.ngrok-free.app`) that tunnels to your `localhost:8000`.

> **Important:** Your local server must already be running before you run `ngrok http <port>` — ngrok just tunnels to what's already there; it doesn't start your app.

### Common ngrok troubleshooting
- `ngrok not recognized` → not installed / not in PATH.
- Wrong port number → make sure it matches the port your `uvicorn` app is actually running on.
- Access denied errors → may need admin permissions, or antivirus is blocking it.
- `ERR_NGROK_...` / "could not import module main" → you're running `uvicorn` from the wrong directory. `cd` into the folder containing `main.py` first.

---

## 6. Testing APIs with Postman

Postman is a dedicated (non-Python) tool for API testing, widely used across the industry regardless of tech stack.

- Choose the **HTTP method** (GET, POST, PUT, PATCH, DELETE...).
- Paste your URL (local or ngrok).
- Click **Send** to view the response, status code, etc.

### HTTP Status Codes seen
| Code | Meaning |
|---|---|
| 200 | OK — success |
| 404 | Not Found — wrong/undefined route |
| 405 | Method Not Allowed — e.g., hitting a POST-only route from a browser |
| 422 | Unprocessable Entity — request body/data failed validation |
| 500 | Internal Server Error — something broke in your code |

---

## 7. Path Parameters (Dynamic Routes)

Used to pass data **as part of the URL path**.

```python
students = {
    1: {"name": "Akash"},
    2: {"name": "Rohit"},
    3: {"name": "Sachin"}
}

@app.get("/student/{student_id}")
def student_search(student_id: int):
    return {
        "id": student_id,
        "name": students[student_id]["name"]
    }
```

- `{student_id}` in the route is a **placeholder** — whatever value is passed in the URL gets mapped to the function's `student_id` argument (same name required).
- `student_id: int` is a **type hint** — FastAPI (via Pydantic) will try to interpret the incoming value as an `int`. Without this, passing `"1"` as a string could cause a `500 Internal Server Error` when used as a dictionary key that expects an int.
- Call it like: `GET /student/1` → returns Akash's data.

---

## 8. Query Parameters

Used to pass data **after a `?`** in the URL — the same style used by Google search (`?q=data+science`) or Gmail-style forms.

```python
@app.get("/add_student")
def add_student(student_id: int, name: str):
    students[student_id] = {"name": name}
    return {"message": "Student added successfully", "students": students}
```

Call it as:
```
http://localhost:8000/add_student?student_id=4&name=Sudhanshu
```
- `?` starts the query string.
- `key=value` pairs are joined with `&`.
- No spaces allowed; string values don't need quotes in the URL itself.

> **GET vs visibility:** With GET, query parameters ARE visible in the URL (like a Google search). This is fine for non-sensitive data, but not for passwords or private data — that's why login forms use POST instead (data goes in the request **body**, not the URL).

### 8.1 Deep Dive — How Query Parameters Add a New Student to the Original Data

This is one of the trickier parts of the class, so let's slow it down with a full before/after example.

**What query parameters actually are:** extra bits of data tacked onto a URL after a `?`, as **key=value** pairs, joined by `&` if there's more than one:

```
http://localhost:8000/add_student?student_id=4&name=Sudhanshu
```

- Everything before `?` → the route (`/add_student`)
- Everything after `?` → the data being passed in (`student_id=4` and `name=Sudhanshu`)

FastAPI automatically maps these to your function's arguments **by name** — as long as your function parameter is called `student_id`, FastAPI knows to grab `student_id` from the query string.

**The original data (before):**
```python
students = {
    1: {"name": "Akash"},
    2: {"name": "Rohit"},
    3: {"name": "Sachin"}
}
```

**The endpoint that adds to it:**
```python
@app.get("/add_student")
def add_student(student_id: int, name: str):
    students[student_id] = {"name": name}
    return {"message": "Student added", "students": students}
```

Notice: this is decorated with `@app.get`, **not** `@app.post`. It's a GET endpoint. But inside the function body, we're doing a dictionary write (`students[student_id] = ...`) — normally a "write/create" operation, the kind of thing POST is meant for.

**Calling it** — hit this in a browser or Postman:
```
http://localhost:8000/add_student?student_id=4&name=Sudhanshu
```
FastAPI reads this as `student_id = 4`, `name = "Sudhanshu"` → runs the function.

**The data (after) — response returned:**
```json
{
  "message": "Student added",
  "students": {
    "1": {"name": "Akash"},
    "2": {"name": "Rohit"},
    "3": {"name": "Sachin"},
    "4": {"name": "Sudhanshu"}
  }
}
```

#### Why this "feels like POST" but isn't really POST

| | This GET example | A true POST |
|---|---|---|
| HTTP method | GET | POST |
| Where data travels | In the URL itself (`?student_id=4&name=Sudhanshu`) | In the request **body** (invisible in the URL) |
| Visible in URL / browser history / server logs? | **Yes** | No |
| Callable directly from a browser address bar? | **Yes** | No — needs Postman or code |
| Semantically "correct" HTTP usage | No — GET is meant for *reading*, not *writing* | Yes — POST is meant for *creating/sending* data |

So functionally, yes — this GET endpoint **does** perform an addition/update, exactly like the POST version does. It's shown this way in class on purpose, to prove that *technically* you can pass data through query params and use it to mutate your dictionary. But it's not the recommended pattern for real applications, because:

- Sensitive data (passwords, tokens, IDs) would be exposed in the URL, browser history, and server access logs.
- Semantically, GET should be "safe" — calling it shouldn't change server state.

The proper POST version does the exact same job, just with the data hidden in the body and validated via a Pydantic model:

```python
from pydantic import BaseModel

class NewData(BaseModel):
    student_id: int
    name: str

@app.post("/add_student_new_value")
def add_student_new_value(new_data: NewData):
    students[new_data.student_id] = {"name": new_data.name}
    return students
```

Same result, same dictionary gets updated — just sent as JSON in the request body instead of the URL, and tested via Postman (Body → raw → JSON) rather than typing a URL.

> **Bottom line:** query parameters *can* be used to add/update data (as shown with GET), but that's a demonstration of what's technically possible — not best practice. POST + Pydantic is the correct way to do additions/writes in real projects.

---

## 9. GET vs POST

| | GET | POST |
|---|---|---|
| Purpose | Retrieve / read data | Send / create data |
| Data location | Visible in the URL (query params) | Hidden in the request **body** |
| Can be called from a browser URL bar? | Yes | No — needs a tool like Postman or code |
| Real-world analogy | Google search | Gmail login form |

### Defining a POST endpoint
```python
@app.post("/add_student_new_value")
def add_student_new_value(new_data: dict):
    students[new_data["student_id"]] = new_data["name"]
    return students
```

- Cannot be hit from a browser address bar — you'll get **405 Method Not Allowed**.
- Test in Postman: select **POST**, go to **Body → raw → JSON**, and send data like:
```json
{
  "student_id": 10,
  "name": "Sudhanshu"
}
```

---

## 10. Data Validation with Pydantic

Whenever you accept a request **body** (typical with POST), FastAPI uses **Pydantic** to validate incoming data against a defined schema — this is standard practice across almost all modern Python frameworks/libraries (LangChain, CrewAI, LangGraph, etc. all use it too).

```python
from pydantic import BaseModel

class NewData(BaseModel):
    student_id: int
    name: str

@app.post("/add_student_new_value")
def add_student_new_value(new_data: NewData):
    students[new_data.student_id] = new_data.name
    return students
```

- `class NewData(BaseModel)` — inherits from Pydantic's `BaseModel`, turning it into a validation schema.
- FastAPI checks the incoming JSON against this schema **before** running your function body.
- If validation fails → you get a `422 Unprocessable Entity` error.
- Note: Pydantic (in recent versions) requires **Python 3.10+**. Errors like "cannot import BaseModel" often mean an old Python environment (e.g. 3.9) is being used — switch to a 3.10+ virtual environment.

---

## 11. In-Memory Data vs Persistent Data (Important Concept!)

A key point raised in class:

- The `students` dictionary is stored in **RAM (main memory)**, not on disk.
- When your server is actively running (serving requests on a single thread), that thread is busy handling the "serving" job — there's no separate thread available to persist/update the running Python file's variable state permanently.
- So additions via API calls are visible in that request/response cycle, but **do not persist** if you restart the server, and won't sync across every open script window.
- **To truly persist data**, you need external storage:
  - A file (`.txt`, `.json`, `.csv`, `.xlsx`) written to disk, or
  - A real database.
- Files/databases are read from and written to **disk (secondary memory)**, which survives server restarts — unlike a plain Python variable, which lives only in RAM for the lifetime of the process.

---

## 12. Full Example Code (Consolidated)

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

# --- In-memory "database" ---
students = {
    1: {"name": "Akash"},
    2: {"name": "Rohit"},
    3: {"name": "Sachin"}
}

# --- Basic GET routes ---
@app.get("/")
def home():
    return {"message": "Hello World"}

@app.get("/students")
def get_students():
    return students

# --- GET with path parameter ---
@app.get("/student/{student_id}")
def student_search(student_id: int):
    return students.get(student_id, {"error": "Student not found"})

# --- GET with query parameters ---
@app.get("/add_student")
def add_student(student_id: int, name: str):
    students[student_id] = {"name": name}
    return {"message": "Student added", "students": students}

# --- Pydantic schema for POST body ---
class NewData(BaseModel):
    student_id: int
    name: str

# --- POST route ---
@app.post("/add_student_new_value")
def add_student_new_value(new_data: NewData):
    students[new_data.student_id] = {"name": new_data.name}
    return students
```

Run with:
```bash
uvicorn main:app --reload
```

---

## 13. Practice Exercises

Try these on your own, building on the code above.

### Exercise 1 — Basic exposure
Create a new function `get_time()` that returns the current server time as JSON (`{"time": "..."}`). Expose it at `/current-time` using GET. Test it in your browser.

### Exercise 2 — Path parameters
Add a `courses` dictionary:
```python
courses = {
    101: "Python Basics",
    102: "FastAPI",
    103: "Machine Learning"
}
```
Create a GET endpoint `/course/{course_id}` that returns the course name for a given ID, and a friendly error message if the ID doesn't exist.

### Exercise 3 — Query parameters
Create a GET endpoint `/search` that takes two query parameters, `keyword` and `limit` (both optional, with defaults), and returns a mock "search results" message using both values. Test by hitting:
```
/search?keyword=fastapi&limit=5
```

### Exercise 4 — Update via GET (not recommended in production, but good practice)
Add an endpoint `/update_course` that takes `course_id` and `new_name` as query parameters and updates the `courses` dictionary. Confirm via `/course/{course_id}` that the update worked (remember: it will only persist in the current running process's memory).

### Exercise 5 — POST + Pydantic validation
Create a Pydantic model `NewCourse` with fields `course_id: int` and `name: str`. Create a POST endpoint `/add_course` that adds a new entry to the `courses` dictionary using this model. Test it in Postman with:
```json
{
  "course_id": 104,
  "name": "Deep Learning"
}
```
Then verify it via GET `/course/104`.

### Exercise 6 — Validation failure test
In Postman, deliberately send invalid data to `/add_course` (e.g., `"course_id": "abc"`, a string instead of an int). Observe the `422` error and read the validation message FastAPI returns.

### Exercise 7 — Expose to the world
1. Run your app with `uvicorn main:app --reload`.
2. In a separate terminal, run `ngrok http 8000`.
3. Share the generated ngrok URL with a friend (or open it on your phone's mobile data) and confirm they can access `/` and `/students`.

### Exercise 8 — Delete operation (stretch goal)
FastAPI also supports `@app.delete("/route")`. Try creating a DELETE endpoint `/delete_student/{student_id}` that removes an entry from the `students` dictionary if it exists, and returns an appropriate message either way.

### Exercise 9 — Persisting to a file (stretch goal)
Modify the `add_student_new_value` POST endpoint so that, in addition to updating the in-memory dictionary, it also writes the full `students` dictionary to a `students.json` file on disk using Python's `json` module. Restart your server and confirm the file's contents are still there (even though the in-memory dictionary resets).

---

## 14. Key Terms Glossary

| Term | Meaning |
|---|---|
| **API** | A way to expose a function so other systems/languages can call it over a network |
| **Endpoint** | A specific URL + method combination mapped to a function |
| **GET** | HTTP method to retrieve data; params visible in URL |
| **POST** | HTTP method to send data; params hidden in the request body |
| **Path parameter** | Value embedded directly in the URL path, e.g. `/student/{id}` |
| **Query parameter** | Value passed after `?` in the URL, e.g. `?id=1&name=x` |
| **Uvicorn** | ASGI server that runs your FastAPI app |
| **Pydantic (`BaseModel`)** | Library used for schema/data validation |
| **ngrok** | Service to tunnel a local port to a public internet URL |
| **Localhost / 127.0.0.1** | Refers to your own machine |
| **Port** | A numbered "door" on your machine where a specific process listens |
| **In-memory data** | Data stored only in RAM — lost/not synced properly across requests/restarts |
| **Persistent data** | Data written to disk (file or database) — survives restarts |

---

*Next class topics (as mentioned by the instructor): deeper dive into GET/PUT/POST/DELETE, deployment of FastAPI apps, and then moving on to vector databases.*
