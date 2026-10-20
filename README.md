# Macedonian Legal Assistant

This is a research project on how a legal-assistant pipeline can work with Macedonian laws. 
It focuses on finding relevant articles, answering in Macedonian Cyrillic, and checking whether the answer is supported by the sources.

The project uses **Retrieval-Augmented Generation (RAG)**: it searches the documents first, then gives the relevant text to a language model. It runs in Google Colab and currently uses four laws: the Criminal Code, Criminal Procedure Law, Law on Justice for Children, and Family Law.

The current setup is for **research and demonstration, not production use yet**. It makes each stage easy to inspect and change within the available hardware. Its answers still need to be checked against the original legal text.

## Architecture: what is used and why

The documents are prepared first. Each question then goes through search, ranking, generation, and checks.

```text
Preparation: PDFs → article records → meaning-based and word-based search indexes
Answering:   question → hybrid search → reranking → Gemma → checks → answer + sources
```

| Part | What it does | Why it fits this research project |
| --- | --- | --- |
| **Colab and Google Drive** | Run the notebook and store the input PDFs. | Allow experiments without setting up a server. Cells make intermediate results visible. |
| **PyMuPDF and article parsing** | Extract text, split it into legal articles, and keep page references. | Articles provide understandable units for retrieval and citations. |
| **[BGE-M3](https://huggingface.co/BAAI/bge-m3) + FAISS** | Turn text into numerical representations and search for similar meanings. FAISS stores and searches those vectors. | BGE-M3 is multilingual; FAISS gives a simple local index for the small corpus without a database server. |
| **BM25 + Reciprocal Rank Fusion** | Match words and combine those results with the meaning-based search. | Legal wording matters. Combining the two methods lets the project test both exact terms and differently worded questions. |
| **[BGE-reranker-v2-m3](https://huggingface.co/BAAI/bge-reranker-v2-m3)** | Read each candidate article together with the question and reorder the results. | Provides a closer relevance check before sending evidence to the language model. |
| **[Gemma 3 4B IT](https://huggingface.co/google/gemma-3-4b-it)** | Generate up to two short claims in Macedonian Cyrillic, with source IDs. | A pretrained 4B model provides a manageable starting point for local experiments without training a model from scratch. |
| **Code checks + a second Gemma call** | Check source IDs, references, numbers, and whether claims are supported and relevant. | Make errors easier to inspect. Reusing Gemma avoids loading another language model, but its verification is not independent. |

The answer is assembled from the retained claims, with citations taken from the article metadata. The system can return a partial answer or report that its checks rejected the claims.

Requests for a specific article return its stored text directly. Hypothetical questions also use simple age extraction and query variants, but applying the law to the facts is still a weak point. The models are pretrained; no fine-tuning was done.

These choices provide a research baseline. The tests do not yet establish that this is the best combination of models or that every stage improves accuracy.

## Test results

The main test set contains 10 factual and 10 hypothetical questions. The saved run completed 16 calls before running out of GPU memory.

| Result | Factual | Hypothetical |
| --- | ---: | ---: |
| Completed calls | 10 | 6 |
| Status `ok` | 6 | 3 |
| Partial answer | 3 | 2 |
| Output-format error | 1 | 1 |
| Interrupted by memory error | 0 | 1 |
| Not reached | 0 | 3 |

`ok` means the answer passed the current checks. Some accepted answers still contained errors, so these counts do not measure legal accuracy.

Median response time was about **105 seconds for factual questions** and **139 seconds for completed hypothetical questions**. Some calls took more than five minutes.

The [full test log](results/test_run.txt) contain the recorded results. They were taken from the saved notebook run.

## What works and what needs improvement

Direct article lookup works in the tested example. Search finds useful provisions for several factual questions, and the system produces Cyrillic answers from Cyrillic, Latin-transliterated, and English inputs. Some incorrect citations are caught by the checks.

The main problems are:

- **Hypothetical reasoning:** answers sometimes change a stated fact or apply a rule without checking its conditions.
- **Verification:** the same model writes and checks the answer, and can accept its own mistakes.
- **Validation:** one missing source-ID field can cause the whole answer to be rejected. A user-provided age can also be incorrectly rejected as an unsupported number.
- **Completeness and language:** some answers omit relevant rules, repeat phrases, or use awkward Macedonian.
- **Source quality:** parsing reported 18 duplicate article-ID warnings. Law versions and amendments are not automatically checked.

## Hardware limits

Testing used **Google Colab with an NVIDIA T4**, with 14.56 GiB of GPU memory reported by the runtime. Model loading and long evidence passages leave limited space for generation. The reranker runs on CPU, which can also slow the pipeline.

The seventh hypothetical test failed with a CUDA out-of-memory error. More GPU memory could help with execution, but reasoning and verification errors also need changes to the pipeline.

## Next steps

- Fix duplicate article IDs and track document versions.
- Improve fact handling, claim validation, and checks of legal conditions.
- Retrieve evidence for each part of a hypothetical question.
- Limit context by token count, cache indexes, and measure memory and time per stage.
- Build a larger test set with expert-reviewed answers.
- Add a Ministry of Justice crawler and gradually expand the legal collection. The crawler is not implemented in this notebook.


## Run the notebook

Open [Macedonian_legal_LLM.ipynb](Macedonian_legal_LLM.ipynb) in Google Colab, select a GPU runtime, and mount Google Drive. Put the PDFs in the folder set by `ROOT`, with filenames matching `LAWS`. Obtain access to Gemma on Hugging Face and log in when prompted.

Run the cells in order and try individual questions before the full test suite. PDFs and model weights are not included. The published notebook keeps the original code; saved outputs are stored separately in `results/`.
