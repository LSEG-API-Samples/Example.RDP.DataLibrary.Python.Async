# Concurrent Data Platform API Calls with Python Asyncio and Data Library for Python

A Jupyter notebook demonstrating how to issue multiple [LSEG Data Library for Python](https://developers.lseg.com/en/api-catalog/lseg-data-platform/lseg-data-library-for-python) API requests concurrently using Python's built-in `asyncio` module.

---

## Overview

Rather than fetching market data sequentially, this project demonstrate how to use `asyncio.gather` with Data Library Historical `get_data_async` method to send multiple historical price data concurrently.

---

## Project Structure

```
.
├── notebook/
│   ├── ld_notebook_async_gather.ipynb    # Main notebook
│   ├── lseg-data.config.json             # Data Library logging configuration
│   ├── .env                              # Platform Session credentials (not committed)
│   └── .env.example                      # Credentials template
├── images                                # Project images folder
├── .venv                                 # Project Python virtual environment
├── LICENSE.md                            # Project License
├── requirements.txt                      # Pinned Python dependencies
└── README.md
```

---

## Prerequisites

- Python 3.11 or higher
- A valid LSEG Data Platform account with:
  - API Key
  - Machine ID (username)
  - Password

---

## Setup

### 1. Create and activate a virtual environment

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

### 2. Install dependencies

```powershell
(venv)$>python -m pip install --upgrade pip
(venv)$>python -m pip install -r requirements.txt
```

### 3. Configure credentials

Copy `.env.example` to `.env` inside the `notebook/` folder and fill in your credentials:

```env
LSEG_API_KEY=your_api_key_here
LSEG_MACHINE_ID=your_machine_id_here
LSEG_PASSWORD=your_password_here
```

### 4. Run the notebook

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
- [Python Asyncio library](https://docs.python.org/3/library/asyncio.html) page.
- [A Conceptual Overview of asyncio](https://docs.python.org/3/howto/a-conceptual-overview-of-asyncio.html#a-conceptual-overview-of-asyncio) article.
- [Python's asyncio: A Hands-On Walkthrough](https://realpython.com/async-io-python/)
- [Asynchronous HTTP Requests in Python with HTTPX and asyncio](https://www.twilio.com/en-us/blog/asynchronous-http-requests-in-python-with-httpx-and-asyncio)
- [Asyncio gather function document](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather) page.
- [Asyncio TaskGroup function document](https://docs.python.org/3/library/asyncio-task.html#task-groups) page.

For any question related to this example or Data Library, please use the Developers Community [Q&A Forum](https://community.developers.refinitiv.com).