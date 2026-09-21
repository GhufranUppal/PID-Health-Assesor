# Tools the agent uses

The agent is granted a **minimal** tool set (`tools: [read, edit, search, execute,
todo]` in the agent frontmatter) plus the notebook and browser tools that VS Code
exposes. Each tool has a specific role in the workflow.

| Tool (alias) | When the agent uses it | Example |
|---|---|---|
| **read** | Read the field guide, the foundation module, existing episodes/CSVs, and prior numbered notebooks before writing anything. | Read `analyze_pid_two_minute.py` to reuse `load_cov`, `value_at`. |
| **search** | Locate helpers, constants, verdict strings, or the next free notebook number. | Grep for `SATURATION_VALUE` or `Integral windup`. |
| **edit** | Create/modify the `analyze_pid_<topic>.py` module, the numbered notebook, docs, and figures. | Write a new rolling-window analyzer that imports the foundation module. |
| **execute** | Run the analysis module and export notebooks. | `python run_analysis.py`; `nbconvert --to html --execute`. |
| **todo** | Track multi-step deliverables (module → notebook → export → verify). | A 4-item list per analysis. |
| **notebook tools** | Get the notebook summary, insert/edit cells, run cells. | Insert a worked-examples cell after the episodes cell. |
| **browser tools** | Optionally open the generated HTML report to confirm it renders (cards, chart, embedded source). | Open `PID/pid_5min_loop_assessment.html`. |

## Tool-use rules
- **read before edit.** Never modify a module without reading it first.
- **reuse, don't reinvent.** Import helpers from `analyze_pid_two_minute.py` instead of
  re-implementing loaders/metrics.
- **execute from the repo root** so `import analyze_pid_two_minute` resolves and the
  `PID/` data path is found.
- **portable interpreter.** Any Python 3.11+ with `requirements.txt` installed; do not
  hardcode an interpreter path.
- **never overwrite** prior numbered notebooks/reports — increment the number.

## What the agent does NOT need
- No network calls, no MCP servers, no package installs beyond `requirements.txt`.
- No write access outside the repo. Generated artifacts land in `PID/`.
