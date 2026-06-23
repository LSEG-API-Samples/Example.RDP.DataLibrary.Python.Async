# Concurrent Data Platform API Calls with Python Asyncio Gather and Data Library for Python

## Overview

This article is a semi-sequel to my [Concurrent Data Platform API Calls with Python Asyncio and HTTPX](https://github.com/LSEG-API-Samples/Example.RDP.Python.Async.HTTPX) article. That article shows how to use Python and the [HTTPX](https://www.python-httpx.org/) library to make concurrent HTTP REST requests to LSEG [Data Platform](https://developers.lseg.com/en/api-catalog/refinitiv-data-platform/refinitiv-data-platform-apis) asynchronously. This project shifts away from manually sending HTTP REST requests and instead uses the easy-to-use [LSEG Data Library for Python](https://developers.lseg.com/en/api-catalog/lseg-data-platform/lseg-data-library-for-python). The Data Library for Python Historical Pricing module offers the `get_data_async` method to request historical data asynchronously, letting developers send multiple requests concurrently without blocking the process.

**Note**: This article is based on the Data Library for Python version 2.1.1 using the Platform Session. The library behavior might change in future releases.

## What are Data Platform APIs?

Let’s start with what the Data Platform APIs are. [LSEG Data Platform](https://developers.lseg.com/en/api-catalog/refinitiv-data-platform/refinitiv-data-platform-apis) (RDP APIs, also known as Delivery Platform in LSEG Real-Time) provides simple web based API access to a broad range of LSEG content. The Data Library connects and consumes data from the Data Platform when using the **Platform Session**.

RDP APIs give developers seamless and holistic access to all of the LSEG content such as Historical Pricing, Environmental Social and Governance (ESG), News, Research, etc, and commingled with their content, enriching, integrating, and distributing the data through a single interface, delivered wherever they need it.

For more detail regarding the Data Platform, please see the following APIs resources: 
- [Quick Start](https://developers.lseg.com/en/api-catalog/refinitiv-data-platform/refinitiv-data-platform-apis/quick-start) page.
- [Tutorials](https://developers.lseg.com/en/api-catalog/refinitiv-data-platform/refinitiv-data-platform-apis/tutorials) page.
- [RDP APIs: Introduction to the Request-Response API](https://developers.lseg.com/en/api-catalog/refinitiv-data-platform/refinitiv-data-platform-apis/tutorials#introduction-to-the-request-response-api) page.

That covers and overview of Data Platform APIs.

## What is Python Asyncio? And how it relate to Data Library for Python

Thats bring us to the next important library, the Python asyncio. Python [asyncio](https://docs.python.org/3/library/asyncio.html) is a library for writing concurrent code with async/await. Common asyncio interfaces for concurrent programming include:

- [asyncio.create_task(coro)](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task), which wraps one [coroutine](https://docs.python.org/3/library/asyncio-task.html#coroutine) in a [Task](https://docs.python.org/3/library/asyncio-task.html#asyncio.Task) and schedules it for execution, returning the Task object.
- [asyncio.gather(*aws)](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather), which runs [awaitables](https://docs.python.org/3/library/asyncio-task.html#asyncio-awaitables) in the aws sequence concurrently.
- [Task Groups](https://docs.python.org/3/library/asyncio-task.html#task-groups), a modern task-management API that provides a reliable way to wait for all tasks in a group to finish using [asyncio.TaskGroup](https://docs.python.org/3/library/asyncio-task.html#asyncio.TaskGroup) together with task creation APIs such as [asyncio.create_task(coro)](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task).

This article demonstrates how to use the [Data Library for Python](https://developers.lseg.com/en/api-catalog/lseg-data-platform/lseg-data-library-for-python) Content Layer Historical Pricing get_data_async method for requesting multiple RICs concurrently using asyncio gather.

The workflow uses a Platform Session connection. If you are using another session type, refer to the [Data Library Quickstart](https://developers.lseg.com/en/api-catalog/lseg-data-platform/lseg-data-library-for-python/quick-start).

## (Recap) What are Synchronous and Asynchronous Execution Models?

Before proceeding, I would like to briefly recap the synchronous and asynchronous execution models as follows.

**Synchronous** code runs tasks one at a time — each request must complete before the next one starts. The program blocks and waits at every I/O-bound call, so if a request takes 60 seconds, nothing else runs for those 60 seconds. Fine for a single request, but a real bottleneck when fetching data with many calls.

![synchronous](images/01_synchronous_simple.png)

**Asynchronous** code lets multiple tasks run concurrently. While one request is waiting for a network response, the event loop hands control to the next task instead of sitting idle.

![asynchronous](images/02_asynchronous_simple.png)

The real payoff comes when you have **many requests to make**. With `asyncio.gather()` and `asyncio.TaskGroup()`, all requests are fired concurrently so the total time is roughly that of the single slowest response — not the sum of all response times.

That covers the basic of synchronous and asynchronous executions.

## Throttling and Rate Limits 

My next point is the API calls rate limits. The Data Platform API request limits (throttles) to effectively manage and protect its service and ensure fair usage across the non-streaming content. 

An application would receive an error from the API call if an application reached or exceeds a limit (especially with the Asynchronous HTTP calls). You required to make some necessary adjustments to rectify the interaction with the API and retry the respective API call. 

Two different server errors on API request limits are: 

| **HTTP Status** | **Detail** |
| --- | --- |
| **429** | **Error Message**: too many attempts |
|  | **Description**: A per account limit where the number of requests per second is limited for each account accessing the platform. If this limit is reached, applications will receive a standard HTTP error (HTTP 429 too many requests). |
|  | **Suggestion**: Please reduce the number of requests per second and retry. |

Please find more detail regarding the Data Platform HTTP error status messages from the [RDP API General Guidelines](https://developers.lseg.com/en/api-catalog/refinitiv-data-platform/refinitiv-data-platform-apis/documentation) document page.

The Historical Pricing endpoint rate limits information is available on the **Reference** tab of the [Data Platform API Playground](https://apidocs.refinitiv.com/Apps/ApiDocs) page. The current rate limits (**As of Mar 2026**) is as follows:

![historical rate limit](images/03_historical-pricing-ratelimits.png)

## Prerequisite

You should have a basic understanding of Python’s asyncio before getting started. If you need a quick refresher, these resources are solid:

- [Python's asyncio: A Hands-On Walkthrough](https://realpython.com/async-io-python/)
- [Asyncio Architecture in Python: Event Loops, Tasks, and Futures Explained](https://dev.to/imsushant12/asyncio-architecture-in-python-event-loops-tasks-and-futures-explained-4pn3)
- [A Conceptual Overview of asyncio](https://docs.python.org/3/howto/a-conceptual-overview-of-asyncio.html)
- [Python Asynchronous I/O](https://docs.python.org/3/library/asyncio.html)

**Requirements**

Make sure you have the following set up:

- Python 3.11+
- LSEG Data Platform credentials with Historical Pricing permission:
  - Machine ID
  - Password
  - AppKey

Please your LSEG representative or account manager for the Data Platform Access.

That’s all I have to say about this article and example code prerequisite.

## Access Layer get_history vs Content Layer historical_pricing

That brings us to a big question, why use Content Layer Historical Pricing rather than `get_history` method?

The `get_history` method is part of the Library *Access Layer*. It is simple and convenient, but synchronous. Calls block execution until complete.

The `historical_pricing` module is part of the *Content Layer*. The Content Layer allows developers to access the same content as Access Layer which are a more flexible manner:

- Richer and fuller responses where available.
- Asynchronous and event-driven modes in addition to synchronous usage.
- Logical content modules for market data domains such as Level 1 Market Price Data (snapshot/streaming), News, Historical Pricing and so on.

The module lets developers set historical data query via *definition* then get data via synchronous `get_data` and asynchronous `get_data_async` methods. I am focusing on the asynchronous `get_data_async` method of the Historical Pricing module here.

## Code Walkthrough

Now we come to the code walkthrough. This article focuses primarily on the asynchronous code.

The first step is to import the required libraries. The main libraries are `lseg.data` and `asyncio`.

### Import Required Libraries

```python
import os
import asyncio
from IPython.display import Markdown, display
import lseg.data as ld
from lseg.data import session
from lseg.data.content import historical_pricing
from lseg.data._errors import LDError
from lseg.data.content.historical_pricing import Adjustments, Intervals
from dotenv import load_dotenv
import pandas as pd
pd.set_option("future.no_silent_downcasting", True)
```


### Open a Platform Session

Moving on to the next step, create a Data Library session object to authenticate, manage the connection, and retrieve data.

The code below gets the Data Platform credential from the OS environment variables. You can use the [python-dotenv](https://pypi.org/project/python-dotenv/) library to load credentials from `.env` file as well.

```python

# Retrieve Platform Session credentials from environment variables
app_key = os.getenv('LSEG_API_KEY')
username = os.getenv('LSEG_MACHINE_ID')
password = os.getenv('LSEG_PASSWORD')
# Create a platform session with provided credentials for authentication
ld_session = session.platform.Definition(
         app_key=app_key,
         grant=session.platform.GrantPassword(
             username=username,
             password=password
         ),
         signon_control=True
).get_session()

# Set this session as the default for all subsequent Data Library calls
session.set_default(ld_session)

# Open the connection to the LSEG Data Platform
ld_session.open()
# 
```

If the library can open the session successfully, you should see the **<OpenState.Opened: 'Opened'>** output message. 

The next step is creating the data request variables such as dictionary of company RICs and Name, request fields, etc. 

### Declare Instruments and Request Parameters

```python
# -- Instrument universe --------------------------------------------------------
INSTRUMENTS = {
    "NVDA.O":  "NVIDIA",
    "AAPL.O":  "Apple",
    "MSFT.O":  "Microsoft",
    "AMZN.O":  "Amazon",
    "GOOG.O":  "Alphabet",
    # ....
    "CVX.N":   "Chevron Corporation",
    "BAC.N":   "Bank of America Corporation",
    "CAT.N":   "Caterpillar Inc.",
}

# -- Date range ----------------------------------------------------------------
START = "2025-11-01T00:00:00Z"
END   = "2026-02-28T23:59:59Z"

# -- Event correction filters ---------------------------------------------------
# Only include exchange-level and manual corrections; filters out duplicates
# and administrative adjustments that would distort event counts.
EVENT_ADJUSTMENTS = [Adjustments.EXCHANGE_CORRECTION, Adjustments.MANUAL_CORRECTION]

# -- Field lists ----------------------------------------------------------------
# Defined once as constants so each list comprehension can reuse them.
EVENT_FIELDS    = ["EVENT_TYPE", "TRDPRC_1", "TRDVOL_1"]
INTRADAY_FIELDS = ["TRDPRC_1", "BID", "ASK"]
INTERDAY_FIELDS = ["BID", "ASK", "OPEN_PRC", "HIGH_1", "LOW_1", "TRDPRC_1", "NUM_MOVES", "TRNOVR_UNS"]
```

### Using asyncio.gather

That brings us to the most to the most direct and easiest way to request historical data concurrently, combine Historical Pricing `get_data_async` calls with [`asyncio.gather(*aws)`](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather) method.

**await asyncio.gather(*aws, return_exceptions=False)**

- Runs [awaitable objects](https://docs.python.org/3/library/asyncio-task.html#asyncio-awaitables) in the `aws` sequence concurrently.
- If all awaitables succeed, it returns a Python list of results in the same order as `aws`.
- `return_exceptions` controls how exceptions are handled:
  - If `False` (default): the first exception is raised immediately to the caller waiting on `gather()`. Other awaitables are not automatically cancelled and may continue running.
  - If `True`: exceptions are returned in the result list (instead of being raised immediately), alongside successful results.

In default mode (`return_exceptions=False`), your code may stop at the first error and not automatically collect outcomes from the other still-running awaitables. This can leave unfinished or uncollected task outcomes that are easy to miss. To handle this pattern safely, an application must keep task references and explicitly inspect task status/results when needed manually.

That is why many applications use `asyncio.gather(..., return_exceptions=True)` when they need complete visibility of both success and failure results in one place.

In this example, I use `historical_pricing.events.Definition`, which returns Historical Pricing Events data similar to the Data Platform `/data/historical-pricing/v1/views/events/` endpoint.

The first step is to define a `display_response` method to display returned historical data as a DataFrame.

### Helper: Display Responses Safely

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
        # Task raised a Python exception (e.g. network error, timeout)
        if isinstance(api_response, Exception):
            print(f"\nTask failed with exception: {api_response}")
            continue

        # Guard against unexpected response types
        if not hasattr(api_response, 'closure'):
            print(f"\nUnexpected response type: {type(api_response)}")
            continue

        print(f"\nResponse received for: {api_response.closure}")

        if api_response.is_success:
            display(api_response.data.df)
        else:
            # HTTP-level failure (4xx / 5xx from the platform)
            print(f"Request failed - HTTP status: {api_response.http_status}")
```


You may notice that the `display_response` method above is more defensive than the one used in [EX-2.01.02-HistoricalPricing-ParallelRequests.ipynb](https://github.com/LSEG-API-Samples/Example.DataLibrary.Python/blob/lseg-data-examples/Examples/2-Content/2.01-HistoricalPricing/EX-2.01.02-HistoricalPricing-ParallelRequests.ipynb), which only checks whether each response is successful.

```python
def display_reponse(response):
    print(response)
    print("\nReponse received for", response.closure)
    if response.is_success:
        display(response.data.df)
    else:
        print(response.http_status)
```

This `display_response`  handles Python exceptions that can appear in the returned list when using `asyncio.gather(..., return_exceptions=True)`, in addition to HTTP-level failures. This makes concurrent request handling easier to debug and safer in real applications.

### Requesting Data

Next, we group multiple calls to the `get_data_async` method with `asyncio.gather()` and run them as awaitable coroutines.

I am demonstrating with `historical_pricing.events.Definition` definition.

```python
# Convert dictionary keys to a list of RIC symbols (kept for quick inspection/debugging).
rics = list(INSTRUMENTS.keys())

# Convert dictionary items to (RIC, company) pairs so each request can carry a readable label.
list_of_rics_companies = list(INSTRUMENTS.items())

try:
    # Create a concurrent batch of event requests for the first three instruments.
    # The star-unpack passes each coroutine as a separate argument to gather.
    tasks = asyncio.gather(
        *[
            historical_pricing.events.Definition(universe=ric, fields=EVENT_FIELDS, count=5).get_data_async(closure=company)
            for ric, company in list_of_rics_companies[0:3]
        ],
        return_exceptions=True  # Collect all outcomes.
    )

    # Wait for the entire batch to finish and collect all response objects.
    historical_data = await tasks  # pylint: disable=await-outside-async

    display(Markdown("**Companies Historical Price Events**"))
    display_response(historical_data)
except* LDError as errors:
    for error in errors.exceptions:
        print(error)
```

![event definition dataframe results](/images/04_dataframe_1.png)

When sending multiple Historical Pricing Definition with **a single RIC** request, each RIC gets its own data response grouping together sequently in a Python *list* returns from `await tasks` statement.

```python
print(f" Data type is {type(historical_data)} and length is {len(historical_data)}")
```

[image here]

You can extract a specific company response by closure label.

```python
next(
    response.data.df
    for response in historical_data
    if getattr(response, "closure", None) == "NVIDIA"
)
```

[image here]

### Request Data with gather: Summaries Example

```python
try:
    tasks = asyncio.gather(
        *[
            historical_pricing.summaries.Definition(
                universe=ric,
                fields=INTRADAY_FIELDS,
                count=5,
                interval=Intervals.FIVE_MINUTES
            ).get_data_async(closure=company)
            for ric, company in list_of_rics_companies[3:6]
        ],
        return_exceptions=True
    )

    historical_data = await tasks  # pylint: disable=await-outside-async

    display(Markdown("**Companies Historical Price Intraday data (5-minute intervals)**"))
    display_response(historical_data)
except* LDError as errors:
    for error in errors.exceptions:
        print(error)
```

## How return_exceptions=True Handles Errors

With return_exceptions=True, successes and failures are returned together in one list.

### Invalid and Non-Permission RICs

```python
invalid_instrument_cases = {
    "IBM.N": "IBM",
    "INVALIDRIC.O": "Invalid Instrument",
    "JPM.N": "JPMorgan Chase & Co.",
    "ASML.AS": "ASML"
}

rics_with_errors = list(invalid_instrument_cases.keys())
list_of_rics_companies_with_errors = list(invalid_instrument_cases.items())

try:
    tasks = asyncio.gather(
        *[
            historical_pricing.summaries.Definition(
                universe=ric,
                fields=INTERDAY_FIELDS,
                count=5,
                interval=Intervals.DAILY
            ).get_data_async(closure=company)
            for ric, company in list_of_rics_companies_with_errors
        ],
        return_exceptions=True
    )

    historical_data = await tasks  # pylint: disable=await-outside-async

    display(Markdown("**Companies Historical Price Summaries with RIC error**"))
    display_response(historical_data)
except* LDError as errors:
    for error in errors.exceptions:
        print(error)
```

Typical outcomes include:

- INVALIDRIC.O: instrument not found.
- ASML.AS: user permission denied for the instrument.

### Invalid Fields

```python
EVENT_FIELDS_WITH_INVALID = EVENT_FIELDS + ["INVALID_FIELD"]
try:
    tasks = asyncio.gather(
        historical_pricing.events.Definition(
            universe="VOD.L",
            fields=EVENT_FIELDS_WITH_INVALID,
            count=5
        ).get_data_async(closure="Vodaphone"),
        return_exceptions=True
    )

    historical_data = await tasks  # pylint: disable=await-outside-async

    display(Markdown("**Companies Historical Price Events with RIC error**"))
    display_response(historical_data)
except* LDError as errors:
    for error in errors.exceptions:
        print(error)
```

The library can handle mixed valid/invalid fields in one request; invalid fields are omitted from response.data.df. You can inspect field-level errors in response.data.raw.

```python
historical_data[0].data.raw
```

Example raw error payload:

```json
{
  "code": "TS.Intraday.UserRequestError.90006",
  "message": "The universe does not support the following fields: [INVALID_FIELD]."
}
```

If all fields are invalid, the request fails.

```python
try:
    tasks = asyncio.gather(
        historical_pricing.events.Definition(
            universe="VOD.L",
            fields="INVALID_FIELD",
            count=5
        ).get_data_async(closure="Vodaphone"),
        return_exceptions=True
    )

    historical_data = await tasks  # pylint: disable=await-outside-async

    display(Markdown("**Companies Historical Price Events with RIC error**"))
    display_response(historical_data)
except* LDError as errors:
    for error in errors.exceptions:
        print(error)
```

## Can Events and Summaries Be Mixed in One gather Call?

Yes. Events and Summaries requests can be run concurrently in a single gather call.

See the parallel requests example in the official content-layer samples:
https://github.com/LSEG-API-Samples/Example.DataLibrary.Python/blob/lseg-data-examples/Examples/2-Content/2.01-HistoricalPricing/EX-2.01.02-HistoricalPricing-ParallelRequests.ipynb

## Close the Session

```python
# Close the session to release resources.
ld_session.close()
ld.close_session()
```

## What About List-of-RIC Requests?

Historical Pricing definition universe accepts both single RIC and list-of-RIC inputs.

Single-RIC approach (recommended):

- Each request returns its own dataframe and raw JSON.
- Success and failure handling is straightforward per instrument.

List-of-RIC approach:

- One request returns a multi-index dataframe with all RICs combined.
- Error handling and per-instrument parsing are harder.

Recommendation: Use multiple single-RIC requests with asyncio.gather for better control and diagnostics.

## Summary: Data Library Historical Pricing with Asyncio Gather

asyncio.gather(..., return_exceptions=True) is practical for concurrent batch requests when you need full visibility of all outcomes.

### What it does

- Runs all request coroutines concurrently.
- Returns a result list in input order.
- Includes both successful responses and exceptions.

### Why this is useful

- Process valid instruments even when some requests fail.
- Centralize error handling for batch workflows.
- Build clearer logs and user-facing reports.

### How to process results safely

- Iterate each result item.
- If it is an exception, record/report it.
- If it is a response, process response.data.df.

### Good use cases

- Best-effort batch requests across many RICs.
- Monitoring jobs where partial results are useful.
- Exploratory runs where both data and errors are needed.

### Performance note

See notebook/ld_notebook_gather_performance.ipynb for a 30-instrument interday performance demonstration.

The demo uses response.data.raw to display raw JSON output. If you use response.data.df, additional processing time is needed to build DataFrames from JSON.

### Important note

return_exceptions=True does not hide errors. It returns errors as list items, so your code must explicitly handle both successes and exceptions.

## Is asyncio.gather the Only Concurrency Option?

No. asyncio.gather is common, but not the only option.

Alternatives:

- asyncio.create_task(...) plus explicit await.
- asyncio.as_completed(...) to process results as tasks finish.
- asyncio.wait(...) for lower-level coordination and timeouts.
- asyncio.to_thread(...) or executors for blocking work.
- asyncio.TaskGroup (Python 3.11+) for structured concurrency.

TaskGroup is a modern approach often compared with gather for safer task lifecycle management.


### What Next?

Please wait for how to use Data Library Historical Pricing `get_data_async` with `asyncio.TaskGroup` in the next article.