# Think Like a Real Programmer

## Step 0: Shut Up and Think

Before writing a single line of code, ask yourself: **what exactly am I building and why?**

Most people fail because they jump to code. Real programmers sit with the problem until it becomes boringly clear.

---

## Step 1: Understand the Problem Domain

### What is a task queue?

Imagine you have a web app. A user clicks "Sign Up". You need to:
- Save them to database (fast — 50ms)
- Send a welcome email (slow — 2 seconds)
- Resize their profile picture (very slow — 10 seconds)

If you do all three synchronously, the user waits 12 seconds staring at a spinner. That's terrible.

**The solution:** Do the fast thing immediately. Put the slow things in a queue. Something else picks them up and does them in the background.

That "something else" is a task queue system.

### Real-world examples

- **Celery** — Python's most popular task queue (used by Instagram, Mozilla)
- **Sidekiq** — Ruby equivalent (used by GitHub)
- **Bull** — Node.js equivalent

### Why build one yourself?

Because a task queue is the **perfect compression of every hard problem in software**:

| Problem | How a task queue forces you to solve it |
|---------|----------------------------------------|
| How do two programs talk? | Network protocols, serialization |
| How do you handle failure? | Retries, dead letter queues, error recovery |
| How do you handle scale? | Concurrency, multiprocessing, connection pooling |
| How do you handle state? | Databases, state machines |
| How do you observe the system? | Logging, metrics, dashboards |
| How do you make it reliable? | Heartbeats, graceful shutdown, persistence |
| How do you make it usable? | APIs, CLIs, documentation |
| How do you make it correct? | Testing, type checking |
| How do you make it fast? | Profiling, optimization, async I/O |
| How do you make it maintainable? | Architecture, patterns, packaging |

One project. Ten problems. That's why this is the one.

---

## Step 2: Define the Requirements

### What must this thing DO?

Split requirements into three tiers:

**Must have (MVP):**
- Define a function as a "task" using a decorator
- Put a task into a queue
- A worker process picks it up and executes it
- If it fails, retry it N times
- Know whether a task succeeded or failed

**Should have (Real):**
- Multiple queues (high priority, low priority, etc.)
- Multiple workers running in parallel
- Task timeout — kill a task if it runs too long
- Exponential backoff on retries
- A way to check task status

**Nice to have (Production):**
- Web dashboard showing live tasks
- CLI tool to manage workers
- Task chaining (task B runs after task A)
- Scheduled tasks (cron-like)
- Rate limiting
- Dead letter queue for permanently failed tasks

### What are the CONSTRAINTS?

This is what separates amateurs from pros. Amateurs only think about what the system does. Pros think about what the system must NOT do:

- A task must never be executed twice (idempotency)
- A task must never be lost if the worker crashes mid-execution
- The system must not crash if Redis goes down temporarily
- A slow task must not block other tasks
- A worker must not become a zombie process
- The queue must not grow unbounded (backpressure)

---

## Step 3: Decompose Into Sub-Problems

Now we break the monster into pieces small enough to solve one at a time.

```
TASK QUEUE SYSTEM
│
├── 1. Task Definition Layer
│       "How do users define tasks?"
│
├── 2. Serialization Layer  
│       "How do we convert a task + its arguments into bytes and back?"
│
├── 3. Broker Layer
│       "How do we store and retrieve tasks?"
│
├── 4. Worker Layer
│       "How do we execute tasks safely?"
│
├── 5. State Management Layer
│       "How do we track what happened to each task?"
│
├── 6. API Layer
│       "How do external systems interact with this?"
│
└── 7. Observability Layer
        "How do we know what's happening?"
```

Each layer is a separate problem. Solve them one at a time. **Each one is independently testable.**

---

## Step 4: Design Each Layer (Mental Models, Not Code)

### Layer 1: Task Definition

**The problem:** A user writes a normal Python function. Somehow, the system needs to know about it, store metadata about it, and be able to call it later by name (not by reference, because the worker is a different process).

**The key insight:** In Python, functions are first-class objects. You can wrap them, inspect them, and store references to them in a dictionary (a "registry").

**The design:**
- A decorator wraps the function
- The decorator adds the function to a global dictionary: `name → function`
- The decorator can also accept options (max retries, timeout, which queue)
- When a worker receives a task by name, it looks it up in this dictionary

**The trap:** The registry must be the same in both the client process and the worker process. This means both must import the same task modules. This is a real constraint that trips people up.

---

### Layer 2: Serialization

**The problem:** You need to send a task across a network or through Redis. Networks only understand bytes. So you need to convert `(function_name, args, kwargs, metadata)` into bytes and back.

**The key insight:** Not everything can be serialized. What if someone passes a database connection as an argument? What if they pass a lambda? You need to decide: fail fast, or silently drop unsupported types?

**The design decisions:**
- **Format:** JSON (human-readable, universal) vs MsgPack (binary, faster) vs Pickle (can serialize anything Python, but a security nightmare)
- **Strategy:** Start with JSON. Add MsgPack as an option. Never use Pickle unless you fully understand the security implications.
- **Custom types:** You need a way to serialize your `TaskState` enum, UUIDs, datetimes, etc. This means a custom encoder/decoder.

**The trap:** JSON converts all dict keys to strings. If you use integers as keys, they'll come back as strings. This causes subtle bugs.

---

### Layer 3: Broker

**The problem:** You need a place to store tasks that is:
- Fast (thousands of operations per second)
- Reliable (survives crashes)
- Supports blocking operations ("wait until a task is available")

**The key insight:** You don't build this from scratch. You use Redis. Redis has exactly the right data structures:
- `LPUSH` / `BRPOP` — perfect for a queue (push to left, pop from right = FIFO)
- `HSET` / `HGETALL` — perfect for storing task state (hash with fields like status, result, error)
- `EXPIRE` — perfect for cleanup

**But** — you should understand what Redis is doing under the hood. It speaks a protocol called RESP (REdis Serialization Protocol). Building a minimal RESP client teaches you how network protocols work.

**The design:**
- A connection class that opens a TCP socket, sends RESP commands, reads RESP responses
- A connection pool so you don't open a new connection for every operation
- A high-level client with methods like `enqueue()`, `dequeue()`, `get_status()`

**The trap:** Connection leaks. If you don't return connections to the pool, you'll exhaust Redis's max connections and everything freezes.

---

### Layer 4: Worker

**The problem:** A long-running process that continuously pulls tasks from the broker and executes them.

**The key insight:** A worker has a lifecycle:
```
STARTUP → REGISTER → HEARTBEAT → POLL → EXECUTE → POLL → EXECUTE → ... → SHUTDOWN
```

Each of these phases has its own challenges.

**Design decisions:**

**Concurrency model:** How many tasks can one worker run at once?
- One at a time (simplest, but wastes resources)
- Thread pool (works for I/O-bound tasks, but Python's GIL limits CPU parallelism)
- Async (great for I/O-bound, but what if the task is CPU-bound?)
- Multiple processes (true parallelism, but heavier)
- **The real answer:** A mix. The worker runs an async event loop for I/O (talking to Redis), but executes CPU-bound tasks in a thread/process pool.

**Task execution safety:**
- What if the task runs forever? → You need a timeout mechanism
- What if the task crashes? → You need to catch the exception, record it, and decide whether to retry
- What if the task modifies global state? → That's the user's problem, but you should document it

**Retries:**
- How many times? (configurable per task)
- How long between retries? (exponential backoff: 1s, 2s, 4s, 8s...)
- Should ALL exceptions be retried? (No. `ValueError` might be a permanent bug. `ConnectionError` might be transient.)
- This means you need a retry decision system — not just "retry on any error"

**Graceful shutdown:**
- When you send SIGTERM to a worker, it shouldn't die mid-task
- It should finish the current task, stop accepting new ones, then exit
- This requires signal handling and an event loop

**The trap:** Zombie workers. If a worker crashes hard (OOM kill, segfault), its heartbeat stops but Redis doesn't know. You need a separate mechanism to detect dead workers.

---

### Layer 5: State Management

**The problem:** A user submits a task and wants to know: "Is it done? Did it fail? What was the result?"

**The key insight:** Task state is a state machine:

```
PENDING → QUEUED → RUNNING → SUCCESS
                      ↓
                    FAILED → RETRYING → RUNNING → ...
                      ↓
                  (max retries exceeded) → PERMANENTLY_FAILED
```

**Design:**
- Store state in Redis hashes (one hash per task)
- Every state transition updates the hash
- The API layer reads from these hashes

**The trap:** Race conditions. What if two workers somehow pick up the same task? You need some form of locking or atomic operations.

---

### Layer 6: API

**The problem:** How does a web app submit tasks and check status?

**The design:**
- `POST /tasks` — submit a task (body: `{"task": "send_email", "args": [...], "kwargs": {...}}`)
- `GET /tasks/{id}` — get task status
- `GET /tasks` — list all tasks (with filters)
- `GET /workers` — list active workers
- `DELETE /tasks/{id}` — cancel a task

**The deeper problem:** You could use Flask or FastAPI. But if you want to truly understand Python, build a minimal async web framework. It's simpler than you think:

1. Open a TCP socket
2. Read the HTTP request (it's just text)
3. Parse the method, path, headers, body
4. Route to the right handler function
5. Format an HTTP response and send it back

That's it. That's what Flask does under the hood.

---

### Layer 7: Observability

**The problem:** In production, things break in ways you never imagined. You need to see inside the system without changing the code.

**What you need:**
- **Structured logging** — not `print()`, but JSON logs with task_id, timestamp, worker_id
- **Metrics** — tasks/second, queue depth, failure rate (could push to Prometheus)
- **Dashboard** — a web page that shows live task status, worker health, queue lengths
- **Tracing** — follow a single task through the entire system

---

## Step 5: Identify the Hard Problems

Not all problems are equal. The easy parts will take 20% of your time. The hard parts will take 80%.

### Hard Problem 1: Exactly-once execution

Can you guarantee a task runs exactly once? 

**Answer: No. Not truly.** This is a fundamental limitation of distributed systems. You can only offer:
- **At-most-once** (might lose tasks) — bad
- **At-least-once** (might run twice) — what most systems do
- **Exactly-once** (only possible if the task is idempotent, i.e., running it twice gives the same result as running it once)

**The lesson:** Design your tasks to be idempotent. Document this clearly.

### Hard Problem 2: Worker crashes mid-task

Worker is running a task. It's 90% done. The process gets killed (OOM, deploy, server restart).

What happens? The task was marked "running" but never marked "success" or "failed". It's stuck.

**Solution:** 
- When a worker picks up a task, record a "last seen running" timestamp
- A separate monitor checks for tasks that have been "running" too long
- If the worker's heartbeat has stopped, the task is reassigned

### Hard Problem 3: Queue buildup (backpressure)

What if tasks arrive faster than workers can process them? The queue grows forever. Redis runs out of memory.

**Solution:**
- Monitor queue length
- When it exceeds a threshold, either: reject new tasks, or spin up more workers (auto-scaling)
- This is a whole subsystem in production systems

### Hard Problem 4: Testing async concurrent code

How do you test that a retry works? That a timeout fires? That two workers don't grab the same task?

**Solution:**
- Don't test against real Redis in unit tests — mock the broker
- Use `asyncio.test_utils` for time manipulation
- Write integration tests that spin up real Redis (use Docker)
- Test failure modes deliberately (kill worker mid-task, kill Redis, etc.)

---

## Step 6: The Build Order

This is critical. **Don't build layers in order.** Build in dependency order, where each step produces something testable:

```
Week 1:  Task registry (decorator + dict lookup)
         ↓ Test: can you register a task and look it up by name?
         
Week 2:  Serialization (JSON encode/decode of task + args)
         ↓ Test: can you round-trip a task through JSON?
         
Week 3:  Redis client (connect, LPUSH, BRPOP, HSET, HGETALL)
         ↓ Test: can you push a string and pop it?
         
Week 4:  Basic worker (poll Redis, execute task, update state)
         ↓ Test: can a worker process one task end-to-end?
         
Week 5:  Retries + timeouts + error handling
         ↓ Test: does a failing task retry? Does a hanging task timeout?
         
Week 6:  Connection pooling + multiple workers
         ↓ Test: can 4 workers process tasks concurrently?
         
Week 7:  Graceful shutdown + heartbeats
         ↓ Test: does SIGTERM finish the current task and exit cleanly?
         
Week 8:  API server
         ↓ Test: can you submit and query tasks via HTTP?
         
Week 9:  Dashboard (HTML + WebSocket for live updates)
         ↓ Test: can you see tasks moving through the system in real-time?
         
Week 10: CLI tool + packaging + documentation
         ↓ Test: can someone pip install your package and use it?
```

---

## Step 7: What This Teaches You (The Map)

When you're done, here's what you'll actually understand — not "I used X", but "I understand why X exists":

| You'll Stop Asking | You'll Start Understanding |
|---|---|
| "What's a decorator for?" | Decorators are function composition — they solve "I want to add behavior without changing the function" |
| "When do I use async?" | Async is for I/O-bound concurrency — when you're waiting on network/disk, not CPU |
| "Why do I need type hints?" | Types are documentation that the machine can verify — they catch bugs at write-time, not runtime |
| "What's a metaclass?" | Metaclasses are "classes that make classes" — used when you need to control how a class is created, not how instances behave |
| "Why use a connection pool?" | Opening a TCP connection is expensive (~1ms). Reusing them means 100x throughput improvement |
| "What's the GIL?" | The GIL prevents multiple threads from executing Python bytecode simultaneously — it's irrelevant for I/O, fatal for CPU work |
| "Why not just use threads?" | Threads share memory → race conditions. Processes don't → need serialization to communicate. Choose based on the problem. |

---

## Step 8: The Real Test

You know you've mastered Python when you can answer these questions about your own project:

1. **"What happens if Redis restarts while a worker is executing a task?"**
2. **"What happens if two workers pop the same task at the exact same millisecond?"**
3. **"What happens if a task's arguments include an open file handle?"**
4. **"What happens if the queue has 1 million tasks and you restart all workers?"**
5. **"What's the maximum throughput of your system and what's the bottleneck?"**
6. **"How much memory does each worker use and why?"**
7. **"What happens if someone passes a lambda as a task argument?"**

If you can answer all seven — with specifics, not hand-waving — you're not just a Python programmer. You're an engineer.

---

## Your Next Move

Don't write code yet. 

Pick **Layer 1 (Task Registry)**. Grab a piece of paper. Draw out:
- What the user writes (the decorator syntax)
- What the decorator needs to store
- How a worker looks up a task by name
- What edge cases exist (duplicate names? Missing tasks?)

When you can explain it to a rubber duck without hesitation, **then** open your editor.

Which layer do you want to think through first?