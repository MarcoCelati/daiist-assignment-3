# Assignment 3 Report

- **Name**:
- **Student ID**:
- **Group**: [BBADBA 5A | BBADBA 5B | PPLEDBA 5A]

> Every "TODO (yours)" below is a part the brief says must be your own work. Replace each one, and delete these notes before submitting.

## Use case

The people asking are students who are curious about the possibility of moving to the UK for study reasons. What they do with the answer is evaluate whether or not it's worth the effort of doing it as well as start gathering the information and documents needed or asked for. A confident wrong answer will give the person asking false information which they will act on and believe unquestionably, thereby wasting both time and money, as well as breaking the actual visa conditions upon arriving in the UK. In contrast, an "I don't know" answer would simply send them to the official government source or their university's international office and would only cost a couple minutes of time in checking.

## Corpus

Eight GOV.UK pages on the Student and Graduate routes, all under the **Open Government Licence v3.0**, with no personal data. They were fetched on 2026-11-24 through the GOV.UK Content API, and that snapshot is committed in `corpus/` so the test set and saved vectors stay valid.

| Source name | Document | Last updated | Words |
|---|---|---|---|
| `student-visa-guide` | [Student visa](https://www.gov.uk/student-visa) | 2024-10-31 | 5,166 |
| `appendix-student` | [Immigration Rules: Appendix Student](https://www.gov.uk/guidance/immigration-rules/immigration-rules-appendix-student) | 2026-10-08 | 8,241 |
| `student-caseworker-guidance` | [Student and Child Student (caseworker guidance)](https://www.gov.uk/government/publications/student-route-caseworker-guidance/student-and-child-student) | 2026-10-16 | 27,013 |
| `graduate-visa-guide` | [Graduate visa](https://www.gov.uk/graduate-visa) | 2024-10-31 | 2,813 |
| `appendix-graduate` | [Immigration Rules Appendix Graduate](https://www.gov.uk/guidance/immigration-rules/immigration-rules-appendix-graduate) | 2026-08-03 | 1,994 |
| `graduate-caseworker-guidance` | [Graduate caseworker guidance](https://www.gov.uk/government/publications/graduate-caseworker-guidance/graduate-caseworker-guidance-accessible-version) | 2026-08-03 | 6,430 |
| `appendix-finance` | [Immigration Rules Appendix Finance](https://www.gov.uk/guidance/immigration-rules/immigration-rules-appendix-finance) | 2025-04-09 | 1,409 |
| `appendix-english-language` | [Immigration Rules Appendix English Language](https://www.gov.uk/guidance/immigration-rules/immigration-rules-appendix-english-language) | 2026-03-26 | 1,643 |
| **Total** | | | **54,709** |

Word counts are after cleaning. The set covers the same rules at three levels: the plain-English guides, the legal rules (numbered paragraphs such as ST 1.2) and the caseworker guidance that explains how officials apply them.

## Ingestion

- **Extract:** each page comes from the GOV.UK Content API as JSON, not from scraping the web page. The HTML body is walked in order, and every paragraph, list item and table becomes a block that records its heading path (e.g. `Money you need > Course fee`) and, for the Immigration Rules, its rule number (e.g. `ST 4.1`).
- **Clean:** whitespace and HTML entities are normalised. Sections about the guidance document itself (Contacts, Publication, Changes from last version) are dropped.
- **Extraction problems fixed:**
  - Joining HTML tags left stray spaces, e.g. "Confirmation of Acceptance for Studies ( CAS )" in both text and headings; these are now removed.
  - 39 blocks were pure noise: "Official – sensitive: start/end of section" markers and stubs saying "the information in this section has been removed", now dropped.
  - GOV.UK writes one rule as "ST26.1", without the space. The rule pattern accepts that and stores it as ST 26.1, so it can be cited like every other rule.
- **Chunk, two designs** (a chunk never spans two documents):
  - *fixed*: 150-word windows with a 30-word overlap, ignoring structure → **458 chunks**.
  - *section*: one heading's text per chunk; sections over 300 words are split at block boundaries and sections under 40 words are merged into the next one → **402 chunks**.
- **Metadata on every chunk:** source name, title, URL, location (the rule number or heading where it starts), heading path, every heading and rule number it covers, word count, chunk number, design, and origin (corpus or added).
- **Embed and store:** `gemini-embedding-2` at 768 dimensions, with each chunk prefixed by its document title and heading path, in batches of 20, saved to ChromaDB (`index/chroma`, cosine distance) after every batch. Chunk ids are a hash of design, source and text, so a re-run embeds only new chunks; question vectors are cached in `index/query_vectors.json`.

## Test set

30 questions in `eval/questions.csv`: **8 single, 13 multi, 9 unanswerable**. In the two expected columns, `;` separates the passages a multi question needs, and `|` lists the other places where the same fact appears; `testset.py check` verifies every source and location against the documents.

> **TODO (yours):** how you wrote the questions, and two examples.

## Retrieval

A hit means that a retrieved chunk covers an expected location (any one place for a single question, every needed passage for a multi one). "Words added to the prompt" is the total length of the top-k chunks.

| design | questions | hit@1 | hit@3 | hit@5 | single hit@5 | multi hit@5 | words top1 | top3 | top5 |
|---|---|---|---|---|---|---|---|---|---|
| fixed | 21 | 19% | 38% | 48% | 62% | 38% | 150 | 447 | 747 |
| section | 21 | 14% | 48% | 67% | 75% | 62% | 161 | 472 | 811 |

I chose "section" because although there is a small tradeoff with it performing slightly worse for 1st (14% vs. 19%), it performs much better for top3 (48% vs. 38%) and top5 (67% vs. 48%), and as our agent reads it as long as it is in the top 5, then it doesn't matter whether it is 1st or 5th.

## Tools and subagent

`server.py` exposes two MCP tools; their docstrings are the descriptions the model reads.

- **search(query):** searches the GOV.UK guidance on the Student and Graduate visas and returns the 5 most relevant passages, best first, each starting with a citation label such as `[appendix-student, ST 1.2]`, followed by its section, GOV.UK link and text.
- **add_document(path):** adds one local file (.html, .json, .pdf, .md or .txt) to the corpus using `index.py`'s own pipeline (`chunk_file` and `embed_and_store`). It is only to be used when the user explicitly asks.

The subagent (`.claude/agents/corpus-researcher.md`) can use only the search tool. It must search before answering, answer only from the returned passages, cite every claim with its label, and otherwise reply with the refusal sentence.

> **TODO (yours):** one search result (from `uv run python main.py ask "..."` or your editor), and one transcript of the subagent in Claude Code or Copilot.

## Adding a document

Current behaviour (`index.add_document`):
- An unchanged file that is already indexed is skipped, so no quota is spent.
- A changed file with the same name replaces its old chunks.
- A file named like a corpus document is refused, so the core corpus can't be overwritten.

> **TODO (yours):** confirm or change this behaviour and give your reason. Then run the demo with `uv run python evaluate.py --demo <question id> <file>` (after your final evaluation), and paste the question, the document and the answers before and after from `eval/demo_add_document.md`.

## Harness and guardrails

`agent.py` runs a LangChain `create_agent` on Gemini (`gemini-3.6-flash`), with the tools from `server.py` over MCP and `AGENTS.md` as its system prompt (plus today's date, since several rules depend on dates).

- **Approval:** a call to `add_document` pauses the agent (HumanInTheLoopMiddleware). The user sees the path and answers y/N in the terminal; with nobody at the keyboard (evaluation, `check.py`) the call is rejected.
- **Call limits:** at most 6 tool calls and 8 model calls per question.
- **Time limits and retries:** `main.py ask` stops after 150 s and prints the reason, and a tool call that takes over 60 s is abandoned (the model is told it failed). Per-minute rate limits and server errors are retried; a used-up daily quota, or a model with no free quota, is not.
- **Model:** a fixed version, so the evaluation can be repeated. Google's alias `gemini-flash-latest` was not used: on 25 November 2026 it pointed to a newer model that timed out on Google's side (`uv run python agent.py --test-models` checks which models answer).
- **Fallback model:** if the main model fails, `main.py ask` tries `gemini-3.5-flash`, then `gemini-flash-lite-latest`, and its last line names the model that answered. Evaluation never falls back, so every graded answer comes from one model; it waits out rate limits instead.
- **Extra guardrail, citation check:** after the agent answers, every `[source, location]` citation must match a passage that search returned in this run (two citations written in one bracket are checked separately), and an answer that isn't a refusal must cite something. Otherwise the answer is replaced by the refusal sentence, and the blocked draft is kept in the trace.

> **TODO (yours):** your `AGENTS.md` rules, each with its reason, and one case where the citation check fired (listed in `eval/summary.md`; `main.py ask` also prints a note when it fires).

## Evaluation

`main.py evaluate` runs the agent on all 30 questions and checks in code:
- refusals, by the exact refusal sentence
- whether every citation is one that search returned
- whether the agent's own searches found the expected passage

Answers are cached, so a run stopped by the daily quota carries on where it left off; an answer whose searches all failed is not counted and runs again next time.

An LLM judge then grades each answer with the rubric in `eval/judge_rubric.md`, using the question, the reference answer and the retrieved text, and returns `correct`, `supported` and a reason. Agreement with my own grades (`eval/human_labels.csv`) is reported as a percentage and Cohen's kappa.

> **TODO (yours):** your rubric, your agreement with the judge, and the results (correct, unsupported, correct refusals, wrong refusals) from `eval/summary.md`.

## Failures

> **TODO (yours):** classify each failure in `eval/failures.csv` (retrieval miss, wrong answer from the right passage, unsupported claim, wrong refusal, bad test question), give the counts, and say what to fix first.

## Decision

> **TODO (yours):** would you let students rely on it, for what, and with which safeguard? Include the cost and latency per question from `eval/summary.md`.

## AI use

> **TODO (yours):** check this draft against what you actually did, and edit it.

I used Claude (Anthropic) as follows:

- **Code:** it wrote the code: `index.py`, `server.py`, `agent.py`, `evaluate.py`, the test-set helper `testset.py`, and the subagent file.
- **Options:** it suggested options for the topic, the documents and the chunking comparison; I chose UK student visas and fixed-size versus section-based chunking.
- **Test set:** I wrote all 30 questions. Claude looked up and verified where each answer appears in the documents (the expected sources and locations).
- **Report:** it drafted this report's factual descriptions: the corpus table, ingestion, tools and guardrails.

The following are my own:
- the use case
- the reference answers
- the `AGENTS.md` rules
- the judge rubric
- my grades
- the analysis and the decision
