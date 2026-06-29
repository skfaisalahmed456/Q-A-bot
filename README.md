# 📄 Document Q&A Bot — RAG Pipeline

A Retrieval-Augmented Generation (RAG) system that answers natural language questions strictly from a collection of local documents (PDF, DOCX, TXT). Built with LangChain, FAISS, HuggingFace Embeddings, and Google Gemini 2.5 Flash.

> If the answer is not present in the documents, the bot replies:
> *"I couldn't find this information in the uploaded documents."*

---

## 🎯 What It Does

- Ingests **5 documents** across PDF, DOCX, and TXT formats
- Chunks, embeds, and stores them in a **persistent FAISS vector index**
- Accepts natural language questions and retrieves the most relevant chunks
- Generates grounded answers using **Gemini 2.5 Flash** with source citations
- Runs end-to-end in **Google Colab** as an interactive notebook

---

## 🗂️ Project Structure

```
document-qa-bot/
│
├── docs/                        # Knowledge base documents
│   ├── Python.pdf
│   ├── AI.pdf
│   ├── Machine_Learning.pdf
│   ├── Database.docx
│   └── Cloud_Computing.txt
│
├── vectorstore/                 # Persisted FAISS index (auto-generated)
│   ├── index.faiss
│   └── index.pkl
│
├── Document_QA_RAG.ipynb        # Main notebook (run top to bottom)
└── README.md
```

---

## 🧰 Tech Stack

| Component | Library / Tool | Version |
|---|---|---|
| Notebook environment | Google Colab | — |
| LLM | Google Gemini 2.5 Flash via `langchain-google-genai` | latest |
| Embeddings | `sentence-transformers/all-MiniLM-L6-v2` | via HuggingFace |
| Vector store | FAISS (CPU) | `faiss-cpu` |
| LangChain core | `langchain`, `langchain-core`, `langchain-community` | 1.x |
| Legacy chains | `langchain-classic` | latest |
| Text splitter | `langchain-text-splitters` | latest |
| PDF loader | `PyPDFLoader` (pypdf) | latest |
| DOCX parser | `python-docx` | latest |
| Gemini SDK | `google-generativeai` | latest |

---

## 🏗️ Architecture Overview

```
 ┌─────────────┐     ┌──────────────┐     ┌───────────────────┐
 │  Documents  │────▶│   Chunking   │────▶│  HuggingFace      │
 │ PDF/DOCX/TXT│     │ Recursive    │     │  Embeddings       │
 └─────────────┘     │ CharSplitter │     │  (all-MiniLM-L6)  │
                     └──────────────┘     └────────┬──────────┘
                                                   │
                                          ┌────────▼──────────┐
                                          │   FAISS Vector    │
                                          │   Store (disk)    │
                                          └────────┬──────────┘
                                                   │
 ┌─────────────┐     ┌──────────────┐     ┌────────▼──────────┐
 │  User Query │────▶│ Embed Query  │────▶│  MMR Retrieval    │
 └─────────────┘     │ (same model) │     │  top-k=6 chunks   │
                     └──────────────┘     └────────┬──────────┘
                                                   │
                                          ┌────────▼──────────┐
                                          │  Gemini 2.5 Flash │
                                          │  (answer + cite)  │
                                          └────────┬──────────┘
                                                   │
                                          ┌────────▼──────────┐
                                          │  Answer + Sources │
                                          └───────────────────┘
```

**Pipeline steps:**

1. **Indexing** (run once) — load documents → chunk → embed → store to FAISS
2. **Querying** (interactive) — embed query → MMR similarity search → pass context to Gemini → return answer with citations

---

## ✂️ Chunking Strategy

**Method:** `RecursiveCharacterTextSplitter`

| Parameter | Value |
|---|---|
| `chunk_size` | 1000 characters |
| `chunk_overlap` | 150 characters |
| Separators | `\n\n`, `\n`, `. `, ` `, `""` |

**Why this strategy?**
Recursive character splitting tries larger separators first (paragraph breaks, then line breaks, then sentence endings) before falling back to character splits. This keeps semantically related sentences together while respecting the chunk size limit. The 150-character overlap prevents context loss at chunk boundaries — critical for questions whose answers span paragraph breaks. Fixed-size splitting was avoided because it can cut mid-sentence; sentence-based splitting (NLTK) was avoided due to Colab environment conflicts.

---

## 🔢 Embedding Model

**Model:** `sentence-transformers/all-MiniLM-L6-v2`

- Produces 384-dimensional dense vectors
- Runs entirely locally — no API cost for embedding
- Fast inference suitable for Colab CPU
- Strong performance on semantic similarity for English text
- Embeddings are L2-normalized for consistent cosine similarity scores

---

## 🗄️ Vector Database

**Choice:** FAISS (Facebook AI Similarity Search)

- Runs in-process — no separate server to manage
- Persists to disk (`vectorstore/`) so re-indexing is not required on every run
- Supports MMR (Maximal Marginal Relevance) retrieval out of the box via LangChain
- Scales well for small-to-medium document collections (< 100K chunks)

**Retrieval settings:**

| Parameter | Value | Purpose |
|---|---|---|
| `search_type` | `mmr` | Diversifies results across documents |
| `k` | 6 | Chunks passed to LLM as context |
| `fetch_k` | 20 | MMR candidate pool size |
| `lambda_mult` | 0.7 | Balance relevance vs diversity |

---

## ⚙️ Setup Instructions

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_USERNAME/document-qa-bot.git
cd document-qa-bot
```

### 2. Open the notebook in Google Colab

Upload `Document_QA_RAG.ipynb` to [colab.research.google.com](https://colab.research.google.com) or open directly from GitHub.

### 3. Get a Google Gemini API key

1. Go to [aistudio.google.com](https://aistudio.google.com)
2. Click **Get API key** → **Create API key**
3. Copy the key — you will paste it when prompted in Cell 4

### 4. Add documents

Place your PDF, DOCX, and TXT files inside the `docs/` folder. The notebook already includes 5 sample documents covering Python, AI, Machine Learning, Databases, and Cloud Computing.

### 5. Run the notebook top to bottom

| Cells | Step |
|---|---|
| Cell 1 | Install all dependencies |
| Cell 2 | Upload / place documents in `docs/` |
| Cell 3 | Import libraries |
| Cell 4 | Configure Gemini API key |
| Cells 5–7 | Load PDF, DOCX, TXT documents |
| Cell 8 | Merge all documents |
| Cell 9 | Chunk documents |
| Cell 10 | Create embeddings |
| Cell 11 | Build and persist FAISS index |
| Cell 12 | Verify index |
| Cells 13–18 | Interactive Q&A loop |

> **Note:** Cells 1–12 (indexing) only need to run once. After the vectorstore is saved to disk, you can go straight to Cell 13 on subsequent sessions.

---

## 🔑 Environment Variables

Never commit API keys to your repository. The notebook reads the key via `getpass` (secure prompt) or from an environment variable:

```bash
# Option A — set before launching Colab (or in a .env file locally)
export GOOGLE_API_KEY="your-key-here"
```

```python
# Option B — the notebook will securely prompt you at runtime
GOOGLE_API_KEY = getpass.getpass("Enter your Google Gemini API key: ")
```

Add a `.env` file to `.gitignore`:

```
# .gitignore
.env
vectorstore/
__pycache__/
*.pyc
```

---

## 💬 Example Queries

| Question | Expected Source(s) |
|---|---|
| What is the difference between supervised and unsupervised learning? | `Machine_Learning.pdf` |
| How does Python support object-oriented programming? | `Python.pdf` |
| What are ACID properties in databases? | `Database.docx` |
| What is the difference between IaaS, PaaS, and SaaS? | `Cloud_Computing.txt` |
| How do transformer models work in deep learning? | `AI.pdf` |
| How does gradient boosting differ from random forest? | `Machine_Learning.pdf` |
| What vector databases are used in RAG systems? | `Database.docx`, `Cloud_Computing.txt` |
| What is the current stock price of Nvidia? | *(out-of-scope — bot declines)* |

---

## ⚠️ Known Limitations

| Limitation | Reason |
|---|---|
| No conversation memory | Each question is independent; the bot has no context of previous turns |
| English only | The embedding model (`all-MiniLM-L6-v2`) performs best on English text |
| Scanned PDFs not supported | `PyPDFLoader` extracts text layer only; image-based PDFs return empty pages |
| Answer quality depends on chunk boundaries | If an answer spans two chunks without overlap, it may be missed |
| Gemini rate limits | Free-tier Gemini API has requests-per-minute limits; slow down if you hit errors |
| FAISS is not distributed | Suitable for local/single-machine use; not production-scale |

---

## 📄 License

This project was built as part of an AI Engineering internship assignment. Feel free to fork and extend.

---

## 🙏 Acknowledgements

- [LangChain](https://github.com/langchain-ai/langchain) — RAG pipeline framework
- [FAISS](https://github.com/facebookresearch/faiss) — vector similarity search
- [Sentence Transformers](https://www.sbert.net/) — embedding model
- [Google Gemini](https://deepmind.google/technologies/gemini/) — answer generation LLM
