# -Vehicle-Specification-Extraction-using-LLM-and-RAG-Pipeline

# Vehicle Specification Extraction

## 1. Project Overview

Vehicle Specification Extraction is a retrieval-augmented question-answering system designed to extract technical specifications—such as torque values, fluid capacities, and part numbers—from vehicle service manuals in PDF format.

The system retrieves relevant passages from the manual and uses a large language model (LLM) to generate answers grounded in the retrieved evidence. It also provides source-page information to help users verify the results and is designed to abstain when the requested specification cannot be found.

## 2. System Design

The system follows a Retrieval-Augmented Generation (RAG) architecture.

```text
Service Manual PDF
        |
        v
PDF Text Extraction
(src/extract_pdf.py)
        |
        v
Extracted Pages (JSONL)
(outputs/pages.jsonl)
        |
        v
Chunking and Embedding
(src/build_index.py)
        |
        v
Searchable Retrieval Index
(outputs/index/)
        |
        v
User Question
        |
        v
Hybrid Retrieval
(Semantic + Keyword Matching)
        |
        v
Relevant Manual Passages
        |
        v
LLM-Based Answer Generation
        |
        v
Answer + Source References
```

### 2.1 PDF Text Extraction

The `src/extract_pdf.py` script extracts text from the service-manual PDF using PyMuPDF. The extracted content is stored in `outputs/pages.jsonl`, with page-level information preserved for source attribution.

Keeping page boundaries allows the system to identify the PDF page containing the relevant specification.

### 2.2 Index Construction

The `src/build_index.py` script processes the extracted text, divides it into smaller chunks, generates text embeddings, and saves the retrieval index in `outputs/index/`.

Chunking makes it easier to retrieve relevant passages without processing the entire manual for every question. Embeddings support semantic matching between user questions and manual passages.

### 2.3 Information Retrieval

The `src/ask.py` script retrieves passages relevant to the user's question. The current implementation combines semantic similarity with keyword-based scoring.

Semantic retrieval identifies passages with related meanings, while keyword matching helps with exact technical terms, component names, and section identifiers.

This hybrid approach is particularly useful when distinguishing between similar components, such as front and rear brake calipers.

### 2.4 Answer Generation

The retrieved passages are supplied to the configured OpenAI model to generate a structured answer.

The answer should remain grounded in the retrieved manual evidence and include relevant source details. If the requested specification is not found, the system can return an empty result rather than inventing a value.

A regex-based fallback provides limited functionality when model-based generation is unavailable.

## 3. Tools and Technologies

| Tool | Purpose |
|---|---|
| Python 3.11 | Implements the extraction, indexing, and question-answering pipeline. |
| PyMuPDF | Extracts text from PDF documents. |
| Sentence Transformers | Generates embeddings for semantic retrieval. |
| JSON Lines (JSONL) | Stores extracted page text and page-level information. |
| Hybrid retrieval | Combines semantic similarity and keyword-based matching. |
| OpenAI API | Generates answers from retrieved evidence. |
| Virtual environment (`venv`) | Isolates project dependencies. |
| pip | Installs the required Python packages. |

The exact embedding model and indexing implementation are defined in the project source files and `requirements.txt`.

## 4. Installation and Execution

### Step 1: Create a virtual environment

```bash
python3.11 -m venv .venv
source .venv/bin/activate
```

### Step 2: Install dependencies

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### Step 3: Extract PDF text

Place the service manual at `data/manual.pdf` and run:

```bash
python src/extract_pdf.py \
  --pdf data/manual.pdf \
  --out outputs/pages.jsonl
```

### Step 4: Build the retrieval index

```bash
python src/build_index.py \
  --input outputs/pages.jsonl \
  --out-dir outputs/index
```

### Step 5: Configure the language model

Set the OpenAI API key and model name as environment variables:

```bash
export OPENAI_API_KEY="YOUR_API_KEY"
export OPENAI_MODEL="gpt-5.4-mini"
```

Use a valid API key and never commit it to version control.

### Step 6: Ask a question

```bash
python src/ask.py \
  --index-dir outputs/index \
  --question "What is the torque for the front brake caliper guide pin bolts?"
```

## 5. Example Questions

The system can be tested using questions such as:

- What is the torque for the front brake caliper guide pin bolts?
- What is the torque for the rear brake caliper guide pin bolts?
- What is the torque for the brake caliper anchor plate bolts?
- What is the brake fluid fill capacity listed in Section 206-03, Front Disc Brake?
- What is the part number for a component that is not mentioned in the manual?

The last question tests whether the system can abstain when evidence is unavailable.

Questions should specify the component, location, or manual section when multiple interpretations are possible.

## 6. Testing and Evaluation

The system should be evaluated using a manually verified set of questions covering:

1. Torque specifications.
2. Fluid capacities.
3. Part numbers.
4. Missing specifications.
5. Ambiguous or section-dependent questions.

Useful evaluation metrics include:

- **Answer accuracy:** Whether the extracted specification matches the verified answer.
- **Retrieval recall:** Whether the correct supporting passage appears among the retrieved results.
- **Source accuracy:** Whether the cited PDF page and section support the answer.
- **Abstention quality:** Whether the system avoids answering when sufficient evidence is unavailable.
- **Unsupported-answer rate:** How often the system produces values not supported by the manual.

A test set of at least 20–30 questions is a practical starting point. Each question should have a verified expected answer and source page.

## 7. Limitations

The current implementation has several limitations:

- Scanned pages may require Optical Character Recognition (OCR).
- Complex tables can lose their row and column relationships during PDF text extraction.
- Similar component names may lead to incorrect retrieval.
- Broad questions may return specifications from multiple manual sections.
- Part numbers in tables or illustrations may be difficult to retrieve.
- LLM-generated answers may still be incorrect, even when relevant passages are retrieved.
- The regex fallback has limited contextual understanding.
- Unit conversions and formatting differences require careful validation.

For safety-critical repair work, specifications should always be checked against the original service manual.

## 8. Ideas for Improvement

### 8.1 Improve Document Processing

- Add OCR support for scanned manuals.
- Improve table extraction to preserve relationships between components, values, and units.
- Preserve section titles, subsection identifiers, PDF page numbers, and printed manual page numbers separately.
- Use section-aware chunking to reduce confusion between similar specifications.

### 8.2 Improve Retrieval

- Tune chunk size and overlap using a verified evaluation dataset.
- Add metadata filters for component type, vehicle location, and manual section.
- Compare the current hybrid retrieval method with BM25 keyword retrieval.
- Add a reranking model to improve the ordering of retrieved passages.

### 8.3 Improve Answer Reliability

- Require a supporting evidence excerpt for every extracted specification.
- Validate model output against a predefined structured schema.
- Keep specifications from different sections separate rather than combining them into a single value.
- Preserve the original units printed in the manual.
- Implement explicit abstention rules for missing, conflicting, or insufficient evidence.
- Use pattern-based validation for part numbers.

### 8.4 Improve Evaluation

- Build a versioned dataset of questions, verified answers, and source pages.
- Evaluate retrieval quality separately from answer-generation quality.
- Add regression tests whenever the retrieval algorithm, prompt, or embedding model changes.
- Measure latency and API usage alongside accuracy.

### 8.5 Improve Efficiency and Usability

- Cache embeddings and the retrieval index to avoid unnecessary recomputation.
- Pin dependency versions for reproducibility.
- Add configuration options for the manual path, model, and retrieval settings.
- Develop a simple web interface after the system's accuracy has been evaluated.

## 9. Security Considerations

- Store API keys in environment variables or a local `.env` file excluded from version control.
- Never commit credentials or sensitive documents without authorization.
- Revoke and replace any API key that becomes exposed.
- Treat retrieved document content as evidence, not as executable instructions.
- Preserve source references so users can verify important specifications.

## 10. Conclusion

The Vehicle Specification Extraction system combines PDF text extraction, embedding-based retrieval, keyword matching, and LLM-based answer generation to retrieve technical specifications from vehicle service manuals.

Its main strengths are evidence-based retrieval, source-page traceability, and the ability to abstain when information is missing. Future improvements should focus on table extraction, section-aware retrieval, structured evidence validation, and systematic evaluation of answer accuracy.
