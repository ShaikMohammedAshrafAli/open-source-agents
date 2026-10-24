# Git PR Review Agent

> An autonomous AI agent for security audit, code quality review, and automated Conventional Pull Request description generation from Git diffs.

---

## Overview

### Problem Statement & Intended Users
- **Problem**: Code reviews are frequently slowed down by manual verification of basic hygiene (leaked credentials, hardcoded keys, missing tests, unpinned dependencies) and writing detailed PR descriptions. Developers often submit PRs with empty or vague descriptions, making peer review tedious and increasing the risk of accidental secret leaks reaching production.
- **Intended Users**: Software engineers, open-source contributors, devops engineers, and tech leads who want instant, pre-commit or pre-merge feedback and automated PR documentation directly in their terminal or CI/CD pipelines.

### Key Features
- **Deterministic Secret & Vulnerability Scan**: Instant pre-LLM detection of leaked API keys (Google AI, GitHub PATs, OpenAI, AWS, Stripe), committed `.env` files, and dangerous patterns (`eval()`, `exec()`, raw SQL string formatting).
- **Multi-Model Flexibility**: Built for Google Gemini (`gemini-2.5-flash`, `gemini-1.5-flash`), local offline Google Gemma 2 (`gemma2:2b` via Ollama), and a zero-credential Mock engine for instant CI testing.
- **Automated Conventional PR Generation**: Synthesizes git diffs into structured Markdown PR descriptions with standardized titles (`feat(...)`, `fix(...)`), change summaries, architectural notes, and verification checklists.
- **Dual Interface (CLI & REST API)**: Execute as a fast terminal tool (`python main.py`) or run as a standalone FastAPI service (`python main.py --serve`) with interactive Swagger documentation.
- **Strict Risk Scoring & Verdicts**: Assigns a risk score (0-100) and authoritative review verdicts (`APPROVE`, `REQUEST_CHANGES`, `COMMENT`).

---

## Technical Stack

- **Language & Runtime**: Python 3.9+ / 3.12+
- **Agent Framework / Core Library**: Custom multi-stage agent pipeline (`git_utils`, `security_scanner`, `llm_client`, `agent`)
- **API Framework**: FastAPI & Uvicorn (interactive REST API and Swagger UI at `/docs`)
- **Models**:
  - Google Gemini 2.5 Flash / Gemini 1.5 Flash (via `google-genai` / `google-generativeai`)
  - Google Gemma 2 (`gemma2:2b` local offline inference via Ollama)
  - Built-in Mock Agent Engine (deterministic, zero-network test suite)
- **Validation & Tooling**: Pydantic v2, Pytest, python-dotenv

---

## Architecture & Data Flow

```
[Git Working Tree / Diff File]
              │
              ▼
    [Git Chunker & Filter] ─── (strips lockfiles, minified assets, binaries)
              │
              ├──► [Deterministic Security Scanner] ──► [Critical Leaks / Anti-Patterns]
              │                                                     │
              ▼                                                     ▼
     [LLM Review Agent] ───────────────────────────────► [Issue Aggregator & Deduplicator]
    (Gemini / Gemma 2 / Mock)                                       │
              │                                                     ▼
              ▼                                             [Risk Scorer & Verdict]
    [PR Summary Synthesizer]                                (APPROVE / REQUEST_CHANGES)
              │                                                     │
              └─────────────────────┬───────────────────────────────┘
                                    │
                                    ▼
                     [Structured Review Report]
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
       [Markdown PR Description]          [REST API / JSON Payload]
        (pr_description.md / CLI)          (/api/review & /api/health)
```

---

## Prerequisites

Before running this project, ensure you have:
- Python 3.9 or higher (tested on Python 3.9 and 3.12)
- Git 2.30+ installed
- *(Optional for cloud inference)*: A Google Gemini API key from [Google AI Studio](https://aistudio.google.com/)
- *(Optional for offline local inference)*: [Ollama](https://ollama.com/) with `ollama run gemma2:2b`

---

## Installation & Setup

1. **Navigate to the agent directory:**
   ```bash
   cd agents/git-pr-review-agent
   ```

2. **Set up a virtual environment and install dependencies:**
   ```bash
   python -m venv .venv

   # On Linux/macOS:
   source .venv/bin/activate

   # On Windows:
   .venv\Scripts\activate

   pip install -r requirements.txt
   ```

3. **Configure Environment Variables (Optional):**
   Copy the example environment file:
   ```bash
   cp .env.example .env
   ```
   Add your Gemini API key if using cloud inference, or leave blank to automatically use local Ollama (`gemma2:2b`) or Mock mode:
   ```bash
   GEMINI_API_KEY=AIzaSy...
   ```

---

## Configuration & Environment Variables

| Variable Name | Required | Default | Description |
| :--- | :--- | :--- | :--- |
| `GEMINI_API_KEY` | No | *None* | Google Gemini API key for cloud model reasoning |
| `LLM_PROVIDER` | No | `auto` | Preferred provider: `gemini`, `ollama`, or `mock` |
| `MODEL_NAME` | No | `gemini-2.5-flash` | Model identifier (`gemini-2.5-flash`, `gemma2:2b`, etc.) |
| `OLLAMA_BASE_URL` | No | `http://localhost:11434` | Ollama HTTP endpoint for local offline inference |
| `PORT` | No | `8000` | Port for the FastAPI server when running `--serve` |
| `HOST` | No | `0.0.0.0` | Host bind address for the FastAPI server |

---

## How to Run

### Option 1: Review Local Git Changes (CLI)

```bash
# Review current unstaged and staged changes in your git repository:
python main.py

# Review against a specific target branch (e.g. main):
python main.py --branch main

# Review only staged changes:
python main.py --staged

# Save generated PR description directly to a markdown file:
python main.py --output pr_description.md

# Output machine-readable JSON for CI/CD pipelines:
python main.py --json
```

### Option 2: Review from a Patch File or Stdin

```bash
# Review a pre-existing diff/patch file:
python main.py --diff-file sample.patch

# Pipe directly from git:
git diff main | python main.py
```

### Option 3: Offline Local Execution with Google Gemma 2

```bash
# Force using local Gemma 2 via Ollama (100% private, no internet needed):
python main.py --provider ollama --model gemma2:2b
```

### Option 4: Zero-Credential Mock Mode (for CI/CD testing)

```bash
python main.py --provider mock
```

### Option 5: Run as a REST API Server

```bash
python main.py --serve --port 8000
```
Open [http://localhost:8000/docs](http://localhost:8000/docs) in your browser to access the interactive Swagger UI.

---

## Example Input & Output

### Example Input Diff
```diff
diff --git a/auth/token_manager.py b/auth/token_manager.py
new file mode 100644
--- /dev/null
+++ b/auth/token_manager.py
@@ -0,0 +1,5 @@
+import os
+
+GEMINI_KEY = "AIzaSyB1234567890abcdef1234567890abcdef"
+def verify_token(raw_input):
+    return eval(raw_input)
```

### Example Terminal / Markdown Output
```markdown
# feat(auth): add token manager implementation

> 🤖 *Automated review and PR description generated by **Git PR Review Agent**.*

## Summary
Added initial token manager module under auth/token_manager.py with key definitions.

## Review Overview
- **Verdict**: 🔴 **CHANGES REQUESTED**
- **Risk Score**: `100/100`
- **Model Used**: `Gemini (gemini-2.5-flash)`
- **Changes**: `1 files` (+5 / -0)

## 🔍 Code Review Findings
### 🚨 [CRITICAL] Google API / Gemini Secret Key
**File**: `auth/token_manager.py:L3` | **Category**: `security`
Potential secret detected on line 3 of auth/token_manager.py.
> **Suggestion**: Remove this API key immediately! Use environment variables or a secret manager.

### 🚨 [CRITICAL] Arbitrary Code Execution via eval()
**File**: `auth/token_manager.py:L5` | **Category**: `security`
Dangerous security anti-pattern found: return eval(raw_input)
> **Suggestion**: eval() allows arbitrary code execution. Refactor to use safe parsing.

## 📋 Verification Checklist
- [ ] Remove hardcoded credentials from source code
- [ ] Refactor eval() to safe ast.literal_eval or json parsing
- [ ] Run automated test suite: pytest
```

---

## How to Test

Run the comprehensive unit test suite:

```bash
pytest -v
```

The test suite validates:
1. Static security scanner against leaked API keys, tokens, and code smells.
2. Safe diff chunking and lockfile filtering.
3. Multi-provider fallbacks and mock generation.
4. Risk score escalation on critical findings.
5. Markdown PR generation formatting.
6. FastAPI health and review REST endpoints.

---

## Known Limitations

- **Binary & Large Asset Diffs**: Non-text assets (images, compiled binaries) are filtered out to conserve token limits and avoid parsing errors.
- **Very Large Diffs**: Single diffs exceeding 35,000 characters are safely truncated to fit within model context windows without crashing.
- **Local Ollama Speed**: When running offline with local `gemma2:2b`, generation latency depends on local CPU/GPU hardware.

---

## Security & Privacy Considerations

- **Zero Secrets Tracked**: `.env` and secret patterns are in `.gitignore`. The static scanner actively blocks committed `.env` files.
- **Air-Gapped Operation**: With `--provider ollama` or `--provider mock`, zero code diffs or telemetry are transmitted outside the local machine.
- **Safe Command Execution**: Git commands are run with explicit argument arrays without `shell=True` to prevent command injection.

---

## Future Improvements

- [ ] GitHub Actions workflow integration for automated PR review comments.
- [ ] Multi-turn interactive discussion with the agent on specific review findings.
- [ ] Support for AST-based Python semantic diffing.

---

## License

*This project is contributed under the repository's open-source license.*
