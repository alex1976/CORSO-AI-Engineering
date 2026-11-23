# Module 4: The Heart of the Project - Building a RAG System

## Learning objectives

By the end of the module you will be able to:

- explain why an LLM alone cannot answer questions about private documents and what alternatives exist;
- design the ingestion pipeline: loading, cleaning, chunking, embedding, indexing;
- choose a chunking strategy justified by the type of document;
- implement the retrieval loop: question, vector, search, context building, generation;
- recognize the points where a RAG system fails and know where to intervene.

## 1. Why RAG

An LLM knows what it saw during training, up to a certain date, and it does not know your company's internal documents. There are three ways to close this gap:

1. **Fine-tuning**: retraining the model on the documents. Expensive, slow to update, hard to verify, and it does not eliminate hallucinations.
2. **Long context**: pasting the whole document into the prompt. It works for small documents, but it costs a lot, slows things down and often makes accuracy worse when the context is full of irrelevant information.
3. **RAG (Retrieval-Augmented Generation)**: searching the documents for only the fragments relevant to the question and passing them to the model as context.

RAG is the most common choice when data changes often, when you need to cite sources and when the volume is too large to fit in the context. However, the quality of its answers is limited by the quality of retrieval: if the right fragment is not found, the model will answer poorly even if it is excellent.

## 2. The two-phase pipeline

A RAG system has two paths.

**Offline (ingestion)**, run whenever the documents change:
loading → cleaning → chunking → embedding → indexing in the vector database.

**Online (querying)**, run on every question:
question → question embedding → search for the most similar chunks → prompt construction → generation → answer with sources.

Many problems originate in the first path and show up in the second. When an answer is wrong, the first question to ask is: did the right chunk exist in the index?

## 3. Ingestion

### Loading and cleaning

Real documents are messy: PDFs with repeated headers and footers, broken tables, scans, multi-column text, page numbers, tables of contents. Before chunking you must decide what to keep.

Exercise: take a real PDF (a manual, a contract, a company policy) and extract the text with two different tools. Compare the results on tables and headers. Note what should be removed and what should be kept as metadata.

### Metadata

Every chunk should carry information needed for filtering and citation:

- source document and version;
- page or section;
- publication or update date;
- document type or department;
- chunk identifier.

Without metadata you cannot filter (for example: "only the policies currently in force") or show the user where the answer comes from.

### Chunking

Chunking splits the text into blocks. The choice of size and strategy has more influence than almost any other parameter.

Common strategies:

- **Fixed size with overlap**: blocks of fixed length (for example 1000 characters) with an overlap (for example 200 characters) so that a concept is not split between two blocks without a point of contact. This is the starting strategy recommended in the course.
- **Recursive by separators**: tries to split on paragraphs, then sentences, then words, keeping units of meaning whenever possible.
- **By structure**: uses headings, sections and articles as natural boundaries. Ideal for legal documents or numbered manuals.
- **Semantic**: splits where the topic changes, measured with embeddings. More expensive, not always better.

Trade-offs:

- small chunks: more precise retrieval, but the individual fragment may not contain the necessary context;
- large chunks: more context per fragment, but more noise and more tokens in the prompt;
- high overlap: less information lost at the boundary, but more duplication and higher indexing cost.

The 1000/200 value is a starting point, not a rule. It must be validated on your corpus.

### Chunking exercise

Take a document of at least ten pages with numbered sections. Create three versions of the index: fixed size 500, 1000 and 2000 characters, with overlap 200. For ten questions whose answer you already know, check whether the correct chunk appears among the top three results. Note which configuration works best and why.

## 4. Embeddings and vector databases

### Embeddings

An embedding is a vector of numbers (from a few hundred to a few thousand dimensions) that represents the meaning of a text. Texts with similar meaning produce nearby vectors, usually measured with cosine similarity.

Two sentences such as "How do I request a refund?" and "Procedure for getting my money back" have few words in common but nearby vectors: this is what makes semantic search more useful than keyword search.

Note: the embedding model is a separate choice from the generation model. It must be chosen for the language of the documents (Italian, multilingual) and must be used identically in ingestion and in querying. Changing the embedding model requires reindexing everything.

### Vector databases

A vector database stores vectors together with their metadata and lets you search for the vectors closest to a given vector. Tools mentioned in the course:

- **ChromaDB**: simple, suited to prototypes and local environments;
- **Qdrant**: performance and metadata filters, suited to production;
- **Pinecone**: managed service, with no infrastructure to administer.

For the course a local database is enough. The choice matters less than attention to metadata, filters and reindexing.

### Similarity search and hybrid search

Pure vector search does not always find exact codes, acronyms or proper names (for example "error E18" or "article 17-bis"). For this reason real systems often combine vector search with keyword search (BM25 or similar), merging the results. It is one of the first improvements to consider.

## 5. The retrieval loop

The online path, step by step:

1. The user asks a question.
2. The question is turned into a vector with the same embedding model.
3. The vector database is searched for the k most similar fragments (typically 3-5, possibly filtered by metadata).
4. Optional: a reranker reorders the fragments with a more accurate model, discarding those that are not very relevant.
5. The prompt is built: instruction, numbered fragments with their source, question.
6. The model generates the answer, citing the fragments used.
7. If no fragment exceeds a relevance threshold, the system says it does not have the information instead of making it up.

Example generation prompt:

```text
Answer the question using only the fragments provided.
If the answer is not present in the fragments, write: "I did not find this information in the available documents."
Cite the source of every statement with [fragment_number].

Fragments:
[1] (manuale_lavatrice.pdf, p. 12) ...
[2] (manuale_lavatrice.pdf, p. 13) ...

Question: {{domanda}}
```

### The value of the threshold

Without a threshold, retrieval always returns k fragments, even when none of them is relevant. The model, at that point, tries to answer anyway. A similarity threshold, or a relevance check, is often the difference between a reliable system and one that hallucinates with confidence.

## 6. Advanced retrieval techniques

Basic RAG (question embedding, top-k, generation) is a good starting point. When the evaluation in module 6 shows insufficient recall, more sophisticated techniques are introduced. Each one adds cost or latency: they should be introduced one at a time and measured.

### Hybrid search

It combines vector search (meaning) with keyword search, for example BM25 (exact terms). The two result lists are merged, often with Reciprocal Rank Fusion: each document receives a score based on its position in each list.

When it is needed: error codes, product names, articles of law, acronyms. "E18" has little semantic meaning, but it is a valuable exact match.

### Query rewriting

The user's question is often incomplete or colloquial. Before retrieval, a model rephrases it.

- **Rewriting with history**: "What if the filter is already clean?" becomes "Persistent error E18 after cleaning the drain filter".
- **Multi-query**: several variants are generated from the question and the results are merged.
- **HyDE (Hypothetical Document Embeddings)**: the model writes a hypothetical answer and the search uses its embedding, which often resembles the documents more than the question does.

### Reranking

Vector retrieval is fast but approximate. A reranker (usually a cross-encoder model) evaluates each question-fragment pair more carefully and reorders the results. Typical scheme: retrieve 20-50 candidates, rerank, pass the best 3-5 to the model.

### Hierarchical chunking (small-to-big)

Small fragments, which are precise in retrieval, are indexed, but the model is given the larger block that contains them (the whole paragraph or section). It combines the precision of small chunks with the context of large chunks.

### Contextual enrichment of chunks

An isolated fragment such as "Clean the filter every three months" does not say which product or section it refers to. Before embedding, a short description of the context (document, chapter, topic) is prepended to each chunk, generated only once during ingestion. It improves retrieval when documents have many fragments that are similar to each other.

### Metadata filters

Restrict the search before computing similarity: only the specified washing machine model, only the policies in force, only the documents of the user's department. It is also a security mechanism: a user must not retrieve documents they do not have access to.

### Which technique for which symptom

| Measured symptom | Technique to try first |
|---|---|
| Codes and acronyms not found | Hybrid search |
| Colloquial or follow-up questions fail | Query rewriting |
| The right fragment is retrieved but ranked low | Reranking |
| The right fragment is found but context is missing | Hierarchical chunking |
| Many nearly identical fragments across different documents | Contextual enrichment, metadata filters |

### Exercise

On your prototype, measure recall@3 with the module 6 dataset. Add hybrid search and measure again; then add reranking and measure once more. Report the three values and the average time per request in each configuration.

## 7. Where a RAG system fails

When the answer is wrong, classify the problem before fixing it:

| Symptom | Probable cause | Where to intervene |
|---|---|---|
| The right fragment is not among the results | Chunking, embedding, query too different from the text | Chunking, hybrid search, query rewriting |
| The right fragment is found but ranked low | Weak ranking | Reranker, number of results |
| The right fragment is in the context but the answer is wrong | The model ignores the context or misinterprets it | Prompt, instructions, fragment format |
| Correct answer but no source | Prompt without a citation requirement | Generation prompt |
| Question spanning several documents, partial answer | Retrieval with k too low or without aggregation | Multiple queries, higher k, synthesis |
| Outdated information | Index not updated, version metadata missing | Update pipeline, date filters |

Module 6 shows how to measure these causes separately.

## 8. Use cases to develop

Choose one of the following and build it as a small prototype with a Streamlit interface.

- **HR policy assistant**: vacation, expense reimbursements, remote work. Documents with different versions; the answer must report the version in force on the date of the question.
- **Chatbot on the documentation of an open source API**: questions about endpoints and parameters. Check its ability to handle acronyms and function names (hybrid search).
- **Meeting minutes assistant**: questions such as "What decisions did we make about X?" over an archive of minutes with date and participants as metadata.
- **Advisor for a municipal regulation or a contract**: answers that cite article and paragraph. Here the structure of the document must be used as the chunking criterion.

For each one, deliver: metadata schema, a justified chunking strategy, ten test questions with the expected answer and the expected source, and a failure analysis using the table in section 7.

## Further reading

- [Lewis et al., "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks" (2020)](https://arxiv.org/abs/2005.11401): the paper that defined RAG.
- [Gao et al., "Retrieval-Augmented Generation for Large Language Models: A Survey" (2023)](https://arxiv.org/abs/2312.10997): survey of RAG techniques, from the basic approach to advanced variants.
- [Anthropic, "Introducing Contextual Retrieval" (2024)](https://www.anthropic.com/news/contextual-retrieval): contextual enrichment of chunks, as discussed in the module.
- [ChromaDB documentation](https://docs.trychroma.com): sections on collections, metadata and filters.
- [Qdrant documentation](https://qdrant.tech/documentation/): sections on hybrid search and payload filters.
