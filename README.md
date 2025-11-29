# 📚 Primary Science RAG + Quiz Generator API

A FastAPI-based backend that:

1. **Processes a folder of PDFs** (digital or scanned),
2. **Extracts text** (native PDF text or OCR),
3. **Chunks and embeds content** using SentenceTransformers,
4. **Stores embeddings in Pinecone** for retrieval,
5. Provides a **RAG endpoint** to generate *primary-level Singapore Science notes* using GPT,
6. Provides a **Quiz generation endpoint** that retrieves MCQs from Weaviate and returns 10 questions via GPT.

---

## ✨ Features

### PDF Ingestion + Indexing

* Supports **digital PDFs** via PyMuPDFLoader.
* Supports **scanned PDFs** via OCR (Tesseract + pdf2image).
* Splits content into overlapping chunks.
* Embeds chunks using `all-MiniLM-L6-v2`.
* Stores vectors in a Pinecone index under a namespace.

### RAG Notes Generation

* Retrieves relevant chunks from Pinecone.
* Injects retrieved context into a teacher-style prompt.
* Uses GPT (via OpenAI API) to generate structured notes for primary school students.

### Quiz Generation (Weaviate Hybrid Search)

* Hybrid-retrieves matching MCQs from Weaviate cloud collection.
* Formats them into context.
* GPT produces **10 MCQs only** (no intro/outro).

---

## 🧠 High-Level Architecture

**Part A: PDF ➜ Pinecone**

```
PDF folder
   ├─ digital pdf → PyMuPDFLoader
   └─ scanned pdf → pdf2image → pytesseract OCR
            ↓
   LangChain Documents
            ↓
 Split into chunks (RecursiveCharacterTextSplitter)
            ↓
 Embed chunks (SentenceTransformer all-MiniLM-L6-v2)
            ↓
 Upsert to Pinecone (new-hybrid-index)
```

**Part B: Notes RAG**

```
User query
   ↓
Pinecone similarity search
   ↓
context + question → prompt
   ↓
OpenAI GPT response
   ↓
Return notes
```

**Part C: Quiz RAG**

```
(level, difficulty, subject)
   ↓
Construct query sentence
   ↓
Weaviate hybrid search (ScienceQuestion collection)
   ↓
Extract questions into context
   ↓
GPT returns ONLY MCQs
```

---

## 📦 Tech Stack

* **FastAPI** (web server)
* **LangChain** (RAG + chains)
* **PyMuPDFLoader** (digital PDF text extraction)
* **pytesseract + pdf2image** (OCR for scanned PDFs)
* **SentenceTransformers** (`all-MiniLM-L6-v2`)
* **Pinecone** (vector DB for notes)
* **Weaviate Cloud** (MCQ store + hybrid search)
* **OpenAI GPT Models** (`gpt-4o`)

---

## ✅ Requirements

### Python

* Python **3.9+** recommended.

### System Dependencies (OCR)

You must install these at OS level:

**Ubuntu/Debian**

```bash
sudo apt-get update
sudo apt-get install -y tesseract-ocr poppler-utils
```

**Mac (Homebrew)**

```bash
brew install tesseract poppler
```

---

## 📜 Installation

1. Clone repo and enter folder:

```bash
git clone <your-repo-url>
cd <your-repo>
```

2. Create and activate venv:

```bash
python -m venv venv
source venv/bin/activate   # windows: venv\Scripts\activate
```

3. Install Python packages:

```bash
pip install -r requirements.txt
```

---

## 🔐 Environment Variables

Set these before running:

```bash
export OPENAI_API_KEY="your_openai_key"
export PINECONE_API_KEY="your_pinecone_key"
export PORT=4000
```

Weaviate is currently **hardcoded** inside the script:

* `cluster_url`
* `api_key`
  If you plan to deploy publicly, move these into env vars.

---

## 📂 PDF Indexing Workflow

Your script defines ingestion utilities, but does **not call them automatically** on startup.
To index PDFs, run these steps manually in a Python shell or separate script:

```python
from main import document_processing, Split_chunks, embedding_output, VectorStoring

docs = document_processing("path/to/pdfs")
splits = Split_chunks(docs)
embeddings = embedding_output(splits)
VectorStoring(splits, embeddings)
```

### Notes

* Pinecone index name used: **`new-hybrid-index`**
* Namespace during upsert: **`p3-p6-namespace`**
* Namespace for retrieval in notes endpoint: **`final-note-namespace`**

⚠️ **Important:** Your indexing and retrieval namespaces differ.
If you want retrieval to find indexed content, use **the same namespace** in both places.

---

## 🚀 Running the API

```bash
python main.py
```

or directly with uvicorn:

```bash
uvicorn main:app --host 0.0.0.0 --port 4000
```

---

## 🔌 API Endpoints

### 1) Generate Notes (RAG from Pinecone)

**GET**

```
/output/{input_msg}
```

**Example**

```bash
curl "http://localhost:4000/output/What is photosynthesis?"
```

**Response**

```json
{
  "message": "Teacher-style primary science notes..."
}
```

---

### 2) Generate MCQs (RAG from Weaviate)

**POST**

```
/generate
```

**Request Body**

```json
{
  "level": "Primary 5",
  "difficulty": "medium",
  "subject": "Electricity"
}
```

**Example**

```bash
curl -X POST "http://localhost:4000/generate" \
-H "Content-Type: application/json" \
-d '{"level":"Primary 5", "difficulty":"easy", "subject":"Forces"}'
```

**Response**

```json
{
  "result": "1) ...\n2) ...\n..."
}
```

---

## 🧪 Example Prompts Used

### Notes Prompt

Teacher assistant prompt that:

* speaks for Primary Singapore Science level,
* uses only retrieved context,
* builds basics first,
* avoids missing details.

### Quiz Prompt

Teacher assistant prompt that:

* extracts questions from Weaviate context only,
* returns MCQs only,
* no intro/conclusion.

---

## 🛠 Troubleshooting

### OCR returns blank text

* Ensure `tesseract` installed correctly.
* Check poppler install (`pdftoppm` should exist).
* Some PDFs may be too low-resolution.

### Pinecone “index not found”

* Create the index in Pinecone dashboard first:

  * name: `new-hybrid-index`
  * dimension: `384` (MiniLM output size)
  * metric: cosine

### Weaviate auth issues

* Verify API key and cluster URL.
* Confirm collection name `ScienceQuestion` exists.

### Namespace mismatch (no retrieval results)

Align:

* indexing → `VectorStoring(... namespace="p3-p6-namespace")`
* retrieval → `PineconeVectorStore(... namespace="final-note-namespace")`

---

## 🔒 Security Notes

* Don’t hardcode Weaviate API keys in production.
* Use `.env` or secret manager (Render, AWS, GCP).
* Add rate limiting if public.

---

## 📌 Suggested Improvements

Not required, but strongly recommended:

1. **Auto-ingestion CLI**

   * Add a script like `ingest.py` for smooth PDF indexing.

2. **Consistent IDs**

   * You currently use page numbers as IDs → duplicates across PDFs.
   * Better: `f"{filename}-p{page}"`

3. **Batch retrieval settings**

   * `similarity_search()` defaults might return too many / too few chunks.
   * Set `k=4` or `k=6`.

4. **Separate digital detection**

   * `check_for_digital_pdf` returns after first page only.
   * Should scan more pages for robustness.

5. **Async OpenAI**

   * For scale, use async OpenAI calls in FastAPI.

---

## 📄 License
MIT
