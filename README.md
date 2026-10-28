# Antibiotic Resistance Investigator

A multi-agent biological research assistant built with Python, AutoGen (AG2 Classic) and Gemini (`gemini-3.1-flash-lite`) in Google Colab. You ask a question about antibiotic resistance, the agents collect evidence from PubMed, UniProt and ClinicalTrials.gov, and the system writes a report with evidence, sources and limitations.

I chose antibiotic resistance because it is a real problem, it has a lot of public data, and it can be studied from broad to specific. Every question goes through five levels: (1) foundations, (2) mechanisms, (3) bacteria and antibiotics, (4) specific diseases (UTI, TB, gonorrhoea, MRSA), (5) prevention.

## 1. Multi-agent system with clear responsibilities

| Agent | Responsibility | Tools it can request |
|---|---|---|
| Researcher | Finds and reads papers and trials, one level at a time | `search_papers`, `get_paper_details`, `search_clinical_trials` |
| Analyst | Plans the five levels, judges if the evidence is enough, checks resistance proteins | `lookup_resistance_protein` |
| Reviewer_Writer | Writes the final report, checks it | `validate_report` |
| UserProxy (no LLM) | Runs every tool, checks the plan, sends reminders | runs all five |

The AI agents only request tools. The UserProxy is the only one that runs them (`code_execution_config=False`), so every call goes through one place where it is logged.

## 2. Five registered tools

1. `search_papers`: PubMed search (ID, year, journal, title).
2. `get_paper_details`: one PubMed paper (authors, journal, year, link, abstract).
3. `lookup_resistance_protein`: UniProt, reviewed entries only (protein, gene, organism, function, link).
4. `search_clinical_trials`: ClinicalTrials.gov (status, phase, conditions, interventions). A trial is a registration, not a result.
5. `validate_report`: plain-code check of the final report.

Every tool takes a `reason` (one sentence) and returns errors as text starting with `ERROR:`, so the agent can change its request. Everything is saved in a trace (request, reason, result, error).

## 3. Collaboration to investigate and evaluate evidence

A `GroupChat` with my own `pick_next_speaker` function decides who speaks, not the LLM:

1. The Analyst writes a five-level plan. A code check accepts or rejects it (3 rejections stop the run).
2. For each level the Analyst gives the question, the Researcher requests tools and reports, and the Analyst decides if the evidence is enough or sends it back (max 2 times).
3. The Analyst sends a summary to the Reviewer_Writer, who writes the report and runs `validate_report` (max 3 tries).

**Why a user-defined workflow:** no routing loops or skipped steps; hard code checks between phases; easy tracing of which agent failed; bounded use of the free API (120 messages, 5 reminders); each agent has one job.
**What it costs:** it depends on exact markers like `LEVEL 3`; it is rigid (forward only through five levels); agents have less freedom; I maintain the routing code myself;

## 4. Reports with supporting evidence, sources and limitations

Every report has four headings: **Question, Findings, Evidence / Sources, Limitations**.

- **Supporting evidence:** each factual sentence in Findings needs a citation, going from broad to specific.
- **Sources:** cited as `[PMID 12345678]`, `[NCT01234567]` or `[UniProt P37958]`.
- **Limitations:** at least 20 words, and must mention missing levels, that only abstracts were read, that UniProt reviewed entries can miss new proteins, that trials are not results, and that the checks have limits.

`validate_report` checks the four sections, the length of Limitations, that there is at least one citation, and that every cited ID was really retrieved by a tool (papers must be opened, not only seen in search). It cannot check whether a source really supports a claim.

## 5. Testing on multiple questions

I evaluate with automatic measures (validate_report result and tries, levels covered, tool requests and errors, messages). 

| Run | Question | validate_report | tries | levels | tool errors | messages |
|---|---|---|---|---|---|---|
| 1 | MRSA | PASS | 1 | 5 of 5 | 1 | 57 |
| 2 | Resistance vs treatment failure | FAIL or none | 0 | 5 of 5 | 7 | 58 |
| 2b | Same question, re-run | PASS | 2 | 3 of 5 | 3 | 67 |
| 3 | Mutation and gene transfer | FAIL or none | 0 | 5 of 5 | 3 | 51 |
| 4 | *E. coli* and UTI | PASS | 1 | 4 of 5 | 2 | 62 |
| 5 | Tuberculosis | PASS | 2 | 5 of 5 | 3 | 60 |
| 6 | Gonorrhoea vs MRSA | FAIL or none | 1 | 3 of 5 | 0 | 19 |

4 of 7 runs gave a passing report. Failed runs are kept. What worked: tool requests and the trace, recovery from tool errors, and the citation check (it made agents open papers they had only seen in search, in Runs 2b and 5). What did not: Runs 2 and 3 ended without a report.

## 6. Limitations

- A passing `validate_report` does not mean the claims are true.
- Only abstracts are read, not full papers. UniProt reviewed entries only. Trials are registrations, not results.
- The Reviewer_Writer is an AI checking AI-written text. No expert has checked the biology.
- Gemini free tier often returns 503, so some runs end early and results change between runs.
- The workflow is fixed (five levels, forward only). And `HUMAN_APPROVAL = True` is tested.

## 7. How to run

1. Open the notebook in Google Colab.
2. Add your Gemini key to Colab Secrets as `GEMINI_API_KEY` (never put it in a cell).
3. Run the cells from top to bottom.
4. Run `run_test("Run name", "Your question")` and then `show_table()`.


