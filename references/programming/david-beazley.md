# Async & Concurrency Quality Reference — David Beazley

Philosophy: David Beazley ("Python Cookbook", "Python Essential Reference", creator of curio async framework). Focus: asyncio internals, coroutines, generators, concurrency patterns, event loops, GIL, race conditions, proper async/await usage, structured concurrency.
Stack context: Python 3.10+ async trading bot, asyncio event loop, websockets, aiohttp, multiple concurrent WS streams (Binance, Polymarket RTDS, Polymarket CLOB), background tasks (redeem worker, PnL resolver), FAK order execution with timeouts.

Every finding must describe the **concrete failure mode** — not just "bad practice."

---

## Principle 1: A single blocking call in the event loop freezes everything

*Beazley: "The event loop is single-threaded. One blocking call and your entire application goes catatonic. There is no magic scheduler coming to save you."*

This is the #1 bug pattern in async Python codebases. When `requests.get()` or `urllib.request.urlopen()` appears inside an `async def`, the entire event loop blocks. Every WebSocket heartbeat stops. Every pending order times out server-side. Every price update is missed. The bot is clinically dead for the duration of that blocking call, but the process is still alive — so no watchdog triggers.

### What to check

**Blocking HTTP libraries in async code**
- Search for `import requests`, `import urllib`, `import http.client` in any module that is called from the event loop. These are synchronous — they block the thread.
- The fix is `aiohttp` or `httpx` with async mode, not wrapping `requests` in `run_in_executor` (that's a band-aid, not a cure — see Principle 9).
- Severity: **P1** — a `requests.get()` to Polymarket API taking 2 seconds blocks ALL WS streams for 2 seconds. Heartbeats miss, connections drop, stale data propagates.

**`time.sleep()` in async code**
- `time.sleep(n)` blocks the event loop for `n` seconds. Use `await asyncio.sleep(n)`.
- Even `time.sleep(0.1)` in a tight loop accumulates — 10 iterations = 1 second of total freeze.
- Severity: **P1** if in the main trading loop or any frequently-called path.

**Blocking file I/O**
- `open().read()`, `json.load(open(...))`, `pathlib.Path.read_text()` — all blocking. For occasional config reads this is P3. For logging to disk on every tick, this is P1.
- Use `aiofiles` for frequent file operations, or `loop.run_in_executor()` for infrequent ones.
- Severity: **P1** if in the hot path (per-tick, per-message), **P3** if at startup only.

**DNS resolution**
- `aiohttp` uses `getaddrinfo()` which is blocking by default. A DNS timeout (common on VPS) blocks the loop for 5-30 seconds.
- Use `aiodns` as the resolver: `aiohttp.TCPConnector(resolver=aiohttp.AsyncResolver())`.
- Severity: **P2** — rare but catastrophic when it hits.

---

## Principle 2: Every network call needs a timeout — no exceptions

*Beazley: "An await without a timeout is a promise that your program might hang forever. Networks are unreliable. Your code must be more pessimistic than you are."*

An `await ws.recv()` without a timeout will hang indefinitely if the remote end silently dies (no FIN, no RST — just silence). The coroutine is suspended forever. The task looks alive but will never resume. Meanwhile, the bot thinks the WebSocket is connected because no exception was raised.

### What to check

**Naked `await` on network operations**
- Every `await session.get(...)`, `await session.post(...)`, `await ws.recv()`, `await ws.send(...)` must be wrapped in `asyncio.wait_for(coro, timeout=N)` or use the library's built-in timeout parameter.
- Search for `await.*recv\(\)` and `await.*send\(\)` without an enclosing `wait_for` or `timeout=` parameter.
- Severity: **P1** — a hung `recv()` on the Binance WS means no price updates, and the bot trades on stale prices indefinitely.

**Timeout values that are too generous**
- `timeout=300` (5 minutes) on a WebSocket recv is effectively no timeout. If Binance hasn't sent data in 30 seconds, something is wrong.
- Appropriate timeouts: WS recv = 15-30s (heartbeat interval + margin), HTTP request = 5-10s, order placement = 10-15s.
- Severity: **P2** for timeouts > 60s on any operation that should complete in seconds.

**Missing timeout on `asyncio.gather()`**
- `await asyncio.gather(task_a, task_b)` hangs if either task hangs. Wrap in `asyncio.wait_for()` or use `asyncio.wait()` with a timeout parameter.
- Severity: **P2** if any gathered task involves network I/O.

**Timeout handling — retry vs raise**
- When a timeout fires, does the code retry, raise, or silently continue? `except asyncio.TimeoutError: pass` is worse than no timeout — it swallows the evidence that the network is broken.
- Severity: **P1** if `TimeoutError` is caught and silently ignored.

---

## Principle 3: WebSocket connections die silently — you must actively detect staleness

*Beazley: "TCP connections don't notify you when they die. Half-open connections are the ghosts that haunt long-running systems."*

A WebSocket connection can be in a state where `ws.open` returns `True` but no data will ever arrive. The remote server crashed, an intermediate proxy timed out, or the network path is broken. TCP keepalive (if enabled) takes 2+ hours to detect this by default. Your bot cannot wait 2 hours.

### What to check

**Staleness detection via heartbeat**
- For each WS stream (Binance, RTDS, CLOB): is there a "last message received" timestamp? Is there a coroutine that checks `now - last_message > threshold` and triggers reconnection?
- The threshold must be tighter than the server's ping interval. If Binance sends pings every 3 minutes, check staleness every 60 seconds with a threshold of 90 seconds.
- Severity: **P1** — without staleness detection, a dead WS means the bot trades on frozen data. For a price feed, this is catastrophic.

**Reconnection logic**
- When a WS is detected as stale or disconnected, does the code: (1) close the old connection cleanly, (2) wait with exponential backoff, (3) re-establish the connection, (4) re-subscribe to the correct channels, (5) validate that data is flowing again?
- Common bug: reconnection re-creates the WS object but doesn't re-subscribe. The connection is open but no data arrives.
- Severity: **P1** if there's no reconnection logic. **P2** if reconnection exists but doesn't re-subscribe.

**Heartbeat/pong handling**
- Binance requires responding to ping frames. Polymarket RTDS expects periodic heartbeat messages. If the bot doesn't respond, the server disconnects.
- Check: is the pong handler automatic (most WS libraries do this) or does the protocol require an application-level heartbeat response?
- For RTDS: is the heartbeat message sent on the correct interval? Missing even one can trigger disconnect.
- Severity: **P1** for missing application-level heartbeat when the protocol requires it.

**Concurrent reconnection storms**
- If 3 WS streams all detect staleness simultaneously (e.g., the VPS lost internet for 30s), do they all try to reconnect at the same time? This can overwhelm DNS resolution, hit rate limits, or exhaust file descriptors.
- Use jittered backoff: `delay = base * (2 ** attempt) + random.uniform(0, 1)`.
- Severity: **P2** for reconnection without jitter.

---

## Principle 4: Background tasks must not silently die

*Beazley: "When you fire-and-forget a task, you've made an implicit promise that you don't care if it fails. In a trading bot, you always care."*

`asyncio.create_task(some_coro())` launches a background task. If that task raises an exception and nobody awaits it, Python logs a warning ("Task exception was never retrieved") and the task is gone. The redeem worker is dead. The PnL resolver is dead. Nobody notices until positions aren't being redeemed or PnL numbers stop updating.

### What to check

**Unmonitored `create_task()` calls**
- Every `asyncio.create_task()` must have a corresponding mechanism to detect failure: (a) store a reference to the task, (b) add a done callback, or (c) await it somewhere.
- `asyncio.create_task(some_coro())` with no variable assignment = guaranteed silent death on exception.
- Severity: **P1** for background tasks that handle money (redeem worker, order executor, position reconciler).

**Done callbacks for critical tasks**
- Add `task.add_done_callback(handle_task_exception)` where the callback checks `task.exception()` and triggers restart or alert.
- The callback runs in the event loop, so it must not block.
- Severity: **P2** if tasks are stored but never checked for exceptions.

**Task restart logic**
- When a background task dies, should it be restarted? The redeem worker should be restarted (transient network errors). A task that dies due to a logic bug should NOT be restarted in a tight loop — that's an infinite error loop consuming CPU and logs.
- Use restart with a counter: restart up to N times within M minutes, then halt and alert.
- Severity: **P2** for no restart logic on essential tasks. **P1** for unbounded restart loops.

---

## Principle 5: Race conditions happen at `await` boundaries — every `await` is a yield point

*Beazley: "People think async code is safe from race conditions because there's only one thread. They're wrong. Every await is a context switch, and your invariants can break at every one of them."*

In asyncio, code between two `await` statements runs atomically. But the moment you hit `await`, any other ready coroutine can run and mutate shared state. If you read a price, `await` an order placement, then read the price again, it may have changed. If two coroutines both check inventory, `await`, then modify inventory, the modifications can conflict.

### What to check

**Read-await-write on shared state**
- Pattern: `value = shared_dict[key]` → `await some_io()` → `shared_dict[key] = value + delta`. Between the read and write, another coroutine can modify `shared_dict[key]`.
- In a trading bot: reading current position, awaiting order placement, then updating position count — another coroutine might have updated position in the meantime.
- Severity: **P1** for read-await-write on position state, order state, or balance.

**Concurrent order placement and market data**
- If the order placement coroutine and the market data processing coroutine share state (current price, current spread, inventory), mutations during `await` can create inconsistencies.
- Example: market data updates the price to $0.55 while order placement is using the stale $0.60 price. The order is placed at the wrong level.
- Severity: **P2** — the window is small but real, especially under load.

**`asyncio.Lock()` for critical sections**
- Use `async with lock:` to protect read-modify-write sequences on shared state.
- But beware: holding a lock across an `await` that does network I/O means no other coroutine can access that state until the network call completes. This creates deadlock risk and defeats concurrency.
- Rule: hold locks for the minimum span. Read under lock, release, do I/O, re-acquire lock, validate-and-write.
- Severity: **P2** for missing locks on shared mutable state. **P2** for locks held across network I/O.

---

## Principle 6: Exception propagation in async is not what you think

*Beazley: "In synchronous code, an exception unwinds the stack. In async code, an exception can get trapped in a Future that nobody ever inspects. It's the async equivalent of a black hole."*

Exceptions in async code have three failure modes: (1) the exception is raised in a task nobody awaits (silent loss), (2) the exception is caught too broadly and suppressed, (3) `CancelledError` is caught when it shouldn't be, preventing graceful shutdown.

### What to check

**Bare `except Exception` in async loops**
- `except Exception: log_and_continue` in a trading loop is dangerous. If the exception indicates corrupt state (e.g., partial order fill, broken WS), continuing propagates the corruption.
- Each `except Exception` block must distinguish between recoverable errors (network timeout → retry) and unrecoverable errors (state corruption → halt).
- Severity: **P1** if `except Exception` covers order execution or state updates.

**Catching `CancelledError`**
- In Python 3.9+, `CancelledError` is a subclass of `BaseException`, not `Exception`. But in 3.8, it subclasses `Exception`. If the code targets 3.10+ and has `except Exception`, `CancelledError` is NOT caught — good. But `except BaseException` or explicit `except asyncio.CancelledError` can prevent task cancellation.
- If `CancelledError` is caught, it MUST be re-raised after cleanup. Swallowing it means `task.cancel()` has no effect — the task becomes unkillable.
- Severity: **P1** if `CancelledError` is swallowed anywhere. The bot cannot shut down gracefully.

**Exception groups (Python 3.11+)**
- `asyncio.TaskGroup` raises `ExceptionGroup` when subtasks fail. `except Exception` does NOT catch `ExceptionGroup`. Use `except*` syntax or handle explicitly.
- Severity: **P2** if `TaskGroup` is used without `except*` handling.

**Unhandled exceptions in callbacks**
- `loop.call_soon()`, `loop.call_later()`, and done callbacks that raise exceptions log the error but don't propagate it. The failure is invisible unless you're watching stderr.
- Severity: **P2** for critical logic in callbacks without explicit error handling.

---

## Principle 7: Resource cleanup must survive exceptions and cancellation

*Beazley: "If you acquire a resource, you must release it. Period. async with, try/finally, that's the only way. And your finally block must not block."*

An aiohttp `ClientSession` that isn't closed leaks connections. A WebSocket that isn't closed leaks file descriptors. Over hours and days, the bot runs out of file descriptors or connections and starts failing silently.

### What to check

**`async with` for all resources**
- Every `aiohttp.ClientSession()`, `websockets.connect()`, and `aiofiles.open()` must be used as an async context manager or explicitly closed in a `finally` block.
- Creating a session at module level and never closing it is a leak: `session = aiohttp.ClientSession()` at the top of a file with no corresponding `await session.close()`.
- Severity: **P1** for `ClientSession` created without cleanup — leaked connections accumulate and eventually exhaust the connection pool.

**Cleanup in `finally` must not raise**
- If `finally: await ws.close()` raises an exception (because the connection is already dead), the original exception is lost and the cleanup is incomplete.
- Wrap cleanup in `try/except`: `finally: try: await ws.close() except Exception: pass`.
- Severity: **P2** for `finally` blocks with `await` calls that can raise.

**Cancellation during cleanup**
- If a task is cancelled while inside a `finally` block that does `await`, the cleanup itself is cancelled. Use `asyncio.shield()` for critical cleanup that must complete: `finally: await asyncio.shield(cleanup())`.
- Severity: **P2** for cleanup that involves network calls (e.g., cancelling open orders on shutdown) without shielding.

**File descriptor exhaustion in long-running bots**
- Monitor open FD count: `ls /proc/self/fd | wc -l` (Linux) or `lsof -p PID | wc -l`.
- A steadily growing FD count means something is leaking. Typical culprits: unclosed WS connections from failed reconnection attempts, unclosed aiohttp response objects (`async with session.get(...) as resp:` — if you don't use `async with`, the response isn't released).
- Severity: **P2** for no FD monitoring in a bot that runs for days.

---

## Principle 8: Use structured concurrency — not fire-and-forget

*Beazley: "Structured concurrency is the async equivalent of structured programming. Goto was banished for a reason. Fire-and-forget tasks are the goto of async."*

`asyncio.TaskGroup` (Python 3.11+) ensures that all child tasks complete (or are cancelled) before the parent scope exits. Without it, orphaned tasks linger, consuming resources and mutating state after the parent has moved on.

### What to check

**Orphaned tasks**
- When a function creates tasks and then returns or raises, what happens to those tasks? If they're not awaited or cancelled, they run unsupervised.
- Example: a WS handler creates a heartbeat task and a data-processing task. If the WS handler exits due to an error, do the heartbeat and data tasks get cancelled?
- Severity: **P2** for orphaned tasks that access shared state. **P3** for orphaned tasks that are purely computational.

**TaskGroup for related work**
- If N coroutines must all succeed or all be cancelled (e.g., Binance WS + RTDS WS + order executor = the bot's core), wrap them in a `TaskGroup`.
- If any one fails, the others are cancelled automatically. Without this, one dead WS leaves the bot in a degraded state that nobody detects.
- Severity: **P2** for core bot tasks not grouped or coordinated.

**Nursery pattern for long-lived tasks**
- For tasks that must restart on failure (WS connections, background workers), implement a supervisor pattern: a parent task that starts children, monitors them, and restarts them on failure.
- Without this, each restart creates a new task without cleaning up the old one.
- Severity: **P2** for long-lived tasks with no supervision.

---

## Principle 9: `run_in_executor` is not a fix — it's an admission of defeat

*Beazley: "run_in_executor doesn't make blocking code async. It moves the blocking call to a thread pool. You still have all the problems of threads — limited pool size, no cancellation, GIL contention."*

`loop.run_in_executor(None, requests.get, url)` moves the blocking call to a thread. But the default thread pool has only `min(32, os.cpu_count() + 4)` workers. If 32 blocking calls are in flight, the 33rd blocks the pool. And the blocking call cannot be cancelled — `task.cancel()` cancels the waiting coroutine, not the thread.

### What to check

**`run_in_executor` as a crutch for blocking libraries**
- If the codebase uses `run_in_executor` for HTTP requests, the fix is replacing the library with `aiohttp`/`httpx`, not wrapping it.
- `run_in_executor` is appropriate for: CPU-bound work (hashing, JSON parsing of large payloads), calls to C extensions that release the GIL, interacting with synchronous-only libraries with no async alternative.
- Severity: **P2** for `run_in_executor` used for I/O that has an async alternative.

**Thread pool exhaustion**
- Count the maximum number of concurrent `run_in_executor` calls. If it can exceed the pool size (default ~36), tasks queue up and the event loop appears to hang.
- Set an explicit pool size: `loop.set_default_executor(ThreadPoolExecutor(max_workers=N))` and ensure N is larger than the worst-case concurrent count.
- Severity: **P2** for unbounded concurrent `run_in_executor` calls.

**No cancellation support**
- `task.cancel()` on a task wrapping `run_in_executor` does NOT cancel the underlying thread. The thread continues to completion even after the task is cancelled. If the thread mutates shared state, the mutation happens after the task is "cancelled."
- Severity: **P2** if cancelled executor tasks can mutate critical state.

---

## Principle 10: Graceful shutdown must cancel tasks in the right order

*Beazley: "Shutdown is the hardest part of any concurrent system. If you get it wrong, you corrupt state, lose data, or hang forever."*

SIGTERM arrives. The bot must: (1) stop placing new orders, (2) cancel all pending orders on the exchange, (3) optionally close open positions, (4) close WS connections, (5) close HTTP sessions, (6) flush logs, (7) exit. If this order is wrong — e.g., closing the HTTP session before cancelling orders — the cancel requests fail and orders remain open on the exchange.

### What to check

**Signal handler registration**
- Is there a `signal.signal(signal.SIGTERM, handler)` or `loop.add_signal_handler(signal.SIGTERM, handler)`? Without it, SIGTERM kills the process immediately — no cleanup, no order cancellation.
- The handler should set a flag or cancel a main task, not do cleanup directly (signal handlers have severe restrictions).
- Severity: **P1** — an unhandled SIGTERM leaves orders open on the exchange with no bot to manage them.

**Shutdown ordering**
- Is there a defined shutdown sequence? Verify: (1) new order placement is disabled first, (2) pending exchange orders are cancelled (requires HTTP session to be alive), (3) WS connections are closed, (4) HTTP sessions are closed.
- Common bug: `await session.close()` is called before `await cancel_all_orders()`, so the cancel requests fail with `ClientSession is closed`.
- Severity: **P1** for wrong shutdown order that prevents order cancellation.

**Shutdown timeout**
- Does the shutdown have a maximum duration? If cancelling orders hangs (exchange is down), the bot hangs forever in shutdown.
- Use `asyncio.wait_for(shutdown_coro(), timeout=30)` with a fallback `sys.exit(1)`.
- Severity: **P2** for shutdown with no timeout.

**Double-signal handling**
- First SIGTERM triggers graceful shutdown. Second SIGTERM (or SIGKILL after timeout) forces immediate exit.
- Is this pattern implemented? Without it, an operator who sends SIGTERM during a stuck shutdown must resort to `kill -9`, losing any partial cleanup.
- Severity: **P3**

---

## Principle 11: Memory leaks in long-running processes are bugs, not inconveniences

*Beazley: "A list that grows without bound is not a data structure. It's a timer counting down to OOM."*

A trading bot runs for days or weeks. A list that appends every tick, a dict that caches every order ID, a deque with no maxlen — all of these grow until the process is OOM-killed. The process restarts, state is lost, open orders are orphaned.

### What to check

**Unbounded collections**
- Search for `list.append()`, `dict[key] = ...`, `set.add()` in the hot path. Is there corresponding cleanup (periodic trimming, TTL eviction, `maxlen` on deques)?
- Common offender: order history stored as a growing list. After 100K orders, this list consumes significant memory and iterating it (for PnL calculations) slows the event loop.
- Severity: **P2** for unbounded collections in the hot path.

**Unclosed connections accumulating**
- Failed WS reconnections that create new connection objects without closing old ones. Each dead connection holds buffers, SSL state, and file descriptors.
- Check: does the reconnection code explicitly close the old connection before creating a new one?
- Severity: **P2** for reconnection logic that doesn't close old connections.

**Callback/listener accumulation**
- Every `emitter.on('event', handler)` without a corresponding `off()` is a leak if handlers are created per-connection or per-request.
- In asyncio: `loop.call_later()` returns a handle that must be cancelled if the scheduled callback is no longer needed.
- Severity: **P3** unless handlers hold references to large objects.

**Periodic memory monitoring**
- Does the bot log its memory usage periodically (`resource.getrusage()` or `psutil.Process().memory_info()`)?
- A slow leak is invisible without monitoring. By the time you notice symptoms (slow response, OOM), the damage is done.
- Severity: **P2** for long-running bots with no memory monitoring.

---

## Principle 12: Async generators must be closed — or they leak

*Beazley: "An async generator that isn't closed will hold its frame alive forever. It's the async equivalent of a zombie process."*

An `async for msg in ws:` loop that breaks out without closing the generator leaves the WebSocket frame (and all its local variables) alive. `async def stream_prices(): ... yield price` — if the consumer stops iterating, the generator is suspended at the `yield` with all resources held open.

### What to check

**Async generators without `async with aclosing()`**
- Any `async for item in async_generator()` must be wrapped in `async with contextlib.aclosing(gen) as stream: async for item in stream:` to guarantee cleanup.
- Without `aclosing()`, if the consumer raises an exception or breaks, `aclose()` is never called on the generator.
- Severity: **P2** for async generators that hold resources (WS connections, file handles). **P3** for pure-computation generators.

**Streaming data processing**
- For processing WS message streams: is the consumer keeping up with the producer? If the consumer is slow (e.g., does blocking work), messages queue in asyncio internals, consuming memory.
- Use bounded queues (`asyncio.Queue(maxsize=N)`) between the WS reader and the processor. When the queue is full, drop old messages or block the reader.
- Severity: **P2** for unbounded queues between WS readers and processors.

**Generator finalization timing**
- In CPython, async generators are finalized by the event loop's shutdown routine. If the loop shuts down before finalizing generators, resources leak.
- Call `await gen.aclose()` explicitly in shutdown logic, don't rely on GC.
- Severity: **P3** unless the generator holds exchange connections.

---

## Principle 13: `asyncio.gather()` is not a task group — know the difference

*Beazley: "gather() is a convenience, not a coordination primitive. It doesn't cancel siblings on failure. It doesn't enforce structure. It's the wrong tool for anything critical."*

`asyncio.gather(task_a(), task_b(), task_c())` runs all three concurrently. If `task_b` raises, by default the exception propagates and `task_a`/`task_c` are NOT cancelled — they become orphans. With `return_exceptions=True`, the exception is silently swallowed into the results list.

### What to check

**`gather()` without `return_exceptions` for critical tasks**
- If any gathered task is critical (WS stream, order executor), a failure in one should cancel the others. `gather()` doesn't do this — use `TaskGroup` instead.
- `gather(return_exceptions=True)` is worse: it turns exceptions into values, and the caller must check every result to detect failures. If the caller doesn't check, failures are invisible.
- Severity: **P2** for `gather()` on critical tasks where failure of one should stop the others.

**Result inspection after `gather(return_exceptions=True)`**
- After `results = await asyncio.gather(*tasks, return_exceptions=True)`, does the code iterate results and check `isinstance(r, Exception)`?
- If not, failed tasks are silently ignored.
- Severity: **P2** if results are used without exception checking.

**`gather()` of a dynamic task list**
- `await asyncio.gather(*[process(item) for item in large_list])` launches all tasks simultaneously. If the list is 10,000 items, this is 10,000 concurrent coroutines. Use `asyncio.Semaphore` to limit concurrency.
- Severity: **P2** for unbounded concurrent task creation.

---

## Principle 14: Logging must not block the event loop

*Beazley: "Your logging call is synchronous. Your file write is synchronous. Congratulations, you've just added a blocking call to every line of your hot path."*

The standard `logging` module's `FileHandler` and `StreamHandler` do synchronous I/O. A `logging.info()` in the WS message handler writes to disk synchronously, blocking the event loop. On a VPS with slow disk I/O, a single log line can take 1-10ms. At 100 messages per second, that's 100-1000ms of blocking per second — the event loop gets only 0-90% of the CPU.

### What to check

**Synchronous file handlers in the hot path**
- Is `logging.FileHandler` or `logging.handlers.RotatingFileHandler` used? These block on disk write.
- For high-throughput paths (WS messages, order updates): use `logging.handlers.QueueHandler` with a `QueueListener` that writes from a separate thread.
- Severity: **P2** if logging happens on every WS message with a synchronous handler.

**Log formatting cost**
- Complex log formatters (string interpolation, JSON serialization) add CPU cost that runs in the event loop. Use lazy formatting: `log.info("price=%s qty=%s", price, qty)` not `log.info(f"price={price} qty={qty}")`.
- The f-string is always evaluated. The %-format is only evaluated if the log level is enabled.
- Severity: **P3** for eager formatting. **P2** if the formatting involves serialization of large objects.

**Log rotation blocking**
- `RotatingFileHandler` renames files on rotation. On Linux, file rename is fast. But if the old log is being written to by another process (e.g., `tail -f`), the rename can block.
- Use `WatchedFileHandler` or external rotation (logrotate) with `copytruncate`.
- Severity: **P3**

---

## Principle 15: Coroutines and callbacks do not mix well — pick one and be consistent

*Beazley: "Callbacks were the old way. Coroutines are the new way. Mixing them is how you get a system nobody can debug."*

asyncio supports both callback-style (`loop.call_soon`, `add_done_callback`) and coroutine-style (`await`). Mixing them creates code where control flow is split: part of the logic is in `async def` functions (traceable, debuggable) and part is in callbacks (opaque, no stack trace, no `await` support).

### What to check

**Callbacks doing complex logic**
- Done callbacks (`task.add_done_callback(fn)`) should only do simple bookkeeping: log result, set a flag, schedule a new coroutine. If a callback does I/O, state mutation, or error handling, it should be a coroutine instead.
- The callback receives a `Task` object, not the result. It must call `task.result()` or `task.exception()`, which can raise. Unhandled exceptions in callbacks are logged to stderr and lost.
- Severity: **P2** for callbacks with complex logic. **P3** for callbacks that don't handle exceptions.

**`loop.call_soon()` vs `asyncio.create_task()`**
- `call_soon(fn)` schedules a regular function. `create_task(coro)` schedules a coroutine. If you need `await` inside the scheduled work, you must use `create_task`.
- Common bug: using `call_soon(async_fn())` — this calls the coroutine synchronously (which just creates the coroutine object) and never actually runs it.
- Severity: **P1** if `call_soon` is used with a coroutine — the coroutine never executes.

**Protocol classes with async operations**
- asyncio Protocol/Transport classes (used by some low-level WS libraries) are callback-based. If the `data_received()` callback needs to do async work (e.g., process a message, update state), it must schedule a task, not `await` directly (callbacks are not coroutines).
- Severity: **P2** for protocols that attempt blocking or async operations in callbacks.

---

## Principle 16: The GIL does not protect your async code

*Beazley: "The GIL protects CPython's internal data structures. It does NOT protect your data structures. It does NOT prevent logical race conditions."*

The GIL ensures that only one thread runs Python bytecode at a time. But asyncio is single-threaded — the GIL is irrelevant. Races in asyncio come from interleaving at `await` points, not from threads. People who think "Python is single-threaded so I don't have races" are wrong twice: asyncio has coroutine-level races, and `run_in_executor` introduces actual thread-level races.

### What to check

**Shared state between executor threads and the event loop**
- If `run_in_executor` is used, the thread function MUST NOT read or write any state that the event loop's coroutines also access. The GIL makes individual bytecode operations atomic, but multi-step operations (read-modify-write) are not atomic.
- Use `loop.call_soon_threadsafe()` to pass results from threads back to the event loop.
- Severity: **P1** for shared mutable state between executor threads and coroutines without synchronization.

**`threading.Lock` vs `asyncio.Lock`**
- `threading.Lock` blocks the OS thread (and the event loop, since asyncio runs on one thread). Never use `threading.Lock` in async code — use `asyncio.Lock`.
- Conversely, `asyncio.Lock` cannot be used in `run_in_executor` threads — it requires the event loop.
- Severity: **P1** for `threading.Lock.acquire()` in the event loop thread.

---

## Principle 17: `asyncio.Queue` is the safe way to communicate between tasks

*Beazley: "If you need to pass data between coroutines, use a Queue. It's the one data structure that's designed for concurrent access in asyncio."*

Shared mutable state + await boundaries = race conditions. Queues decouple producer and consumer: the WS reader puts messages on a queue, the processor takes from it. No shared state, no races, and the queue provides natural backpressure.

### What to check

**Direct state mutation instead of message passing**
- If the WS handler directly modifies a shared price dict, and the strategy loop reads the same dict, there's a race at every `await`. Use a queue: WS handler puts price updates on a queue, strategy loop reads from the queue.
- Severity: **P2** for shared mutable state accessed by multiple coroutines.

**Unbounded queues**
- `asyncio.Queue()` without `maxsize` is unbounded. If the producer is faster than the consumer (e.g., rapid WS messages during high volatility), the queue grows until OOM.
- Use `asyncio.Queue(maxsize=1000)` or similar. When full, `put()` blocks the producer — which is correct behavior (backpressure).
- Severity: **P2** for unbounded queues in production.

**Queue.get() without timeout**
- `await queue.get()` hangs forever if the producer dies. Use `asyncio.wait_for(queue.get(), timeout=N)` to detect dead producers.
- Severity: **P2** if the queue is the only data path for critical data (price updates, order fills).

---

## Principle 18: Event loop blocking detection must be built in, not hoped for

*Beazley: "You won't notice a 200ms event loop block until your WebSocket connections start dropping. By then, the damage is done."*

asyncio has a built-in slow callback detector: `loop.slow_callback_duration`. When a callback or coroutine step takes longer than this threshold, asyncio logs a warning. But the default threshold is 100ms, and many codebases don't enable it or monitor the logs.

### What to check

**`loop.set_debug(True)` in development**
- Debug mode enables slow callback warnings, coroutine tracking, and resource warnings. It has performance overhead and should be disabled in production, but must be used during development.
- Severity: **P3** if debug mode was never used during development.

**Custom slow-callback monitoring in production**
- In production, use a periodic task that measures event loop responsiveness: schedule a callback every 1 second and measure the actual delay. If the scheduled-at-1s callback runs at 1.3s, the loop was blocked for 300ms.
- `loop.time()` gives the event loop's internal clock — use it for precise measurement.
- Severity: **P2** for production bots with no loop-blocking detection.

**Profiling async code**
- Standard `cProfile` doesn't work well with async code (it profiles time spent in the function, not time the function blocks the loop). Use `yappi` with `set_clock_type("wall")` or asyncio-specific profilers.
- Severity: **P3** — relevant during performance investigation, not routine review.

---

## Principle 19: Semaphores, not hope, for concurrency limits

*Beazley: "If you don't limit concurrency explicitly, you'll limit it implicitly — by running out of file descriptors, memory, or API rate limits."*

Launching 100 concurrent `aiohttp` requests to Polymarket will hit rate limits, exhaust connections, and likely get the API key banned. Use `asyncio.Semaphore` to bound concurrent operations.

### What to check

**Unbounded concurrent API calls**
- When placing multiple orders or fetching multiple market states, is the concurrency bounded? `await asyncio.gather(*[place_order(o) for o in orders])` with 50 orders = 50 concurrent API requests.
- Use `semaphore = asyncio.Semaphore(5)` and `async with semaphore: await place_order(o)`.
- Severity: **P2** for unbounded concurrent API calls. **P1** if exceeding rate limits causes a ban.

**Semaphore fairness**
- `asyncio.Semaphore` is not strictly FIFO — under contention, the order of acquisition is not guaranteed. If fairness matters (e.g., order placement priority), consider a priority queue instead.
- Severity: **P3** — only relevant for latency-sensitive operations.

**Semaphore in reconnection storms**
- When multiple WS streams reconnect simultaneously, each reconnection involves DNS resolution, TLS handshake, and subscribe messages. Limit concurrent reconnections with a semaphore.
- Severity: **P2** for concurrent reconnection without limits.

---

## Principle 20: `asyncio.shield()` is rarely correct — understand what it does

*Beazley: "shield() doesn't prevent cancellation. It prevents the cancellation of the inner coroutine. The outer coroutine is still cancelled. This distinction confuses everyone."*

`await asyncio.shield(coro())` means: if the task wrapping this `await` is cancelled, the inner `coro()` keeps running, but the `await` raises `CancelledError`. The caller thinks it was cancelled, but `coro()` is still executing. This is almost never what you want.

### What to check

**`shield()` around order placement**
- If `await asyncio.shield(place_order(...))` is used: after cancellation, the order may still be placed on the exchange, but the bot's internal state doesn't reflect it. The result is a phantom order.
- Severity: **P1** if `shield()` is used for exchange operations without post-cancellation reconciliation.

**Legitimate `shield()` use cases**
- The only legitimate use: cleanup operations in `finally` blocks that must complete regardless of cancellation (see Principle 7). Example: `finally: await asyncio.shield(cancel_all_exchange_orders())`.
- Even here, the shielded coroutine should have its own timeout.
- Severity: **P2** for `shield()` used outside of `finally` blocks.

---

## Principle 21: Async context managers must handle `__aexit__` exceptions

*Beazley: "If __aexit__ raises, the original exception is lost. Your cleanup just ate your error message."*

A class implementing `__aenter__` and `__aexit__` for managing WS connections or sessions must handle exceptions in `__aexit__` carefully. If `__aexit__` does `await self.ws.close()` and the connection is already dead, the exception from `close()` replaces the original exception that triggered the `__aexit__`.

### What to check

**`__aexit__` that can raise**
- Any `await` in `__aexit__` can raise. The implementation must catch and suppress cleanup exceptions to preserve the original error.
- Pattern: `async def __aexit__(self, *args): try: await self.close() except Exception: pass`.
- Severity: **P2** for `__aexit__` implementations that don't handle cleanup exceptions.

**Nested async context managers**
- When using `async with A() as a:` → `async with B() as b:`, if `B.__aexit__` raises, `A.__aexit__` may not run (depending on implementation). Use `contextlib.AsyncExitStack` for reliable nested cleanup.
- Severity: **P3** for simple nesting. **P2** for deeply nested resource management.

---

## Principle 22: `await` does not mean "run now" — it means "yield until ready"

*Beazley: "People read 'await' as 'do this now.' It actually means 'I'm willing to wait for this, and other things can run in the meantime.' If you don't understand this, you don't understand async."*

`await asyncio.sleep(0)` does not sleep. It yields control to the event loop, allowing other ready coroutines to run, and resumes immediately after. This is useful for breaking up CPU-intensive work into cooperative chunks. Understanding this is key to reasoning about async code behavior.

### What to check

**CPU-bound work without yielding**
- A loop that processes 10,000 items without a single `await` blocks the event loop for the entire duration. Insert `await asyncio.sleep(0)` every N iterations to yield.
- Example: computing PnL across all historical trades in a single loop — if it takes 500ms, the event loop is frozen for 500ms.
- Severity: **P2** for CPU-bound loops > 100ms without yielding. **P1** if > 1 second.

**Misunderstanding `await` ordering**
- `result_a = await task_a()` then `result_b = await task_b()` is sequential, not concurrent. For concurrent execution: `result_a, result_b = await asyncio.gather(task_a(), task_b())`.
- If both are API calls taking 2 seconds each, sequential = 4 seconds, concurrent = 2 seconds. In a trading bot, 2 seconds of unnecessary latency is 2 seconds of stale data.
- Severity: **P2** for sequential awaits that should be concurrent.

---

## Principle 23: Test your async code with controlled event loops

*Beazley: "You can't test async code by just running it and hoping. You need to control the event loop — advance time, inject failures, verify ordering."*

Async bugs are timing-dependent and non-deterministic. A test that passes 99% of the time but fails 1% is worse than no test — it creates false confidence. Testing async code requires explicit control over timing and interleaving.

### What to check

**No async tests**
- Is the async code tested at all? If the test suite only tests synchronous logic and calls async functions via `asyncio.run()` without exercising concurrency, it misses all concurrency bugs.
- Use `pytest-asyncio` for async test functions. Use `asyncio.Event` and controlled yields to test interleaving.
- Severity: **P2** for untested async code paths that handle money.

**Tests that use `asyncio.sleep()` for synchronization**
- `await asyncio.sleep(0.5)` to "wait for the other task to finish" is a flaky test. Use `asyncio.Event`, `asyncio.Condition`, or `asyncio.Queue` for synchronization.
- Severity: **P3** for flaky tests. **P2** if the test suite is the only validation of async behavior.

**No fault injection**
- Does the test suite simulate: WS disconnection, API timeout, partial fills, concurrent state mutation?
- Without fault injection, the code is only tested on the happy path. All async bugs are on the unhappy path.
- Severity: **P2** for no fault injection in the test suite of a production bot.

---

## Principle 24: `async def` without `await` is a lie

*Beazley: "If your async function doesn't await anything, it's not async. It's a regular function wearing a coroutine costume. And the costume has a cost."*

An `async def` function that contains no `await` still creates a coroutine object, runs through the coroutine machinery, and adds overhead — but provides no benefit. Worse, calling it without `await` (a common mistake) returns a coroutine object that is never executed, and Python may not even warn you.

### What to check

**`async def` without `await`**
- Search for `async def` functions that have no `await` in their body. These should be plain `def` unless they're interface implementations (e.g., implementing an async protocol).
- The real danger: if someone calls `async_fn()` without `await`, the function body never runs. The return value is a coroutine object, not the function's result.
- Python 3.12+ gives a "coroutine was never awaited" warning, but only in debug mode.
- Severity: **P3** for the `async def` itself. **P1** for calling an async function without `await` — the operation is silently skipped.

**Forgetting `await` on async calls**
- `session.close()` instead of `await session.close()` — the session is never closed. `ws.send(data)` instead of `await ws.send(data)` — the message is never sent.
- Enable `PYTHONASYNCIODEBUG=1` during development to catch these.
- Severity: **P1** for missing `await` on side-effectful async calls (order placement, connection close).

---

## Principle 25: The event loop is your runtime — treat it with the same respect as your OS

*Beazley: "The event loop is an operating system for coroutines. It schedules, it dispatches, it manages resources. If you abuse it, everything breaks."*

The event loop is a single point of failure. If it's overloaded, slow, or misconfigured, every component suffers. A healthy event loop is the foundation of a healthy async application.

### What to check

**Running multiple event loops**
- `asyncio.run()` creates a new event loop. If called from within an already-running loop, it raises `RuntimeError`. But `asyncio.new_event_loop()` + `loop.run_until_complete()` can create a second loop on another thread — this is fragile and hard to debug.
- There should be exactly one event loop, created once at startup.
- Severity: **P2** for multiple event loops. **P1** if tasks in one loop reference objects from another.

**Event loop policy**
- On Linux, `uvloop` (`asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())`) provides 2-4x throughput improvement. For a trading bot with multiple WS streams and frequent I/O, this is significant.
- Not strictly a bug, but a missed optimization for production.
- Severity: **P3** for not using `uvloop` on Linux in production.

**Blocking the event loop at startup**
- Synchronous initialization (loading config from database, establishing sync HTTP connections) before `asyncio.run()` is fine. But doing it inside `async def main()` blocks the loop.
- Pattern: do all sync initialization first, then enter the async world. Or use `loop.run_in_executor` for startup tasks that must happen inside the async context.
- Severity: **P2** for blocking calls during async initialization that delay WS connections.

---

## Severity Guide

| Severity | Meaning | Action |
|----------|---------|--------|
| **P1** | The event loop freezes, data is silently lost, orders are missed or duplicated, or shutdown leaves orphaned orders. | Fix before deploying. |
| **P2** | Performance degradation, resource leaks, race conditions under specific timing, or missing safety nets. | Fix before scaling up or running for extended periods. |
| **P3** | Code quality, missed optimizations, or minor robustness gaps. | Track and fix in next iteration. |

### The Overriding Filter

Before writing any finding, apply the Beazley synthesis:

1. **Does this block the event loop?** If a single call can freeze the loop for > 100ms, it is P1. Every heartbeat misses, every timeout is delayed, every price update is stale. A blocked event loop in a trading bot is equivalent to pulling the network cable.
2. **Can this fail silently?** A task that dies without anyone noticing, a coroutine that's never awaited, an exception swallowed by `gather(return_exceptions=True)` — silent failures are worse than crashes. A crash stops trading; a silent failure continues trading with broken assumptions.
3. **Does this survive disconnection?** Every network operation will eventually fail. The code must handle the failure, not just the success. If the WS dies, the API times out, the DNS fails — does the bot detect it, back off, reconnect, and resume?
4. **Is the cleanup path as robust as the happy path?** `finally` blocks, `__aexit__`, shutdown handlers — these run under adverse conditions (exceptions, cancellation, resource exhaustion). If cleanup can raise, and that exception is unhandled, the original error is lost and resources leak.
5. **Would Beazley trust this to run unsupervised for a week?** A trading bot is a long-running process. Every resource must be released, every task must be monitored, every connection must be checked for liveness. If the answer requires "well, we restart it daily" — the code has a leak.
