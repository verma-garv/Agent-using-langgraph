# Agent Using LangGraph

A calculator-tool agent built with LangGraph's Graph API, using Groq as the LLM
provider. This is a deliberate rebuild of `agent-from-scratch/` — same agent
loop (LLM call → check for tool calls → execute tool → feed result back →
repeat until done), this time expressed as a graph of nodes and conditional
edges instead of a hand-written `while True:` loop.

## Why this exists

Built after implementing the agent loop from scratch with no framework, to see
directly what LangGraph abstracts and what it doesn't. Short answer: nothing
conceptually new — the same five moving pieces (LLM call, tool-call check, tool
execution, state update, loop-back) just get named and wired explicitly as a
graph instead of being inline control flow.

| Raw loop (`agent-from-scratch/`) | LangGraph equivalent |
|---|---|
| `client.chat.completions.create(...)` | `llm_call` node |
| `if assistant_message.tool_calls:` | `should_continue` conditional edge |
| manually looping tool calls + appending results | `tool_node` |
| `messages.append(...)` | automatic state updates returned by each node |
| loop repeating to the top | `add_edge("tool_node", "llm_call")` |

## Setup

```bash
python -m venv venv
.\venv\Scripts\Activate.ps1      # PowerShell
pip install langchain langchain-groq langgraph python-dotenv
```

Create a `.env` file (never committed — see `.env.example`):
```
GROQ_API_KEY=your_actual_groq_key_here
```

## Run

```bash
python Agent-using-langchain.py
```

## Notes

- Model: `openai/gpt-oss-120b` via Groq, with `model_provider="groq"` passed
  explicitly to `init_chat_model` (its name-based auto-inference guesses
  "openai" from the `openai/` prefix in the model string, which is wrong here).
- Calculator tool reused from `agent-from-scratch/tools.py`'s design (raw
  string expression in, parsed on the Python side).

## Running on a local model (Ollama)

Set `MODEL_PROVIDER=ollama` in `.env` (default is `groq`) and make sure Ollama
is running with `llama3.2:1b` pulled (`pip install langchain-ollama`).

### Findings: same graph, two models

| Query | Groq `gpt-oss-120b` | Ollama `llama3.2:1b` (Q8_0) |
|---|---|---|
| Multiply 256 and 756 | Valid tool call, correct (193,536) | Valid tool call, correct, noticeably slower |
| 15 times 12, plus 100 | not run | Malformed tool call, answered in plain text (280, correct by luck, no tool used) |
| 847 times 293, plus 1000 | not run | Malformed tool call, answered wrong (249,611 vs correct 249,171), no tool used |

What this showed:
- The 1B model can handle a simple tool call but fails at producing valid
  structured arguments on harder queries: it copies the tool's schema into the
  arguments instead of filling in values.
- When the tool path fails, the model falls back to doing the math itself,
  confidently and wrongly. This is the reason the calculator tool exists.
- Returning the tool error to the model did not make it retry correctly.
- Model size is the main variable for tool-calling reliability here, not quantization.
