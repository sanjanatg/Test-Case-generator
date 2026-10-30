# Test Case Generator

A local-first web app that turns software requirements into an editable QA test case matrix and exports the reviewed result as an Excel workbook.

## Features

- Upload PDF, DOCX, or plain-text requirement documents.
- Extract text in the app process; files are not saved by this application.
- Generate structured test cases through one OpenAI-compatible API client, with presets for OpenAI, Google Gemini, Ollama, and custom compatible servers.
- Review and edit the generated matrix, including execution fields, before export.
- Export a formatted `.xlsx` workbook with wrapped text, filters, and a frozen header.
- Add or remove rows in the editable grid.
- Start without an API key configured; generation reports how to set one when needed.
- Switch between coordinated light and dark themes from the Streamlit app menu.

## Requirements

- Python 3.10 or newer.
- A key for the selected cloud AI endpoint. The app can be opened and used to edit a matrix without one; generation through cloud providers needs a key.
- For Ollama, install and run Ollama locally and download a model first. Ollama does not need an API key.
- Network access from the app machine to the selected endpoint when generating cases.

- To comfortably support 10 concurrent users on a local server executing a deep-reasoning model like deepseek-r1:14b, your primary hardware bottleneck is GPU VRAM capacity and memory bandwidth.
- Because your architecture offloads file parsing and UI rendering to the client browser, the server only processes raw text tokens. However, reasoning models generate large amounts of internal <think> tokens, making highly optimized serving software and specialized hardware essential.

1. Target Server Hardware Requirements

To prevent requests from queuing up and timing out when 10 QA engineers click "Generate Suite" at the exact same moment, you need dual modern enterprise GPUs or a dedicated AI workstation setup.

Component	Minimum Specification (Will feel sluggish)	Recommended Specification (Comfortable & Fast)
GPU	1× NVIDIA RTX A6000 (48GB VRAM)	2× NVIDIA RTX 4090 (48GB combined VRAM) or 1× NVIDIA H100 NVL / A100 (80GB)
GPU Architecture	Ampere (PCIe Gen 4)	Ada Lovelace or Hopper (PCIe Gen 5) — Crucial for high-speed tensor core computations during long reasoning chains.
System RAM	64 GB DDR4	128 GB DDR5 (Helps handle prompt caching over system memory swaps if needed).
CPU	8-Core AMD Ryzen / Intel Core i7	16-Core to 32-Core AMD EPYC or Threadripper (High PCIe lane count to run dual GPUs at full speed).


### Why VRAM matters for 10 concurrent users:

• The Model Footprint: The deepseek-r1:14b model requires roughly 9GB to 12GB of VRAM just to sit in memory.
• The KV Cache Footprint: As 10 users stream reasoning steps simultaneously, the server must hold the history of all active conversations in VRAM (the Key-Value Cache). Long reasoning traces can scale the VRAM needs up dramatically. 48GB of total VRAM provides the comfortable safety buffer needed to prevent out-of-memory (OOM) crashes.

## Setup

Open PowerShell in this folder and run:

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
Copy-Item .env.example .env
```

Open `.env` in a text editor and choose `LLM_PROVIDER`: `OpenAI`, `Gemini`, `Ollama`, or `Custom OpenAI-compatible`. Set the matching model, endpoint, and key values in `.env.example`. Keep `.env` private; it is excluded from Git. The provider, API base URL, model, and key can also be selected or edited in the app sidebar for the current session.

### Provider examples if cloud hosted service 

| Provider | API base URL | Model | Key |
| --- | --- | --- | --- |
| OpenAI | `https://api.openai.com/v1` | `gpt-4o-mini` | OpenAI API key |
| Gemini | `https://generativelanguage.googleapis.com/v1beta/openai/` | `gemini-3.8-flash` | Google AI Studio key |
| Ollama | `http://localhost:11434/v1` | Installed model tag, such as `llama3.2` | Not required |
| Custom | Your server's OpenAI-compatible `/v1` URL | Model name served there | If required by the server |

The app uses the same OpenAI-compatible Chat Completions interface for each preset. Gemini documents compatibility with the OpenAI library and its own base URL; Ollama provides a local OpenAI-compatible endpoint. Other model families, including Llama, work when served behind Ollama or another compatible server. Direct provider APIs that do not implement this interface need a separate adapter.

Run the application:

```powershell
streamlit run app.py
```

Streamlit prints the local address to open in your browser (usually `http://localhost:8501`).

## Switching themes

Use the app menu in the top-right corner, open **Settings**, and select **Light** or **Dark** under **Theme**. The chosen appearance applies to the app, including its sidebar and editable matrix.

## Using the app

1. Upload a `.pdf`, `.docx`, or `.txt` file from the sidebar.
2. Choose the approximate number of cases and select **Generate test cases**.
3. Review and edit the generated rows. The AI leaves execution-only values for the tester.
4. Select **Export Excel** to download the current matrix.

The first version uses the column names from the supplied spreadsheet header, including **Test I/P**, **Pending Rational**, **Remarks**, and **Machine Name**. The alternative **Test VP** schema mentioned in the notes is not used in this workbook.

## Data handling and privacy

The app does not implement a database or upload archive. Uploaded file bytes are read into memory for text extraction, and the resulting text and edits are held in the Streamlit session while it is active. When the user requests AI generation, extracted requirement text is sent to the selected endpoint. Do not use a public endpoint for confidential requirements unless your organization has approved that data flow. The app does not intentionally log the requirement text, but the AI provider or infrastructure may have its own logging and retention policies.

Scanned PDFs without a text layer are not OCR-processed. The current single-request text limit is 80,000 characters; split larger documents into relevant sections before uploading.

## Project files

| File | Purpose |
| --- | --- |
| `app.py` | Streamlit interface, document extraction, generation, and workbook export |
| `requirements.txt` | Python dependencies |
| `.env.example` | Blank API key and endpoint configuration template |
| `software requirement.txt` | Supplied product and architecture notes |
| `Requirement Doc Ex.docx` | Supplied example requirements document |

## Current limitations

- This is a single-user local prototype. It has no authentication, shared projects, or persistent storage.
- Long documents are capped rather than automatically chunked.
- OCR for scanned documents and requirement-to-test traceability links are not included yet.
- Model output should be reviewed by a QA engineer before use; generated tests are suggestions, not execution evidence.
