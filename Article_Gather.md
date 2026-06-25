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

If the library can open the session successfully, you should see the **<OpenState.Opened: 'Opened'>** output message. The library automatic manage the authentication, access token, refresh token, etc. for an application.

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

### Using asyncio.gather with return_exceptions = True

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
        return_exceptions=True  # Prevent gather from raising immediately on the first exception; we want to collect all results.
    )

    # Wait for the entire batch to finish and collect all response objects.
    # Default gather behavior: if any task raises an exception, it is raised at this await line.
    historical_data = await tasks  # pylint: disable=await-outside-async

    # Display a section header before printing each response output.
    display(Markdown("**Companies Historical Price Events**"))
    # Show each response DataFrame on success; otherwise print the HTTP status code.
    display_response(historical_data)
except* LDError as errors:
    for error in errors.exceptions:
        print(error)
```

The result is as follows:

![event definition dataframe results](/images/04_dataframe_1.png)

Please be noticed that when sending multiple Historical Pricing Definition with **a single RIC** request, each RIC gets its own data response grouping together sequently in a Python *list* returns from `await tasks` statement.

```python
print(f" Data type is {type(historical_data)} and length is {len(historical_data)}")
```

![datatype is list when length of number of results](/images/05_dataframe_2.png)

You can extract a specific company response by closure label.

```python
next(
    response.data.df
    for response in historical_data
    if getattr(response, "closure", None) == "NVIDIA"
)
```

![NVIDIA dataframe](/images/06_dataframe_3.png)

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

![Intraday data example](/images/07_dataframe_4.png)

### How return_exceptions=True Handles Errors

Now, what about what if there are errors occur?  With `return_exceptions=True` option, successes and failures are returned together in one list.

When using `asyncio.gather` method with `return_exceptions=True` option, the errors and exceptions are returns in the result list along side the success ones. 


#### Invalid and Non-Permission RICs

I am demonstrating with the invalid RIC code `INVALID_RIC` and non-permission RIC (`ASML.L` for ASML Holding, your permission may be different) requests.

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

![invalid and non-permission RIC request](/images/08_dataframe_5.png)

You can see that the results include both successful responses and error messages:

- `INVALIDRIC.O` returns `The universe is not found.. Requested ric: INVALIDRIC.O` message, which means the instrument was not found.
- `ASML.AS` returns `User has no permission.. Requested ric: ASML.AS` message, which means the user does not have permission to access that instrument.

These error messages appear alongside the historical data returned for the successful requests.

#### Invalid Fields

Now let's see how the library handles invalid fields with the `asyncio.gather`.

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

![request invalid fields](/images/09_dataframe_6.png)

The library can handle mixed valid/invalid fields in one request; invalid fields are omitted from response.data.df. You can inspect field-level errors in `response.data.raw` statement which give you the raw JSON response message.

```python
historical_data[0].data.raw
```

Example raw error payload:

```json
{'universe': {'ric': 'VOD.L'},
 'adjustments': ['exchangeCorrection', 'manualCorrection'],
 'defaultPricingField': 'TRDPRC_1',
 'qos': {'timeliness': 'delayed'},
 'headers': [{'name': 'DATE_TIME', 'type': 'string'},
  {'name': 'EVENT_TYPE', 'type': 'string'},
  {'name': 'TRDPRC_1', 'type': 'number', 'decimalChar': '.'},
  {'name': 'TRDVOL_1', 'type': 'number', 'decimalChar': '.'}],
 'data': [['2026-06-17T07:02:03.966000000Z', 'trade', 110.65, 2.69317668],
  ['2026-06-17T07:02:03.966000000Z', 'trade', 110.65, 2.0876638],
  ['2026-06-17T07:02:03.940000000Z', 'trade', 110.55, 201],
  ['2026-06-17T07:02:03.939000000Z', 'trade', 110.55, 2000],
  ['2026-06-17T07:02:02.919000000Z', 'trade', 110.629, 4488]],
 'status': {'code': 'TS.Intraday.UserRequestError.90006',
  'message': 'The universe does not support the following fields: [INVALID_FIELD].'},
 'meta': {'blendingEntry': {'headers': [{'name': 'COLLECT_DATETIME',
     'type': 'string'},
    {'name': 'RTL', 'type': 'number', 'decimalChar': '.'},
    {'name': 'SOURCE_DATETIME', 'type': 'string'},
    {'name': 'SEQNUM', 'type': 'string'}],
   'data': [['2026-06-17T07:17:01.821000000Z',
     17344,
     '2026-06-17T07:17:01.821000000Z',
     '389367']]}}}
```

You see that the error is available in raw data result from the platform. You can use the raw information to inform users if you need.

```json
{
  "code": "TS.Intraday.UserRequestError.90006",
  "message": "The universe does not support the following fields: [INVALID_FIELD]."
}
```

Please note that if you send a request with only invalid fields (either one invalid field or a list of all invalid fields), the request fails and returns an error to the application.

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

![single invalid field request](/images/10_dataframe_7.png)

That covers how the `return_exception=True` option and Data Library handle errors.

### Can Events and Summaries Be Mixed in One gather Call?

Off cause, you can. Please see an example (with `return_exception=False`) in the [Content layer - How to send parallel requests](https://github.com/LSEG-API-Samples/Example.DataLibrary.Python/blob/lseg-data-examples/Examples/2-Content/2.01-HistoricalPricing/EX-2.01.02-HistoricalPricing-ParallelRequests.ipynb) example on GitHup repository.

That is all I want to say about the Data Library Historical Pricing with Asyncio Gather method.

Now we come to the last section of the code, you can close the session with the following statements.

## Close the Session

```python
# Close the session to release resources; in a long-running application, consider keeping the session open and reusing it for subsequent API calls instead.
ld_session.close()
ld.close_session()
```

## What About List-of-RIC Requests?

The Historical Pricing definitions universe parameter accept both single-RIC and list-of-RICs inputs.

**Single-RIC approach** (recommended): Each request returns its own dataframe and raw json response, making it easy to handle successes and failures individually.

**List-of-RICs approach**: A single request returns a [multi-index](https://pandas.pydata.org/docs/user_guide/advanced.html#multiindex-advanced-indexing) dataframe with data from all RICs combined along with an array of JSON data. This is harder to manage and parse errors per individual instrument.

**Recommendation**: Use multiple single-RIC requests with `asyncio.gather()` for better data handling, as each instrument’s success or failure can be handled independently.

## Summary: Data Library Historical Pricing with Asyncio Gather

That brings us to a summary of using Asyncio Gather method. The `asyncio.gather(..., return_exceptions=True)` pattern is practical for concurrent batch requests when you need full visibility of all outcomes (success and fail).

### What it does

- Runs all request coroutines concurrently.
- Returns one result list in the same order as the input coroutines.
- Keeps successful responses and exceptions together in that list, instead of failing immediately on the first error.

### Why this is useful

- You can still process valid instruments even when some requests fail.
- Error handling is simpler for batch workflows because all outcomes are collected in one place.
- It is easier to build clear logs and user-friendly reports from a single result list.

### How to read the results safely

- Check each item in the returned list.
- If the item is an exception, record or print the error message.
- If the item is a successful response, process `response.data.df` as usual.

### Good use cases

- Best-effort batch requests across many RICs.
- Monitoring jobs where partial data is still valuable.
- Exploratory workflows where you want both data and errors in one run.

### Performance note

For a performance comparison, refer to the [Historical Pricing get_data_async with Asyncio.Gather Performance](https://github.com/LSEG-API-Samples/Example.RDP.DataLibrary.Python.Async/blob/main/notebook/ld_notebook_gather_performance.ipynb) and [Data Library Get History Synchronous Performance](https://github.com/LSEG-API-Samples/Example.RDP.DataLibrary.Python.Async/blob/main/notebook/ld_notebook_gethistory_performance.ipynb) examples, both of which retrieve interday historical data for 30 instruments.

Please note that both examples measure retrieval time only, excluding display overhead.

**Historical Pricing get_data_async with Asyncio.Gather Performance**

![Historical Pricing get_data_async with Asyncio.Gather Performance](/images/11_gather_perf.png)

**Data Library Get History Synchronous Performance**

![Data Library Get History Synchronous Performance](/images/12_get_history_sync_perf.png)

### Important note

The `return_exceptions=True` option does not hide errors. It returns errors as list items, so your code must explicitly handle both successes and exceptions.

## Is asyncio.gather the Only Concurrency Option?

No. While `asyncio.gather()` method is widely used, but it is not the only option for running concurrent tasks.

Depending on your application requirements, you can also use:
- `asyncio.create_task(...)` + explicit `await`: start tasks immediately and await them when appropriate.
- `asyncio.as_completed(...)`: process results as each task finishes.
- `asyncio.wait(...)`: apply lower-level coordination, such as timeouts or partial completion.
- `asyncio.to_thread(...)` / executors: move blocking I/O or CPU-intensive work outside the event loop.
- `asyncio.TaskGroup` (Python 3.11+): use structured concurrency with safer and clearer task lifecycle management.

Among these approaches, `TaskGroup` is now a common choice and is frequently compared with `gather` in modern asyncio design discussions for safer task lifecycle management.

That’s all I have to say about using the Historical Pricing `get_data_async` method with the Python `asyncio.gather()` method.

### What Next?

Please wait for how to use Data Library Historical Pricing `get_data_async` with `asyncio.TaskGroup` in the next article.

## Should I use Data Library or the manual HTTP REST API Coding?

Before I finish, there is one point lef, should you use the Data Library or the manual HTTP REST coding? 

If you are using Python, C#/.NET, or TypeScript, the Data Library offers the following advantages over working directly with the HTTP REST APIs:

1. The Library automatically manages Data Platform authentication and sessions for you, so you do not need to handle sign-in, session expiration, or access-token refresh manually.
2. The Library provides developer-friendly interfaces for sending HTTP data requests. These interfaces range from simple one-line methods in the Access Layer, to richer methods in the Content Layer for more advanced use cases, to lower-level Delivery Layer methods that let you control headers, URLs, parameters, and request bodies while still handling authentication for the application.

However, if you prefer to manage authentication and sessions yourself, or if you are using another programming language such as Java, Go, Rust, Ruby, or C++, the Data Platform HTTP REST APIs are also straightforward and easy to use.

That covers all I wanted to say today. 

