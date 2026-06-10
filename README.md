# TEST

An LLM-driven security analyst for Wazuh.

You ask a question in plain language, for example:

* "correlate the severity-12 events over the last 20 days"
* "is data being exfiltrated from my endpoints?"
* "are there any signs of compromise on this host?"

The agent uses an OpenAI-compatible chat API, typically through a LiteLLM proxy, to run an autonomous investigation. The model chooses which Wazuh tools to call, reads the results, iterates over the evidence, and stops when it can give a verdict. Every tool call is logged as an audit trail.

The project was originally built for Ollama, but the LLM integration has been moved to an OpenAI-compatible API. This makes it usable with LiteLLM, vLLM, OpenAI-compatible local gateways, or any backend exposed through the `/v1/chat/completions` interface.

## TODO

Tool calling depends on the backend model and serving stack.

If the model/backend supports OpenAI-style function calling, keep tools enabled:

```python
tools=TOOL_SCHEMAS,
tool_choice="auto",
```

If the backend does not support native tool calling yet, the request may fail or the model may ignore the tools. For example, vLLM requires automatic tool calling to be explicitly enabled when serving the model:

```bash
--enable-auto-tool-choice --tool-call-parser <parser>
```

For Qwen-family models, choose the parser that matches the model and vLLM version, for example `hermes` or a Qwen-specific parser where supported.

If native tool calling is not available, a fallback ReAct/JSON-in-text protocol could be implemented later, but native OpenAI-style tool calls are preferred.

## Architecture

Three files, one job each:

| File             | Role                                                     |
| ---------------- | -------------------------------------------------------- |
| `app.py`         | Web UI (Flask) + process control. The only file you run. |
| `agent_tools.py` | The agentic loop and the tools the model can call.       |
| `client.py`      | Data layer: Wazuh API + indexer access. No LLM logic.    |

The model reaches Wazuh indirectly through local Python tools.

Those tools access Wazuh in two ways:

* **Wazuh indexer API** (`:9200`, basic auth) for alerts, aggregations, timelines, event sequences, and vulnerability findings.
* **Wazuh server API** (`:55000`, token auth) for enrolled agents and live host inventory.

The LLM itself is reached through an **OpenAI-compatible API endpoint**, for example a LiteLLM proxy running in front of vLLM or another model backend.

## Requirements

* **Python 3.10+**
* **LiteLLM proxy** or another OpenAI-compatible API server reachable from the app.
* A model capable of reliable tool/function calling.
* A **Wazuh** deployment, including manager and indexer, reachable over the network.

Recommended model/backend notes:

* Small models can struggle with multi-step tool use.
* The backend must support OpenAI-compatible tool calls if `tools=TOOL_SCHEMAS` is enabled.
* If using vLLM behind LiteLLM, start vLLM with tool-calling support enabled, for example:

```bash
vllm serve <MODEL_NAME> \
  --host 0.0.0.0 \
  --port 8000 \
  --enable-auto-tool-choice \
  --tool-call-parser <parser>
```

The correct parser depends on the model family and vLLM version.

## Setup

### 1. Install dependencies

```bash
pip install flask flask_cors openai requests
```

If your project still has older Ollama references from a previous version, they can be removed unless another part of the code still imports `ollama`.

### 2. Create a `.env` file

Place a `.env` in the same directory as `app.py`.

```ini
# -- Wazuh API (token auth, port 55000) --
WAZUH_HOST=https://<WAZUH_SERVER_IP>:55000
WAZUH_USER=your_api_user
WAZUH_PASS=your_api_password
WAZUH_SSL=false       # true only if you verify TLS certs

# -- Wazuh Indexer (basic auth, port 9200) --
INDEXER_HOST=https://<WAZUH_INDEXER_IP>:9200
INDEXER_USER=admin
INDEXER_PASS=your_indexer_password

# -- OpenAI-compatible LLM API --
# Example: LiteLLM proxy
OA_HOST=http://<LITELLM_HOST>:4000/v1
OPENAI_API_KEY=your_litellm_or_gateway_key

# Model name as exposed by LiteLLM / the OpenAI-compatible gateway
AGENTIC_MODEL=Qwen3-8B

# -- Optional tuning --
AGENTIC_MAX_STEPS=18            # max model/tool-loop steps per investigation
UI_HOST=0.0.0.0                 # interface the web UI binds to
UI_PORT=5000                    # web UI port
```

Example LiteLLM-backed setup:

```ini
OA_HOST=http://10.237.14.252:4000/v1
OPENAI_API_KEY=sk-anything
AGENTIC_MODEL=Qwen3-8B
```

The API key value depends on your LiteLLM configuration. If LiteLLM authentication is disabled, some deployments still require a placeholder value because the OpenAI Python client expects one.

## LiteLLM / vLLM notes

This application sends requests to:

```text
POST /v1/chat/completions
```

When tool calling is enabled, the request includes:

```python
tools=TOOL_SCHEMAS
tool_choice="auto"
```

If the backend is vLLM and it is not started with automatic tool calling enabled, you may see an error like:

```text
"auto" tool choice requires --enable-auto-tool-choice and --tool-call-parser to be set
```

That is a backend serving configuration issue, not a Python syntax issue.

Fix it by enabling tool calling on the vLLM server, for example:

```bash
vllm serve <MODEL_NAME> \
  --host 0.0.0.0 \
  --port 8000 \
  --enable-auto-tool-choice \
  --tool-call-parser <parser>
```

Then point LiteLLM at that vLLM backend and point this app at LiteLLM through `OA_HOST`.

## Running

`app.py` has built-in process control - no `nohup`, no PID hunting.

```bash
python3 app.py start      # launch in the background (writes agent.pid + agent.log)
python3 app.py status     # is it running? on which port?
python3 app.py stop       # stop the background process cleanly
python3 app.py restart    # stop + start
python3 app.py run        # run in the FOREGROUND (Ctrl+C to quit; use for systemd)
```

Both spellings work:

```bash
python3 app.py stop
python3 app.py --stop
```

Then open the UI:

```text
http://<host>:5000
```

## Using it

* **Run tab** - type a question, click **Run now**, confirm, and watch the agent investigate live. Its tool calls, intermediate reasoning, and final verdict stream into the UI. Click **Stop** to cancel a running investigation.
* **Reports tab** - every completed investigation is saved with its verdict and full tool-call audit trail. Reports render as formatted markdown.
* **Auto-run** - toggle the scheduler to fire a triage automatically every N hours over the last M hours of events.

## How the agent works

The model is given a system prompt and a set of available Wazuh investigation tools.

Typical flow:

1. Start broad with alert aggregation or search.
2. Identify high-severity rule groups, suspicious agents, or repeated indicators.
3. Drill into timelines, event sequences, vulnerabilities, inventory, or cross-host correlations.
4. Stop after enough evidence has been gathered.
5. Produce a final verdict with supporting counts, agents, severities, timestamps, and recommended actions.

The Python process executes the tools locally. The model only decides which tool to call and how to interpret the results.

## Troubleshooting

### `400 Bad Request` with `"auto" tool choice requires --enable-auto-tool-choice`

Your vLLM backend does not have automatic tool calling enabled.

Start vLLM with:

```bash
--enable-auto-tool-choice --tool-call-parser <parser>
```

Then restart LiteLLM if needed.

### The model answers without using tools

Check that:

* `tools=TOOL_SCHEMAS` is enabled in the chat completion call.
* `tool_choice="auto"` is enabled if your backend requires it.
* The model supports tool/function calling.
* LiteLLM is routing the request to the expected model group.
* vLLM or the backend server is started with tool-calling support.

### LiteLLM returns model group errors

Verify that `AGENTIC_MODEL` exactly matches the model name exposed by LiteLLM.

For example, if LiteLLM exposes the model group as:

```text
Qwen3-8B
```

then `.env` should contain:

```ini
AGENTIC_MODEL=Qwen3-8B
```

### Connection errors

Verify:

```ini
OA_HOST=http://<LITELLM_HOST>:4000/v1
```

and test the endpoint manually:

```bash
curl http://<LITELLM_HOST>:4000/v1/models \
  -H "Authorization: Bearer <OPENAI_API_KEY>"
```

## Areas for improvement

If you want to extend this, good directions include:

* **More tools.** A `get_alert_detail` tool to pull one full event with every field, a `compare_time_windows` tool for this week vs. baseline, or an active-response hook to isolate a host would extend what the agent can do.
* **Model selection in the UI.** Let the analyst pick the model per run instead of editing `.env`.
* **Backend capability checks.** Add a startup check that verifies whether the configured model supports tool calling before the first investigation.
* **Tool-calling fallback mode.** Add a ReAct/JSON fallback for models that cannot emit native OpenAI tool calls.
* **Surface the audit trail as a collapsible panel** in the live view, rather than only appending it to the saved report.
* **Persisted, searchable history.** History is currently a JSON file capped at 50 runs; a small database would scale better and allow search.
* **Tighter convergence / cost control.** Larger models explore widely; a per-run tool-call budget surfaced in the UI, or a "quick vs. thorough" mode, would give users control over time and cost.
* **Multi-node / multi-tenant Wazuh.** The data layer assumes one manager and one indexer; supporting clusters would broaden its reach.
