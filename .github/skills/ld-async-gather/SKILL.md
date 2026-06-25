---
name: ld-async-gather
description: "Convert LSEG Data Library synchronous single-RIC loops to concurrent async requests. Use when: migrating sequential historical_pricing.summaries single-RIC loops; fetching historical pricing for multiple RICs concurrently; applying asyncio.gather() or asyncio.TaskGroup() with historical_pricing.summaries; controlling rate limits with semaphore; handling 429 errors; implementing display_response() for async results."
argument-hint: "number of instruments or RIC list"
---

# LSEG Data Library — Async Gather Pattern

Converts sequential single-RIC `historical_pricing.summaries.Definition(...).get_data()` loops into concurrent async requests using `asyncio.gather()` or `asyncio.TaskGroup()`.

---

## When to Use

- Replacing a `for ric in INSTRUMENTS: historical_pricing.summaries.Definition(universe=ric, ...).get_data()` loop
- Fetching historical pricing for many RICs at once (latency dominated by network I/O)
- Demonstrating async performance improvement vs synchronous baseline
- Handling per-request errors without aborting the whole batch
- HTTP 429 rate-limit errors indicate too much concurrency — add a semaphore

---

## Key API Difference

| Synchronous (baseline) | Async (optimized) |
|---|---|
| `historical_pricing.summaries.Definition(universe=ric, ...).get_data()` | `historical_pricing.summaries.Definition(universe=ric, ...).get_data_async(closure=label)` |
| Returns a response object synchronously, one RIC at a time | Returns a response object; all RICs run concurrently |
| `.data.df`, `.is_success`, `.http_status` available immediately | Same fields available after `await` |
| Called in a `for` loop | Dispatched via `asyncio.gather()` or `asyncio.TaskGroup()` |

---

## Procedure

### Step 1 — Update Imports

```python
import asyncio
from lseg.data.content import historical_pricing
from lseg.data.content.historical_pricing import Adjustments, Intervals
from lseg.data._errors import LDError
import pandas as pd
pd.set_option("future.no_silent_downcasting", True)
```

Remove `import time` unless `time.perf_counter()` is still needed for timing.

### Step 2 — Convert `EVENT_ADJUSTMENTS` to Typed Enums

The `historical_pricing` Content Layer uses `Adjustments` enums, not raw strings:

```python
EVENT_ADJUSTMENTS = [
    Adjustments.EXCHANGE_CORRECTION,
    Adjustments.MANUAL_CORRECTION,
    Adjustments.CCH,
    Adjustments.CRE,
    Adjustments.RPO,
    Adjustments.RTS,
]
```

### Step 3 — Add a `display_response()` Helper

This function handles both successful and failed responses returned by `return_exceptions=True`:

```python
def display_response(data):
    """Display the result of each async API call.

    For each response:
    - Prints the exception message if the task raised a Python error.
    - Warns if the response has no closure label (unexpected type).
    - Renders the DataFrame on a successful HTTP response.
    - Prints the HTTP status code on a failed (4xx/5xx) response.
    """
    for api_response in data:
        if isinstance(api_response, Exception):
            print(f"\nTask failed with exception: {api_response}")
            continue

        if not hasattr(api_response, "closure"):
            print(f"\nUnexpected response type: {type(api_response)}")
            continue

        print(f"\nResponse received for: {api_response.closure}")

        if api_response.is_success:
            display(api_response.data.df)
        else:
            print(f"Request failed — HTTP status: {api_response.http_status}")
```

### Step 4a — Pattern A: `asyncio.gather()` (Resilient, Continue-on-Error)

Best for: processing all instruments even if some fail (e.g., invalid RICs, 404s).

```python
hist_data = []
display(Markdown("**Start the wall-clock timer...**"))
start_time = time.perf_counter()
try:
    tasks = asyncio.gather(
        *[
            historical_pricing.summaries.Definition(
                universe=ric,
                fields=INTERDAY_FIELDS,
                interval=Intervals.DAILY,
                start=START,
                end=END,
                adjustments=EVENT_ADJUSTMENTS,
            ).get_data_async(closure=ric)
            for ric in INSTRUMENTS
        ],
        return_exceptions=True,  # Collect all results; don't abort on first error
    )
    hist_data = await tasks  # pylint: disable=await-outside-async
    display_response(hist_data)
except* LDError as errors:
    for error in errors.exceptions:
        print(error)
finally:
    elapsed = time.perf_counter() - start_time
    display(Markdown(
        f"**Fetched ({len(INSTRUMENTS)}) RICs concurrently in {elapsed:0.2f} seconds.**"
    ))
```

### Step 4b — Pattern B: `asyncio.TaskGroup()` (Fail-Fast, Python 3.11+)

Best for: all-or-nothing batches where a single failure should abort the group.

```python
hist_data = []
display(Markdown("**Start the wall-clock timer...**"))
start_time = time.perf_counter()
try:
    async with asyncio.TaskGroup() as tg:
        task_handles = [
            tg.create_task(
                historical_pricing.summaries.Definition(
                    universe=ric,
                    fields=INTERDAY_FIELDS,
                    interval=Intervals.DAILY,
                    start=START,
                    end=END,
                    adjustments=EVENT_ADJUSTMENTS,
                ).get_data_async(closure=ric)
            )
            for ric in INSTRUMENTS
        ]
    hist_data = [t.result() for t in task_handles]
    display_response(hist_data)
except* LDError as errors:
    for error in errors.exceptions:
        print(f"Error: {error}")
finally:
    elapsed = time.perf_counter() - start_time
    display(Markdown(
        f"**Fetched ({len(INSTRUMENTS)}) RICs concurrently in {elapsed:0.2f} seconds.**"
    ))
```

### Step 5 — Add a Semaphore for Rate-Limit Control (HTTP 429)

If HTTP **429 Too Many Requests** errors appear, the platform is being overwhelmed. Wrap each coroutine with a semaphore to cap concurrent in-flight requests:

```python
MAX_CONCURRENT = 5  # Tune down if 429s persist; tune up if platform allows
_sem = asyncio.Semaphore(MAX_CONCURRENT)

async def fetch_with_limit(ric: str):
    async with _sem:
        return await historical_pricing.summaries.Definition(
            universe=ric,
            fields=INTERDAY_FIELDS,
            interval=Intervals.DAILY,
            start=START,
            end=END,
            adjustments=EVENT_ADJUSTMENTS,
        ).get_data_async(closure=ric)
```

Then replace the inner coroutine in `asyncio.gather()` with `fetch_with_limit(ric)`:

```python
tasks = asyncio.gather(
    *[fetch_with_limit(ric) for ric in INSTRUMENTS],
    return_exceptions=True,
)
hist_data = await tasks
```

---

## Decision: `gather()` vs `TaskGroup()`

| | `asyncio.gather()` | `asyncio.TaskGroup()` |
|---|---|---|
| On error | Continues collecting all results (`return_exceptions=True`) | Cancels remaining tasks (fail-fast) |
| Result format | `list` returned by `await tasks` | `[t.result() for t in task_handles]` after block exits |
| Error handling | Check `isinstance(r, Exception)` per item | Catch `except* LDError` ExceptionGroup |
| Python version | 3.7+ | 3.11+ |
| Best for | Mixed valid/invalid RIC lists | Trusted instrument lists, all-or-nothing |

---

## Common Issues

| Symptom | Cause | Fix |
|---|---|---|
| HTTP 429 | Too many concurrent requests | Add semaphore (Step 5), reduce `MAX_CONCURRENT` |
| `await outside async` lint warning | Jupyter runs in an event loop | Add `# pylint: disable=await-outside-async` comment |
| `AttributeError: 'str' object has no attribute 'EXCHANGE_CORRECTION'` | Raw strings passed to `adjustments=` | Use `Adjustments.*` enums (Step 2) |
| Empty `.data.df` | Invalid RIC or date range outside market data | Check `api_response.is_success` and `http_status` |
| `ExceptionGroup` not caught | `except LDError` used instead of `except*` | Change to `except* LDError` |
