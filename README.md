# OpenBSA

**Open Battery Simulation Agent** is a local-workstation application for configuring, running, and inspecting battery simulations. It combines a Dash web interface, PyBaMM-based simulation workers, interactive Plotly results, and an optional AI assistant that can inspect configurations and (when explicitly enabled) submit bounded simulation jobs.
<img width="2050" height="1488" alt="image" src="https://github.com/user-attachments/assets/6a0008ce-371e-4e4e-a4bd-8683e8f957ae" />

> **Project status:** OpenBSA is under active development. A successful numerical solve does not establish accuracy for a particular physical cell. Review [scientific scope and limitations](docs/SCIENTIFIC_SCOPE.md) before using results for engineering decisions.
<img width="2050" height="1488" alt="image" src="https://github.com/user-attachments/assets/a73fd392-3f1f-4fcb-83a7-b60c841fa7e5" />

## Contents

- [Features](#features)
- [Requirements](#requirements)
- [Install and run](#install-and-run)
- [AI providers and local models](#ai-providers-and-local-models)
- [Using the application](#using-the-application)
- [Architecture](#architecture)
- [Data, privacy, and security](#data-privacy-and-security)
- [Configuration](#configuration)
- [Development and tests](#development-and-tests)
- [Documentation](#documentation)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Features

### Battery setup and simulation

- Select cylindrical, prismatic, pouch, or coin geometry and configure cell dimensions and parameters.
- Choose supported electrochemical models: DFN, SPMe, SPM, or ECM. The default model is DFN.
- Select supported anode/cathode material combinations and configure supported degradation features.
- Configure CCCV charging, pulse, GITT, or custom protocols. Custom step and drive-cycle CSV formats are documented in [scientific scope](docs/SCIENTIFIC_SCOPE.md).
- Run simulations in isolated worker processes with job status, cancellation, timeout handling, and session ownership.
- Use interactive plots, measured result summaries, analysis tools, and export features.
- Configure supported battery pack simulations when the optional pack dependencies are installed.

Supported combinations depend on the model, chemistry, geometry, and protocol. Unsupported combinations are intended to fail validation instead of silently substituting a different physical model. See [scientific scope](docs/SCIENTIFIC_SCOPE.md), [coin half-cell behavior](docs/COIN_HALF_CELL.md), and [degradation assumptions](docs/MODULAR_DEGRADATION.md).

### AI assistant

- Use local **Ollama** or **llama.cpp** servers, or the cloud **OpenAI**, **Claude**, and **Gemini** APIs.
- Discover provider models from the AI settings dialog; connection settings are saved as provider-specific profiles.
- Use the assistant for battery questions, configuration review, validation, and measured-result summaries.
- Select chat-only, inspect-and-validate, or bounded-run permissions in the assistant's skills/tools settings.
- Enable optional scientific workflow instructions for model selection, numerical verification, and measured-data comparison.
- View streamed responses where supported and render inline/display math with MathJax.

AI simulation execution is subject to explicit tool mode, run count, provider-round, time, and cycle limits. The AI uses the same validation, job queue, worker, and result pipeline as UI-submitted runs. It cannot execute arbitrary Python or shell commands. Details and provider-specific caveats are in [AI providers and skills](docs/AI_PROVIDERS_AND_SKILLS.md).

## Requirements

- Windows, macOS, or Linux workstation.
- Python > **3.12**.
- pip and a virtual environment.
- Internet access for Python package installation. PyBaMM's optional solver backends may have platform-specific requirements; see [PyBaMM installation](https://docs.pybamm.org/en/stable/source/user_guide/installation/index.html) if installation reports solver errors.
- A modern browser. The web application binds to `127.0.0.1:8050` by default.
- Optional: a separately installed/running Ollama or llama.cpp server, or API credentials for a cloud provider.

The core application does not require an AI model. The AI settings dialog is configured separately after the app starts.

## Install and run

### Windows PowerShell

From the project directory:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -e ".[dev]" -c constraints-tested.txt
python cli.py --server --host 127.0.0.1 --port 8050
```

If PowerShell blocks virtual-environment activation, use the environment's Python directly instead:

```powershell
.\.venv\Scripts\python.exe -m pip install -e ".[dev]" -c constraints-tested.txt
.\.venv\Scripts\python.exe cli.py --server --host 127.0.0.1 --port 8050
```

Open **http://127.0.0.1:8050** in your browser. Stop the server with `Ctrl+C` in its terminal.

### macOS or Linux

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e ".[dev]" -c constraints-tested.txt
python cli.py --server --host 127.0.0.1 --port 8050
```

### CLI options

The `openbsa` command is installed by the editable package. With the virtual environment active, these commands are equivalent:

```bash
openbsa --server --host 127.0.0.1 --port 8050
openbsa --version
```

`python app.py` is also available for direct development startup. For routine local use, bind to `127.0.0.1`; binding to `0.0.0.0` exposes the app to other machines on the network and is not an authenticated multi-user deployment.

### Optional pack dependencies

The optional pack simulation dependencies are not installed by the basic development extra. Install them only if you intend to use pack simulations:

```bash
python -m pip install -e ".[pack]" -c constraints-tested.txt
```

## AI providers and local models

The AI provider is optional and does not affect manual simulation. Open **AI Battery Expert → settings (gear icon)**, choose a provider, configure its endpoint/model or API key, test the connection, select a model, and apply the settings. There is no automatic provider fallback.

### Ollama (local)

1. Install Ollama from [ollama.com/download](https://ollama.com/download) and start its local service. The usual API endpoint is `http://127.0.0.1:11434`.
2. Download a model. For example:

   ```bash
   ollama pull qwen3:1.7b
   ```

   Other installed chat models can be used; available model names depend on your Ollama installation and hardware.
3. Start OpenBSA and open the AI settings dialog. Select **Ollama**, test/list models, select the model, then apply.

OpenBSA's current default Ollama model is `qwen3:1.7b`. If you use a different model, choose its exact installed tag in the settings. The app does not install or start Ollama for you. To use Ollama from a containerized deployment, the Ollama URL must point to a server reachable from the app container; `127.0.0.1` inside a container refers to that container itself.

### llama.cpp (local GGUF model)

llama.cpp runs a GGUF model behind an OpenAI-compatible HTTP server. Install/build [llama.cpp](https://github.com/ggml-org/llama.cpp), obtain a compatible GGUF model from its publisher, then start the server. A typical command is:

```bash
llama-server --model /path/to/model.gguf --host 127.0.0.1 --port 8080 --jinja
```

On Windows, quote paths containing spaces, for example:

```powershell
llama-server.exe --model "D:\Models\model.gguf" --host 127.0.0.1 --port 8080 --jinja
```

In AI settings choose **llama.cpp**, set the base URL to `http://127.0.0.1:8080/v1`, discover/select the model ID advertised by the server, and apply. If the server requires an API key, enter it in the provider's key field. Tool calling requires a model and chat template supported by the server; `--jinja` is commonly needed. A server can answer chat while still lacking reliable tool-call support.

The exact `llama-server` options and compatible model templates vary by llama.cpp build and model. Consult the [llama.cpp server documentation](https://github.com/ggml-org/llama.cpp/tree/master/tools/server) and model license before downloading/using a model. OpenBSA does not download GGUF files or launch llama.cpp.

### Cloud providers

OpenAI, Claude, and Gemini require an account/API key with access to a compatible model. Enter the key in the AI settings dialog, select a model, and explicitly enable the cloud sharing consent before using that provider. Keys entered in the dialog are held in server memory for the browser session; environment variables are also supported:

| Provider | Environment variable |
| --- | --- |
| OpenAI | `OPENAI_API_KEY` |
| Claude | `ANTHROPIC_API_KEY` |
| Gemini | `GEMINI_API_KEY` |

Cloud providers receive messages and enabled skill instructions. If context sharing is enabled, they can also receive the captured configuration and measured selected-result summaries. Review [AI provider data handling](docs/AI_PROVIDERS_AND_SKILLS.md) before enabling cloud use. Never put API keys in chat, source files, or skill Markdown.

### Provider summary

| Provider | Where inference runs | Default endpoint |
| --- | --- | --- |
| Ollama | Local Ollama service | `http://127.0.0.1:11434` |
| llama.cpp | Local llama.cpp server | `http://127.0.0.1:8080/v1` |
| OpenAI | OpenAI API | Official endpoint configured by the app |
| Claude | Anthropic API | Official endpoint configured by the app |
| Gemini | Google API | Official endpoint configured by the app |

## Using the application

1. Configure cell geometry and dimensions.
2. Select a model, materials, parameter values, and any supported degradation options.
3. Choose a protocol and configure its steps/limits.
4. Select **Run Simulation**. The button indicates queued/running and terminal states; double-click the running button to cancel the session-owned job.
5. Inspect results in the dashboard and plots. Export available data/figures from the results tools.
6. Optionally use the AI assistant. Choose its permissions deliberately; inspect/validate mode does not authorize runs.

The UI starts a fresh workspace after a browser refresh. Saved solution history and provider settings are retained according to the browser session; active workspace results and chat turns are not restored as a new workspace state. See [local operations](docs/LOCAL_OPERATIONS.md) for storage, jobs, retention, and backup guidance.

## Architecture

OpenBSA uses module folders as ownership boundaries. The metadata registry describes controls, model capabilities, compatibility, parameters, and result sources; it does not dynamically load arbitrary third-party plugins. The application service depends on interfaces, while a composition root wires concrete Dash, queue, persistence, and PyBaMM adapters.

```text
Dash callbacks ─┐
AI tool adapter ─┼─> composition root ─> application use cases ─> core ports
job worker ─────┘                                            └─> PyBaMM/queue adapters
```

Key directories:

| Path | Responsibility |
| --- | --- |
| `modules/core/` | Framework-independent interfaces and immutable feature descriptions |
| `modules/application/` | Simulation validation, submission, and execution use cases |
| `modules/geometry/` | Geometry feature declarations and calculations |
| `modules/models/` | Model capabilities and degradation feature declarations |
| `modules/parameters/` | Parameter contracts, material policies, defaults, and mappings |
| `modules/protocols/` | Protocol contracts, generation, and parsing |
| `modules/simulations/` | Simulation contracts, preflight, PyBaMM, pack, and degradation adapters |
| `modules/results/` | Canonical result contract, units, summaries, history, and exports |
| `modules/callbacks/` | Dash event wiring and UI job lifecycle |
| `modules/ai_chat/` | Provider adapters, permissions, chat lifecycle, and tools |
| `modules/visualizations/` | Plot and dashboard presentation |
| `modules/catalog.py`, `modules/composition.py` | Metadata assembly and infrastructure wiring |

For module contracts, dependency direction, extension guidance, and sources of truth, see [modules/README.md](modules/README.md). UI layout/theme and AI permission enforcement are kept outside the feature metadata registry.

## Data, privacy, and security

- The default server listens on localhost. The current supported deployment is a single trusted user on a local workstation, not an authenticated shared service.
- Simulation jobs, session data, history, and logs are written under the working directory, primarily `cache/` and `logs/`. Keep a stable `SECRET_KEY` if signed browser sessions must persist across restarts; secure the cache directory accordingly.
- Local provider requests go to the configured local model server. Cloud providers receive data as described above and in [AI providers and skills](docs/AI_PROVIDERS_AND_SKILLS.md).
- Provider secrets entered in the UI are held in server memory for that browser session and are not stored in browser storage or simulation configurations. Restarting the app clears UI-entered keys. Environment-provided keys remain available to the process.
- AI skills are bounded instruction files, not executable plugins. They cannot run shell/Python code or grant tool permissions.
- Do not expose the server to an untrusted network. Authentication and multi-user shared-server secret management are not implemented.

## Configuration

The app uses defaults suitable for localhost development. Configuration is read from environment variables by the app/configuration and job modules. Provider profiles can be configured in the AI settings dialog. Common environment settings include:

| Variable | Purpose |
| --- | --- |
| `HOST`, `PORT` | Listener defaults; CLI `--host`/`--port` are explicit alternatives |
| `SECRET_KEY` | Stable signed-session secret; set a private, random value for persistent sessions |
| `OPENBSA_MAX_JOBS` | Concurrent simulation worker limit (default 2) |
| `OPENBSA_MAX_QUEUED_JOBS` | Maximum queued jobs (default 10) |
| `OPENBSA_JOB_TIMEOUT` | Job timeout in seconds (default 300) |
| `OPENBSA_JOB_RETENTION_DAYS` | Optional terminal job retention; unset/zero disables expiry |
| `OLLAMA_HOST`, `OLLAMA_PORT`, `OLLAMA_MODEL`, `OLLAMA_TIMEOUT` | Ollama endpoint/model defaults |
| `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY` | Optional cloud provider credentials |
| `LLAMA_CPP_API_KEY`, `OLLAMA_API_KEY` | Optional local server credentials |

Set environment variables in the shell or service environment before starting OpenBSA; the app does not automatically load a `.env` file. `.env.example` is a reference template and contains some legacy development settings. `OLLAMA_MODEL` is read by the production configuration; the development configuration currently defaults to `qwen3:1.7b`, and provider profiles can be selected directly in AI settings. For the full job/session/storage behavior, see [local operations](docs/LOCAL_OPERATIONS.md).

## Development and tests

Install the development extra as shown above. Run focused or full tests from the repository root:

```bash
python -m pytest --tb=short -p no:cacheprovider
python -m pytest tests/test_architecture.py tests/test_original_ui_contract.py tests/test_ai_job_lifecycle.py
```

Some solver and integration checks run real PyBaMM simulations and can take longer than UI/unit tests. Test configuration and active regression coverage are in `tests/` and [local operations](docs/LOCAL_OPERATIONS.md). The tested dependency constraints are in `constraints-tested.txt`.

When adding a feature, declare its metadata in the owning module, implement it behind the appropriate domain/engine interface, and update only the relevant projections and explicit AI permissions. New physics or chemistry also needs independent scientific verification; registration alone is not validation. See [modules/README.md](modules/README.md#extending-a-feature).

## Documentation

- [Getting started](docs/getting_started.md)
- [Module architecture and extension guide](modules/README.md)
- [AI providers, permissions, and skills](docs/AI_PROVIDERS_AND_SKILLS.md)
- [Local operations, jobs, storage, and recovery](docs/LOCAL_OPERATIONS.md)
- [Scientific scope and verification status](docs/SCIENTIFIC_SCOPE.md)
- [Coin half-cell model and limitations](docs/COIN_HALF_CELL.md)
- [Degradation assumptions](docs/MODULAR_DEGRADATION.md)
- [PyBaMM parameter map](docs/P2D_PARAMETER_MAP.md)
- [UI performance notes](docs/UI_PERFORMANCE.md)
- [Changelog](CHANGELOG.md)
- [Third-party notices](THIRD_PARTY_NOTICES.md)

## Troubleshooting

**The web page does not open:** Confirm the server process is still running and visit the exact host/port printed at startup. If port 8050 is occupied, pass another port with `--port`.

**Python or PyBaMM installation fails:** Confirm Python is between 3.10 and 3.12, that the virtual environment is active, and install with the tested constraints file. For a PyBaMM solver/backend error, consult the linked PyBaMM installation guide and the error output.

**AI cannot connect:** Start the selected local provider first; verify its URL and model in AI settings; then use **Test connection / list models**. Local AI model servers are independent processes and are not launched by OpenBSA.

**AI responds but cannot run a simulation:** Open its settings and choose the appropriate tool mode. Chat-only and inspect/validate modes do not permit submission. Check the action trace, validation result, and job status; a provider/model may also support text chat without compatible tool calling.

**A result or history entry is missing:** Check the in-app job status and logs. A solver result and a saved history artifact are separate operations; a history-write warning can occur after a solve. Do not remove or edit a live SQLite database. See [local operations](docs/LOCAL_OPERATIONS.md).

## License

OpenBSA is licensed under the [MIT](LICENSE). Third-party code and scientific workflow attributions are listed in [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).
