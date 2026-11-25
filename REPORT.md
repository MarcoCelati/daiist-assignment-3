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

I'm keeping all three rules. Adding the same file again doesn't re-embed anything and just tells the user nothing changed, so it's harmless. The gap is when the same page is saved under a different file name. Then it gets stored twice, and the copies can push other passages out of the top 5. Missed passages were the most common failure in my results, so I'd fix this by checking for duplicate text instead of only the file name. Replacing a changed file matters because GOV.UK pages get updated (the living-cost amounts, for example), and a student shouldn't see the old amount next to the new one. Refusing files named like a corpus document stops someone overwriting one of the eight official documents by accident, which would change what the answers and citations are based on.

**Demo, q12** (`uv run python evaluate.py --demo q12 demo_page.html`, saved in `eval/demo_add_document.md`):

**Question (unanswerable):** My employer isn't a licensed sponsor. Can they get a licence before my Graduate visa runs out, and how long does that take?

**Document added:** `demo_page.html`

**add_document said:** Added 'demo-page': 60 chunks, 60 newly embedded, in design 'section'.

**Before**

> I can't find this in the documents I have.
>
> Please check GOV.UK for official guidance on sponsor licence application processing times and requirements, or speak with an immigration adviser or your employer's HR team.

Citations: none

**After**

> Yes, your employer can apply for a sponsor licence before your Graduate visa runs out.
>
> Most sponsor licence applications are dealt with in less than 8 weeks, though UK Visas and Immigration (UKVI) may need to visit the business [demo-page, How long it takes to get a decision]. Additionally, your employer may be able to pay an extra £750 to get a decision within 10 working days, though this faster service is limited to a small number of applications each working day on a first-come, first-served basis [demo-page, How long it takes to get a decision].

Citations: [demo-page, How long it takes to get a decision]

Before adding the page, the agent refused q12 with the refusal sentence. After adding GOV.UK's "UK visa sponsorship for employers" guide (saved as demo_page.html and split into 60 chunks), it answered with the under-8-weeks decision time, the possible UKVI visit and the £750 option for a decision within 10 working days, each cited to the new document. Its opening sentence, though, says the employer can get a licence before the Graduate visa runs out, which the page can't support, because that depends on how much time is left on the visa. Adding a document closed the coverage gap, but it didn't stop the agent overstating.

## Harness and guardrails

`agent.py` runs a LangChain `create_agent` on Gemini (`gemini-3.5-flash-lite`), with the tools from `server.py` over MCP and `AGENTS.md` as its system prompt (plus today's date, since several rules depend on dates).

- **Approval:** a call to `add_document` pauses the agent (HumanInTheLoopMiddleware). The user sees the path and answers y/N in the terminal; with nobody at the keyboard (evaluation, `check.py`) the call is rejected.
- **Call limits:** at most 6 tool calls and 8 model calls per question.
- **Time limits and retries:** `main.py ask` stops after 150 s and prints the reason, and a tool call that takes over 60 s is abandoned (the model is told it failed). Per-minute rate limits and server errors are retried; a used-up daily quota, or a model with no free quota, is not.
- **Model:** a fixed version, so the evaluation can be repeated. Flash-Lite because on the free tier it allows 500 requests a day, while the full Flash models allow 20 and one evaluation run needs about 150. Google's alias `gemini-flash-latest` was not used: on 25 November 2026 it pointed to a newer model that timed out on Google's side (`uv run python agent.py --test-models` checks which models answer).
- **Fallback model:** if the main model fails, `main.py ask` tries `gemini-3.1-flash-lite`, then `gemini-3.6-flash`, and its last line names the model that answered. Evaluation never falls back, so every graded answer comes from one model; it waits out rate limits instead.
- **Extra guardrail, citation check:** after the agent answers, every `[source, location]` citation must match a passage that search returned in this run (two citations written in one bracket are checked separately), and an answer that isn't a refusal must cite something. Otherwise the answer is replaced by the refusal sentence, and the blocked draft is kept in the trace.

> **TODO (yours):** your `AGENTS.md` rules, each with its reason, and one case where the citation check fired (listed in `eval/summary.md`; `main.py ask` also prints a note when it fires).

## Evaluation

`main.py evaluate` runs the agent on all 30 questions and checks in code:
- refusals, by the exact refusal sentence
- whether every citation is one that search returned
- whether the agent's own searches found the expected passage

Answers are cached, so a run stopped by the daily quota carries on where it left off; an answer whose searches all failed is not counted and runs again next time.

An LLM judge (`gemini-3.1-flash-lite`, a different model from the agent's, so it isn't grading its own answers) then grades each answer with the rubric in `eval/judge_rubric.md`, using the question, the reference answer and the retrieved text, and returns `correct`, `supported` and a reason. Agreement with my own grades (`eval/human_labels.csv`) is reported as a percentage and Cohen's kappa.

The first judge run marked three correct refusals (q12, q13, q15) wrong because the rubric didn't allow the where-to-check line that AGENTS.md asks for. I clarified the rubric and re-ran the judge. First run: 87% agreement on correct, κ 0.63. Second run: 97% agreement on correct, κ 0.89.

> **TODO (yours):** your rubric, your agreement with the judge, and the results (correct, unsupported, correct refusals, wrong refusals) from `eval/summary.md`.

## Failures

Seven answers failed: five caught by both the judge and my grades, and two (q09, q28) only by my grades.

| Cause | Count | Questions |
|---|---|---|
| Retrieval miss | 3 | q06, q11, q28 |
| Unsupported claim | 2 | q09, q26 |
| Wrong refusal | 1 | q01 |
| Wrong answer from the right passage | 1 | q23 |
| Bad test question | 0 | none |

Retrieval misses are the largest group, and all three follow the same pattern. The agent found the passage for the first part of a multi-part question, answered from it, and never retrieved the passage holding the second part: the timing rules for q06, the requirement that the Student visa still be valid for q11, and the link between application date and visa length for q28. Its answers read as complete, so nothing signals the gap to the student. This matches the retrieval numbers: with the section design, hit@5 is 75% for single questions but 62% for multi.

Both unsupported claims add a rule the passages don't contain. In q26 the agent filled a gap (whether delivery riders are employees) with general knowledge, turning a conditional answer into a flat no. In q09 it invented an exception for work placements. The wrong refusal, q01, is the other side of the agent's caution: it had the right passage but refused anyway, probably because rule 4 says related background doesn't count as an answer. The same caution got all nine unanswerable questions right. q23 is a plain arithmetic slip.

No failure came from a bad test question, but writing the test set exposed weaknesses that I fixed or noted. I revised two references (q02, q07) after grading because the Student guide or caseworker guidance disagreed with Appendix Student. And q11's alternative locations for its second part weren't equivalent, since only one of them contains the rule that you must apply before your Student visa expires.

**What to fix first:** retrieval for multi-part questions, because it's the most common cause and the hardest for a student to spot. The simplest change is to rule 1: have the agent run a separate search for each part of the question and check that every part is covered before answering. After that, I'd loosen rule 4's related-background line and re-check q01 against the traps it currently refuses correctly.

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
