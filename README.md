# SqlServer.ScriptClassifier

Answers two questions about a folder of SQL Server scripts you didn't write: **what does each script do, and which ones write?** Every answer comes with the line of the script that justifies it.

> **Status: pre-alpha.** The design and build plan are filed as issues #1 to #6. No code yet. The commands below are the planned interface and may change.

## Why

On a new engagement you often inherit a folder of hundreds of undocumented scripts: maintenance jobs, one-off fixes, diagnostics someone copied from a blog in 2017. Before anyone runs them, or lets an agent run them, you need to know which are safe.

Reading them all by hand takes days. Grepping for `UPDATE` or `DROP` is fast but wrong in both directions: it flags keywords inside comments, and it misses writes hidden in dynamic SQL or behind an `IF`.

ScriptClassifier combines the two. Claude reads each script and fills a fixed schema; deterministic validators then check the answers against the script text, low-confidence results go to a human review queue, and every claim cites its source lines. The output is a manifest you can hand to a client, a DBA, or an automated system.

## What it produces

`classification.json`: one entry per script.

| Field | Values | Notes |
|---|---|---|
| `purpose` | text | One or two sentences |
| `operation_category` | `diagnostic`, `index_maintenance`, `statistics_maintenance`, `backup_restore`, `security`, `schema_analysis`, `data_fix`, `other`, `unclear` | `other` requires `category_detail` |
| `access` | `read_only`, `writes_data`, `writes_schema`, `writes_server_config`, `unclear` | The field that matters most |
| `objects_touched` | list of `{schema, name, kind}` | May be empty |
| `lock_risk` | `none`, `low`, `medium`, `high`, `unclear` | |
| `preconditions` | text or `null` | `null` when the script doesn't say |
| `parameters` | list or `null` | |
| `confidence` | per field | Drives review routing (#5) |
| `evidence` | path + line range + excerpt per key field | Provenance (#6) |
| `conflict_detected` | bool | Header comment and body disagree |

It also produces `review.md`: a checklist of low-confidence and conflicting entries for a human to confirm.

## How it works

| Step | What happens | Issue |
|---|---|---|
| 1. Extract | Claude is forced to call a `classify_script` tool whose schema is generated from a Pydantic model. The result is read from the `tool_use` block and validated back into the same model | #1 |
| 2. Guide | Few-shot examples for the hard cases: commented-out fix blocks, dynamic SQL, read-only on one branch only, no header | #2 |
| 3. Validate and retry | Deterministic checks, e.g. `read_only` but a write keyword appears outside comments. Failures are retried with the specific errors. Failures the script can't resolve (behaviour defined elsewhere) become `unclear`, not retries | #3 |
| 4. Batch | The full library goes through the Message Batches API, `custom_id` = script path. Only failed items are resubmitted | #4 |
| 5. Calibrate | Field-level confidence, checked against a hand-labelled set. Accuracy reported per category and per field | #5 |
| 6. Cite | Each key field carries path, line range and excerpt. Script commit and date recorded so reruns show changes, not contradictions | #6 |

## Design rules

- **Never executes SQL.** It reads text only. No database connection.
- **Null over guess.** Fields the script doesn't reveal are nullable, so the model never has to invent a value to satisfy the schema.
- **Validators outrank the model on `access`.** If a write statement exists outside comments, the script is not `read_only`, whatever the model said.
- **Batch for full runs.** Classification is non-blocking work, so it uses the Batches API (cheaper, asynchronous). Live calls are only for prompt iteration on small samples.

## Setup (planned)

Requires Python 3.10+ and an Anthropic API key.

```bash
python -m venv .venv
.venv\Scripts\activate          # Windows; use source .venv/bin/activate elsewhere
pip install -e .
set ANTHROPIC_API_KEY=...       # never commit it
```

## Usage (planned)

```bash
# Iterate on the prompt with live calls against a small sample
python -m scriptclassifier classify --source C:\Projects\SqlServer.EngineeringToolkit\Scripts --sample 5

# Classify the whole library through the Batches API
python -m scriptclassifier batch --source <scripts-folder> --out classification.json
```

## Limitations

- **Static analysis only.** It can't see what a called procedure does if that procedure lives elsewhere; those fields come back `unclear`.
- **Trust the numbers, not the vibe.** Until calibration (#5) is published here, treat the output as a draft for human review.
- **Script contents are sent to the Claude API.** Before running it on a client's scripts, get permission and check for embedded secrets (connection strings, passwords in `sp_addlinkedsrvlogin` calls). Strip them first.

## Related projects

- [SqlServer.EngineeringToolkit](https://github.com/CatFortman/SqlServer.EngineeringToolkit): the test corpus. Every script in it is classified and hand-checked.
- [SqlServer.OpsAgent](https://github.com/CatFortman/SqlServer.OpsAgent): its `remediate` mode reads `classification.json` to decide which Toolkit scripts an agent may propose.

## Learning context

Part of a hands-on lab for the Claude Certified Architect – Foundations exam. This repo covers structured output with `tool_use` (4.3), few-shot prompting (4.2), validation-retry (4.4), batch processing (4.5), confidence calibration (5.5) and provenance (5.6). Issues carry the `ccar-f` label.
