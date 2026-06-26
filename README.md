# Concurrent Data Platform API Calls with Python Asyncio and Data Library for Python

- Version: 1.0
- Last update: June 2026
- Environment: Python + JupyterLab + Data Platform Account
- Prerequisite: Data Platform access/entitlements

---

## Overview

This project is a semi-sequel to my [Concurrent Data Platform API Calls with Python Asyncio and HTTPX](https://github.com/LSEG-API-Samples/Example.RDP.Python.Async.HTTPX) project. That project shows how to use Python and the [HTTPX](https://www.python-httpx.org/) library to make concurrent HTTP REST requests to LSEG [Data Platform](https://developers.lseg.com/en/api-catalog/refinitiv-data-platform/refinitiv-data-platform-apis) asynchronously. This project shifts away from manually sending HTTP REST requests and instead uses the easy-to-use [LSEG Data Library for Python](https://developers.lseg.com/en/api-catalog/lseg-data-platform/lseg-data-library-for-python). The Data Library for Python Historical Pricing module offers the `get_data_async` method to request historical data asynchronously, letting developers send multiple requests concurrently without blocking the process.

There is already a [Content layer - How to send parallel requests](https://github.com/LSEG-API-Samples/Example.DataLibrary.Python/blob/lseg-data-examples/Examples/2-Content/2.01-HistoricalPricing/EX-2.01.02-HistoricalPricing-ParallelRequests.ipynb) example on GitHup. However, this project provides a more in-depth exploration of making parallel requests using asyncio, offering additional details and greater flexibility beyond what is covered in the original example.


**Note**: This project is based on the Data Library for Python version 2.1.1. The library behavior might change in future releases.


## Throttling and Rate Limits 

The Data Platform API request limits (throttles) to effectively manage and protect its service and ensure fair usage across the non-streaming content. 

An application would receive an error from the API call if an application reached or exceeds a limit (especially with the Asynchronous HTTP calls). You required to make some necessary adjustments to rectify the interaction with the API and retry the respective API call. 

Two different server errors on API request limits are: 

| **HTTP Status** | **Detail** |
| --- | --- |
| **429** | **Error Message**: too many attempts |
|  | **Description**: A per account limit where the number of requests per second is limited for each account accessing the platform. If this limit is reached, applications will receive a standard HTTP error (HTTP 429 too many requests). |
|  | **Suggestion**: Please reduce the number of requests per second and retry. |

Please find more detail regarding the Data Platform HTTP error status messages from the [RDP API General Guidelines](https://developers.lseg.com/en/api-catalog/refinitiv-data-platform/refinitiv-data-platform-apis/documentation) document page.

The Historical Pricing endpoint rate limits information is available on the **Reference** tab of the [Data Platform API Playground](https://apidocs.refinitiv.com/Apps/ApiDocs) page. The current rate limits (**As of Mar 2026**) is as follows:

![historical rate limit](images/05_historical-pricing-ratelimits.png)

---

## Prerequisites

- Python 3.11+
- LSEG Data Platform credentials with Historical Pricing permission:
  - Machine ID
  - Password
  - AppKey

Please your LSEG representative or account manager for the Data Platform Access

---

## Project Structure

```
.
├── notebook/
│   ├── ld_notebook_async_gather.ipynb              # Main notebook
│   ├── ld_notebook_gather_performance.ipynb        # Asyncio Gather performance notebook
│   ├── ld_notebook_gethistory_performance.ipynb    # Access Layer get_history performance notebook
│   ├── lseg-data.config.json                       # Data Library logging configuration
│   ├── .env                                        # Platform Session credentials (not committed)
│   └── .env.example                                # Credentials template
├── images                                          # Project images folder
├── .venv                                           # Project Python virtual environment
├── LICENSE.md                                      # Project License
├── requirements.txt                                # Pinned Python dependencies
├── Article_Gather.md                               # The Asyncio Gather with Data Library article document
└── README.md                                       # The Project README file
```

---

## Prerequisites

- Python 3.11 or higher
- A valid LSEG Data Platform account with:
  - API Key
  - Machine ID (username)
  - Password

---

## Included Notebooks

| Notebook | Description |
|---|---|
| [ld_notebook_async_gather.ipynb](notebook/ld_notebook_async_gather.ipynb) | Demonstrates how to request multiple RICs concurrently using the Data Library Historical Pricing `get_data_async` method combined with `asyncio.gather()`. Covers Events and Summaries definitions, error handling for invalid RICs and fields, and how `return_exceptions=True` keeps all results — successes and failures — in one place. |
| [ld_notebook_gather_performance.ipynb](notebook/ld_notebook_gather_performance.ipynb) | Measures performance of concurrent Historical Pricing Interday requests using `get_data_async` with `asyncio.gather()`, including controlled throttling with `asyncio.Semaphore`. |
| [ld_notebook_gethistory_performance.ipynb](notebook/ld_notebook_gethistory_performance.ipynb) | Measures baseline performance of the synchronous Access Layer `get_history` method for equivalent interday retrieval, used for side-by-side comparison with the async gather approach. |

---

## Project Setup

1. Create and activate a virtual environment

```powershell
# Windows (PowerShell)
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

```bash
# macOS / Linux
python3 -m venv .venv
source .venv/bin/activate
```

2. Install dependencies

```powershell
(venv)$>python -m pip install --upgrade pip
(venv)$>python -m pip install -r requirements.txt
```

3. Configure credentials

Copy `.env.example` to `.env` inside the `notebook/` folder and fill in your credentials:

```env
LSEG_API_KEY=your_api_key_here
LSEG_MACHINE_ID=your_machine_id_here
LSEG_PASSWORD=your_password_here
```

4. Run the notebook

Open `notebook/ld_notebook_async_gather.ipynb` in JupyterLab or VS Code and run the cells in order.

```powershell
(venv)$>notebook\jupyter lab notebook
```

---

## Dependencies

| Package | Purpose |
|---|---|
| `lseg-data` | LSEG Data Library for Python — market data access |
| `jupyterlab` | Interactive notebook environment |
| `python-dotenv` | Load credentials from `.env` without hardcoding them |

See `requirements.txt` for pinned versions.

---

## License

See [LICENSE.md](LICENSE.md).

---

## Reference

You can find more detail regarding the Data Library and Python Asyncio from the following resources:

- [LSEG Data Library for Python](https://developers.lseg.com/en/api-catalog/lseg-data-platform/lseg-data-library-for-python) on the [LSEG Developer Community](https://developers.lseg.com/) website.
- [Data Library for Python - Reference Guide](https://developers.lseg.com/en/api-catalog/lseg-data-platform/lseg-data-library-for-python/documentation#reference-guide)
- [The Data Library for Python  - Quick Reference Guide (Access layer)](https://developers.lseg.com/en/article-catalog/article/the-data-library-for-python-quick-reference-guide-access-layer) article.
- [Essential Guide to the Data Libraries - Generations of Python library (EDAPI, RDP, RD, LD)](https://developers.lseg.com/en/article-catalog/article/essential-guide-to-the-data-libraries) article.
- [Upgrade from using Eikon Data API to the Data library](https://developers.lseg.com/en/article-catalog/article/Upgrade-from-using-Eikon-Data-API-to-the-Data-library) article.
- [Data Library for Python Examples on GitHub](https://github.com/LSEG-API-Samples/Example.DataLibrary.Python) repository.
- [Comparison of Data Library for Python VS Python/requests direct call for the Delivery Platform (RDP) application](https://developers.lseg.com/en/article-catalog/article/comparison-of-data-library-python-vs-python-requests) article.
- [Python Asyncio library](https://docs.python.org/3/library/asyncio.html) page.
- [A Conceptual Overview of asyncio](https://docs.python.org/3/howto/a-conceptual-overview-of-asyncio.html#a-conceptual-overview-of-asyncio) article.
- [Python's asyncio: A Hands-On Walkthrough](https://realpython.com/async-io-python/)
- [Asyncio gather function document](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather) page.
- [Asyncio TaskGroup function document](https://docs.python.org/3/library/asyncio-task.html#task-groups) page.

For any question related to this example or Data Library, please use the Developers Community [Q&A Forum](https://community.developers.refinitiv.com).