# Module 6 — Agentic AI Systems: Warehouse Intelligence Assistant (WIA)

**Author:** Sébastien Bodrero  
**Programme:** Woolf University / Udacity MSc in Artificial Intelligence  
**Module:** Agentic AI Systems (Module 6)

---

## Project Overview

The **Warehouse Intelligence Assistant (WIA)** is a single-agent AI system that helps warehouse
operations managers make real-time, safety-aware fleet coordination decisions through natural
language. The agent uses the Claude API (Haiku model) in a ReAct-style reasoning loop, calling
four simulated warehouse tools to gather context before formulating a recommendation or escalating
to a human supervisor.

---

## Deliverables

| File | Description |
|------|-------------|
| `agentic_system.ipynb` | Main notebook — all 7 project tasks |
| `Agentic_AI_System_Design_Report.html` | Full design report → print to PDF for submission |
| `oral_review.html` | Bilingual (FR/EN) oral exam preparation |
| `requirements.txt` | Python dependencies (generated via `pip freeze`) |

---

## Setup

```bash
# From the p6/ directory
source .venv/bin/activate
pip install -r requirements.txt

# Set your Anthropic API key
export ANTHROPIC_API_KEY="your-key-here" or in Task 4 os.environ["ANTHROPIC_API_KEY"]="your-key-here"
```

## Run the notebook

```bash
jupyter lab agentic_system.ipynb
```

## Validate notebook executes top-to-bottom without errors

```bash
jupyter nbconvert --to notebook --execute agentic_system.ipynb \
  --output agentic_system_executed.ipynb
```

---

## Agent Architecture

```
WarehouseAgent (ReAct loop)
  ├── AgentMemory (conversation history + decision log)
  ├── WarehouseTools
  │     ├── check_inventory(zone_id)
  │     ├── get_agent_status(agent_id)
  │     ├── find_safe_path(start, end, blocked_zones)   ← BFS
  │     └── escalate_to_human(reason, urgency)
  └── Anthropic Claude API (claude-haiku-4-5-20251001)
```

**Safeguards:** human safety zone pre-check · iteration cap (6) · decision logging · low-battery rule

---

## Requirements

- Python 3.9+
- `ANTHROPIC_API_KEY` environment variable set
- See `requirements.txt` for full dependency list
