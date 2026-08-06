---
name: multithreading-rules
description: USE WHEN writing, modifying, or reviewing multi-threaded or multi-process Python code in this repository — threading, concurrent.futures, multiprocessing, queue-based workers, or the dispatcher daemon. Enforce these rules before accepting any change.
---

# Multi-threaded Python Safety Rules

Non-negotiable rules for every thread/process in this repository. The collector dispatcher (`src/agent/dispatcher.py`) and all workers must comply.

## 1. Shared-state discipline

- Every shared mutable object is owned by exactly **one** thread, or guarded by a **lock** that every access takes.
- Document ownership/locking in a comment at declaration: `# owned by dispatcher thread; workers read snapshots only`.
- Never mutate a collection while another thread iterates it. Hand workers immutable snapshots (`tuple(...)`, `copy.deepcopy`).

## 2. Locking

- Use `threading.Lock` by default; `threading.RLock` only when the same thread legitimately re-acquires.
- Acquire locks in a **consistent global order** to prevent deadlock. Never hold a lock across I/O or network calls.
- Prefer `queue.Queue`/`ThreadPoolExecutor` over hand-rolled lock code whenever possible.
- Never use `time.sleep` to coordinate threads — use `threading.Event` / `Condition` with timeouts.

## 3. Executors & queues

- Use `concurrent.futures.ThreadPoolExecutor` under a `with` block (guarantees `shutdown(wait=True)`).
- `max_workers` comes from config/env; default modest (e.g., 4), never unbounded and never `os.cpu_count()` for burst work without a cap.
- Work queues must be **bounded**. Handle `queue.Full` with explicit backpressure (retry-after or drop-and-count), never silent loss.
- Submit unit-of-work objects (small, immutable payloads), not closures over shared state.

## 4. Exceptions & fail-fast

- Wrap worker bodies in `try/except` that logs via `src/lib/logging.py` and converts to a controlled result. Catch `BaseException` only to run shutdown handlers.
- A dead worker must be **visible**: log + increment a failure counter surfaced by the CLI. Never `except Exception: pass`.
- Re-check a shared `threading.Event` after each wait and exit promptly on shutdown.

## 5. Shutdown & signals

- Install `SIGTERM`/`SIGINT` handlers that set a shutdown `Event`, stop accepting work, drain the bounded queue, `executor.shutdown(wait=True, cancel_futures=True)`, flush the spool, then exit 0.
- Support `contextlib`-style / `with`-based lifecycle so a crash path also flushes.
- Never call `os._exit` or `sys.exit` from a worker.

## 6. GIL & CPU-bound work

- Threads are for **I/O-bound** work only. CPU-bound parsing/metrics belong in a process pool (`multiprocessing`) or a separate worker process. Document the choice.

## 7. Idempotency & publishing

- Publishing to the external API must be retry-safe: attach an idempotency key per event batch; retry with exponential backoff + jitter (bounded attempts).
- Never re-publish the same event without checking the spool state.

## 8. Forbidden patterns

- `while True` polling loops without backoff/jitter.
- Global mutable singletons mutated from multiple threads.
- Shared `list.append` / `dict` writes without a lock.
- `logging` level changes from worker threads.
- Unbounded queues.
- `threading.Timer` recursion chains for periodic work.

## 9. Refactoring rules

- Preserve the dispatcher's shutdown contract: any refactor keeps the `Event → drain → shutdown → flush` ordering.
- Change one concurrency concern per change; pair with tests.
- Adding a new collector: must be independently stoppable, idempotent, registered with the dispatcher, and covered by a unit test that exercises concurrent submission.

## 10. Before-merge checklist

- [ ] Shared state ownership documented.
- [ ] Locks ordered; no lock held across network I/O.
- [ ] Executor used under `with`; workers wrapped with controlled exceptions.
- [ ] Queue bounded; backpressure handled.
- [ ] Shutdown path drains and flushes; signal handlers present.
- [ ] Idempotency keys on publishes; retries bounded.
- [ ] No forbidden patterns.
- [ ] `make lint && make test` green.
