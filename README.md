# Bug Report Triage Agent

Udacity Master's in AI, Capstone Project: *Design of Autonomous and Semi-Autonomous Agentic Workflows*

A multi-agent workflow that does a first pass of triage on Eclipse bug reports. For each new report, it returns one suggestion:

| Outcome | Meaning |
|---|---|
| `rejected` | The text is a question or is not about Eclipse |
| `duplicate` | The report describes a bug that is already known |
| `auto_triaged` | The system proposes a severity and a component |
| `needs_human_review` | The system is not confident, or a step failed |

The system only **suggests**. It never edits or closes a bug.

## How it works

Three agents are connected in a fixed LangGraph workflow. All three use Claude Haiku 4.5 (`temperature=0`) with structured output (Pydantic).

```
report -> Router -> Duplicate-finder -> Triage -> decide -> auto_triaged
              |             |                        |
          rejected      duplicate            needs_human_review
```

- **Router**: decides if the text is a `bug`, a `question` or `out_of_scope`.
- **Duplicate-finder**: calls a TF-IDF search tool (top 3 similar bugs from a 500-bug index), then the LLM judges whether one of them is the same problem.
- **Triage**: proposes a severity (`low`, `normal`, `high`), a component (`UI`, `Core`, `SWT`, `TPTP`, `Releng`, `Other`) and a confidence.
- **decide**: plain code. A confidence below 0.5 sends the report to human review.
- **Memory**: the graph state holds one report; a session list stores processed reports and returns the stored result for a repeated bug id.
- **Safeguards**: confidence threshold, step limit, error fallback to human review, id check on duplicates, a log file, and suggest-only output.

## Project files

| File or folder | Content |
|---|---|
| `agentic_system.ipynb` | Main notebook: agents, tool, graph, safeguards, tests, evaluation, diagrams |
| `data_prep.ipynb` | Prepares the data (supporting file) |
| `data/index.csv` | 500 bugs used as the search index |
| `data/test.csv` | 21 test reports with the expected outcome of each one |
| `diagrams/` | Architecture diagrams (`graph.mmd`, `graph.png`, `architecture.mmd`, `architecture.png`) |
| `logs/triage.log` | Log of the last evaluation: one line per node call |
| `Agentic_AI_System_Design_Report.pdf` | The project report |
| `requirements.txt` | Python packages, from `pip freeze` |

## How to run

1. Create and activate a Python environment, then install the packages:
   ```
   pip install -r requirements.txt
   ```
2. Create a `.env` file in the project folder with your Anthropic API key:
   ```
   ANTHROPIC_API_KEY=your_key_here
   ```
   The `.env` file is not part of the submission and is listed in `.gitignore`.
3. Open `agentic_system.ipynb` and use **Restart and Run All**. Run it from the project folder, so the relative paths `data/`, `logs/` and `diagrams/` work.

The notebook makes roughly 150 to 200 calls to Claude Haiku 4.5. The cost is small.

## Configuration

- Model: `claude-haiku-4-5-20251001`, `temperature=0`, `max_tokens=1000`
- Confidence threshold: `CONFIDENCE_THRESHOLD = 0.5` (fixed before the test, not tuned on the test reports)
- Step limit: `MAX_STEPS = 4`
- Prompts: defined in the notebook cells (Router, Duplicate-finder, and Triage in two versions, `TRIAGE_PROMPT_V1` and `TRIAGE_PROMPT_V2`)
- Data: the bug data comes from an earlier course project and is prepared in `data_prep.ipynb`. Names of assignees and reporters were removed.

## Results (final saved run)

- All 21 test reports ended with the expected decision type. The 3 duplicates were matched to their originals.
- On the 15 sampled reports: severity 9 / 15 correct, component 13 / 15 correct. On this balanced set, always answering `normal` gives 5 / 15.
- All 6 reports with a wrong label were auto-triaged, because the model's confidence was 0.70 or higher. Only the vague report went to human review.

Results can differ slightly between executions, even with `temperature=0`. See the report for details.

## Known limitations

- Small and easy test set (21 reports, hand-written duplicates and edge cases).
- The search index has only fixed bugs, and TF-IDF matches words, not meaning.
- The model's own confidence is weak evidence that a label is correct.
- The product name `Z_ARCHIVED` appears only in TPTP bugs. It is a dataset shortcut, so the Triage agent does not receive the product name.

## AI assistance

Claude (Anthropic) was used for planning, review and drafting parts of the code, prompts and text. The design decisions, all runs and all results are my own, and the report states this in more detail.
