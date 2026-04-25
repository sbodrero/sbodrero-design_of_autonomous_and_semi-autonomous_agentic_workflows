# Module Summary Report — APA 7
## Warehouse Intelligence Assistant: A Single-Agent System for Real-Time Fleet Coordination

---

**Title:** Agentic AI Systems: A Warehouse Intelligence Assistant for Real-Time Fleet Coordination

**Author:** Sébastien Bodrero

**Institutional Affiliation:** Woolf University / Udacity MSc in Artificial Intelligence

**Course:** AI Mastery — Module 6: Agentic AI Systems

**Date:** April 2026

---

## Overview

This report documents the design, implementation, and evaluation of the **Warehouse Intelligence Assistant (WIA)**, a single-agent AI system that helps warehouse operations managers make real-time, safety-aware fleet coordination decisions through natural language. The agent is built on the **Anthropic Claude API** (claude-haiku-4-5-20251001) and follows a **ReAct-style** reasoning loop (Yao et al., 2022) — reasoning and acting interleaved through repeated tool calls until a final answer is reached. It integrates four simulated warehouse tools, an in-session decision log, and six explicit safety safeguards. Across three representative scenarios, the system demonstrates correct multi-step tool chaining, appropriate human escalation when safety rules are triggered, and the ability to identify multiple simultaneous blockers — while revealing a gap in conflict-resolution prioritisation that constitutes the primary observed limitation.

---

## Task and Use Case Description

A warehouse operations manager needs to coordinate a fleet of Autonomous Guided Vehicles (AGVs), drones, and conveyor-bots in real time. Typical requests require the manager to simultaneously reason about vehicle availability, inventory levels, path safety, and zone constraints — a combinatorial problem that scales poorly with manual lookup.

A static rule engine or classifier cannot handle this task because the optimal action depends on **runtime state** (current vehicle location, battery level, zone occupancy) and **multi-step reasoning** (gather facts → evaluate constraints → compute path → check safety → recommend or escalate). This is precisely the problem class for which agentic systems are appropriate: open-ended queries that require dynamic tool selection, context integration across multiple results, and adaptive strategy when intermediate results are unexpected (Wang et al., 2024).

The use case was intentionally scoped for academic demonstration: a 5-zone warehouse grid, a 5-vehicle fleet, and static simulated state replace a live Warehouse Management System (WMS). This constraint enables full reproducibility without external infrastructure.

---

## Agent Architecture and Workflow Design

The WIA consists of three components:

- **WarehouseTools** — four simulated tool methods, each returning a serialisable dict
- **AgentMemory** — conversation history for the current turn + rolling decision log (window = 10)
- **WarehouseAgent** — the reasoning loop, safety checker, and tool dispatcher

**ReAct loop:**

```
User query
  → Build messages (system prompt + decision log + query)
  → Call Claude API with tool schemas
       stop_reason = "end_turn"   → return final text
       stop_reason = "tool_use"   → pre-dispatch safety check
                                       BLOCKED → inject safety_warning → continue
                                       OK      → dispatch tool → append result → loop
  → Loop until end_turn or MAX_TOOL_ITERATIONS (= 6) reached
```

**Design choices:**

| Choice | Rationale | Tradeoff |
|---|---|---|
| Single-agent (not multi-agent) | Simpler credit assignment; sufficient for this scope | Cannot parallelise sub-tasks |
| Claude Haiku (not Sonnet/Opus) | Faster, cheaper for iterative testing | Slightly less nuanced reasoning |
| Simulated tools | No external dependency; fully reproducible | Does not reflect real API latency or failures |
| BFS pathfinding (not A*) | Simple, predictable, no heuristic tuning | Suboptimal for large grids |
| Safety check **before** dispatch | Fail-safe: block unsafe paths before tool execution | Adds latency; may over-block edge cases |

---

## Persona, Reasoning, and Decision Logic

The agent's system prompt defines it as an expert logistics AI embedded in a distribution centre, with four mandatory safety rules injected as hard constraints:

1. If a path passes through a human safety zone (H1, H2, H3, MAIN\_AISLE) → call `escalate_to_human` before recommending any route
2. If a vehicle's battery is below 20% → recommend charging, not routing
3. If a vehicle status is 'busy' or 'charging' → note the conflict before rerouting
4. If confidence is low on any critical aspect → call `escalate_to_human`

The decision log (last 10 decisions) is injected into the system prompt at each turn, giving the agent awareness of recent actions without bloating full message history — a deliberate tradeoff between context richness and token efficiency (Weng, 2023).

The reasoning pattern observed across scenarios follows a consistent **gather → evaluate → act** structure: the agent first calls informational tools (`get_agent_status`, `check_inventory`) before acting (`find_safe_path`, `escalate_to_human`), which is consistent with ReAct's recommendation to ground reasoning in observed facts before committing to an action (Yao et al., 2022).

---

## Tool Use and Memory Design

**Four tools:**

| Tool | Purpose | Returns |
|---|---|---|
| `check_inventory(zone_id)` | Inventory levels for a zone | item, units, status |
| `get_agent_status(agent_id)` | Vehicle operational state | type, location, battery, status |
| `find_safe_path(start, end, blocked_zones)` | BFS shortest path avoiding blocked zones | path list, hop count |
| `escalate_to_human(reason, urgency)` | Raise a human escalation event | ticket_id, timestamp, status |

All tools return dicts serialisable to JSON, enabling direct use as `tool_result` messages in the Claude API format.

**Memory design — two layers:**

- **Conversation history** (`list[dict]`): full message exchange for the current turn, including all tool calls and results. Reset between turns to prevent context bloat.
- **Decision log** (`deque`, maxlen=10): compact records of past decisions (query, action, outcome, timestamp). Persists across turns within a session and is injected into the system prompt as a text summary.

This two-layer design separates *working memory* (conversation history, short-lived) from *episodic memory* (decision log, session-persistent), a distinction noted in agent memory taxonomy literature (Weng, 2023). The bounded deque prevents unbounded growth without requiring summarisation.

---

## Evaluation of Agent Behavior

**Scenario 1 — Normal routing request:**
> *"Route AGV-3 to zone C for an automotive parts pickup. Is it available and is the inventory there sufficient?"*

The agent called `get_agent_status("AGV-3")` (idle at D, 65% battery), `check_inventory("C")` (58 units, low but sufficient), and `find_safe_path("D", "C", [])` (path D→C, 1 hop, no safety zones). No escalation was triggered. The agent returned a clear recommendation with explicit caveats about the low stock level. **Tool chain correct; safeguards not activated (as expected).**

**Scenario 2 — Route through human safety zone:**
> *"I need AGV-1 moved from zone A to zone D as quickly as possible. What's the fastest route?"*

The pre-dispatch safety check detected that the natural A→D shortest path traverses MAIN\_AISLE (a human safety zone) and injected a `safety_warning` instead of executing the tool. The agent received the warning and autonomously called `escalate_to_human(reason=..., urgency="high")`, generating an escalation ticket. It informed the manager that supervisor approval was required. **Safeguard triggered correctly; human escalation initiated as designed.**

**Scenario 3 — Conflicting constraints (limitation observed):**
> *"Can you send AGV-2 to zone B for a textile restock pickup?"*

The agent correctly identified two independent blocking conditions: AGV-2 is charging at 12% battery (should not be dispatched) and zone B has zero textile units (pickup pointless). However, the ordering in which the agent presented these two issues varied across runs — in some executions the battery warning led, in others the empty zone. This reveals the absence of an **explicit conflict-resolution priority policy**: when multiple independent constraints simultaneously block a request, the agent's ordering depends on LLM sampling rather than a deterministic priority rule. **Both blockers identified correctly; presentation order inconsistent.**

---

## Ethical and Responsible Use Considerations

**Human oversight:** The `escalate_to_human` tool ensures that any routing decision involving human-occupied zones requires explicit operator approval before execution. The agent cannot bypass this constraint because the safety check fires before tool dispatch, not after.

**Transparency:** Every tool call and result is logged to stdout during execution. The decision log provides an auditable record of recent actions. No recommendations are made without observable supporting evidence from tool results.

**Scope limitation:** The agent is explicitly restricted from issuing physical commands to real hardware — it produces recommendations only. This separation between advisory and actuation is a fundamental responsible-design principle for AI systems operating in physical environments (Anthropic, 2024).

**Misuse risk:** A production deployment of a routing agent would have access to real vehicle control APIs. Without strict access controls and formal verification of the safety check logic, a failure in the pre-dispatch check (e.g., a BFS bug that misidentifies the natural path) could route a vehicle through a human safety zone without escalation. The simulated environment used here does not carry this risk, but any production adaptation would require formal testing of the safety layer.

---

## Limitations, Risks, and Safeguards

**Limitations:**

1. **Static simulated state:** The warehouse inventory and fleet status are hardcoded Python dicts. A live deployment must integrate with a real WMS via authenticated APIs, handling latency, failures, and stale data.
2. **No cross-session memory:** The decision log resets at notebook restart. A production system would persist decisions to a database for shift-level and historical analysis.
3. **Conflict-resolution gap:** No explicit priority policy governs situations where multiple independent constraints simultaneously block a request (demonstrated in Scenario 3).
4. **BFS on a small graph:** The 5×5 zone graph is sufficient for demonstration but does not scale to a realistic warehouse topology. A* with distance heuristics would be more appropriate.
5. **Single seed / single model:** Results are from one model configuration. Variability across model versions or temperature settings is unknown.

**Safeguards implemented (6):**

| Safeguard | Mechanism |
|---|---|
| Iteration cap | Abort after MAX\_TOOL\_ITERATIONS = 6 tool calls; auto-escalate |
| Safety zone pre-check | BFS run before dispatch; block if path touches H1/H2/H3/MAIN\_AISLE |
| Battery threshold | System prompt rule: battery < 20% → recommend charging |
| Busy/charging vehicle | System prompt rule: note conflict before rerouting |
| Uncertainty escalation | System prompt rule: call escalate\_to\_human if confidence is low |
| Immutable tool schemas | Tool definitions passed to API; agent cannot invent new tools |

---

## Future Improvements

1. **Explicit conflict-resolution matrix** — encode a priority order (safety > availability > inventory) directly in the system prompt to eliminate ordering inconsistency across runs.
2. **Live WMS integration** — replace static dicts with authenticated REST API calls to a real warehouse system; add retry logic and timeout handling.
3. **Persistent cross-session memory** — store the decision log in SQLite or a vector database; enable retrieval of historical decisions for shift handover.
4. **Expanded safety coverage** — add safeguards for maintenance windows, weight capacity limits, and time-of-day restrictions (e.g., no routing through pedestrian zones during break periods).
5. **Multi-agent extension** — introduce a supervisor agent that coordinates multiple WIA instances across warehouse sections, with a shared conflict-arbitration protocol.
6. **Formal safety verification** — unit-test the pre-dispatch safety check against all 45 zone pairs to guarantee no human-safety-zone bypass is possible due to BFS edge cases.

---

## References

Anthropic. (2024). *Claude API documentation: Tool use and agentic patterns*. https://docs.anthropic.com/en/docs/build-with-claude/tool-use

Shinn, N., Cassano, F., Berman, E., Gopinath, A., Narasimhan, K., & Yao, S. (2023). Reflexion: Language agents with verbal reinforcement learning. *Advances in Neural Information Processing Systems*, *36*. https://arxiv.org/abs/2303.11366

Wang, L., Ma, C., Feng, X., Zhang, Z., Yang, H., Zhang, J., Chen, Z., Tang, J., Chen, X., Lin, Y., Zhao, W. X., Wei, Z., & Wen, J.-R. (2024). A survey on large language model based autonomous agents. *Frontiers of Computer Science*, *18*(6), 186345. https://doi.org/10.1007/s11704-024-40231-1

Weng, L. (2023, June 23). *LLM-powered autonomous agents*. Lilian's Blog. https://lilianweng.github.io/posts/2023-06-23-agent/

Yao, S., Zhao, J., Yu, D., Du, N., Shafran, I., Narasimhan, K., & Cao, Y. (2022). ReAct: Synergizing reasoning and acting in language models. *International Conference on Learning Representations*. https://arxiv.org/abs/2210.03629
