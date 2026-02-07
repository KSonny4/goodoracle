# GoodOracle MVP Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a lightweight autonomous agent framework (~1,500 LOC) that wraps Claude Code CLI to manage projects, evaluate strategies, maintain persistent memory, and communicate via Telegram — domain-agnostic with trading as first use case.

**Architecture:** Single-process Python agent with a mailbox pattern. Tasks arrive via Telegram messages, cron triggers, or manual CLI invocation. Each task is queued in SQLite, processed by invoking `claude --print --model claude-opus-4-6` with assembled context (identity + memory + task), and results flow back via Telegram and memory updates. Profiles make it domain-agnostic — swap a profile, swap the agent's behavior.

**Tech Stack:** Python 3.12+, Claude Code CLI (Opus 4.6 enforced), SQLite, python-telegram-bot, pytest, ruff

**Inspirations:**
- **nanobot** (HKUDS): message bus, dual-layer memory, skill system, ~4K LOC
- **nanoclaw** (gavrielc): radical simplicity, single-process, ~1.5K LOC
- **Key differentiator:** We use local Claude Code CLI (full tool ecosystem) instead of API calls

---

## Project Structure

```
goodoracle/
├── CLAUDE.md                          # Project manifest
├── Makefile                           # Easy commands: make test, make playground, make smoke
├── pyproject.toml                     # Python project config
├── .env.example                       # Required env vars
├── .gitignore
├── goodoracle/
│   ├── __init__.py
│   ├── __main__.py                    # Entry: python -m goodoracle
│   ├── cli.py                         # CLI commands (run, evaluate, task)
│   ├── config.py                      # Configuration loading
│   ├── claude.py                      # Claude Code CLI wrapper (ENFORCES OPUS 4.6)
│   ├── memory.py                      # Memory management
│   ├── context.py                     # Prompt/context builder
│   ├── mailbox.py                     # SQLite task queue
│   ├── agent.py                       # Core agent loop
│   ├── telegram_bot.py                # Telegram integration
│   └── tools/
│       ├── __init__.py                # Tool registry
│       ├── base.py                    # Tool interface
│       ├── shell.py                   # Shell command execution
│       └── grafana.py                 # Grafana query tool (trading profile)
├── profiles/
│   ├── playground/                    # Safe test profile — real Claude, no external deps
│   │   ├── identity.md                # Simple verifiable test agent
│   │   ├── tools.md                   # Filesystem only
│   │   └── evaluation.md              # Self-check evaluation template
│   └── trading/
│       ├── identity.md                # Agent identity for trading
│       ├── tools.md                   # Available tools description
│       └── evaluation.md              # Evaluation cycle template
├── memory/
│   ├── MEMORY.md                      # Long-term knowledge
│   └── daily/                         # Daily observations (YYYY-MM-DD.md)
├── tests/
│   ├── conftest.py
│   ├── test_config.py
│   ├── test_claude.py
│   ├── test_memory.py
│   ├── test_context.py
│   ├── test_mailbox.py
│   ├── test_agent.py
│   ├── test_tools.py
│   ├── test_telegram.py
│   └── test_cli.py
├── scripts/
│   ├── smoke-test.sh                  # Full E2E verification with REAL Claude CLI
│   ├── install-schedule.sh            # Install cron/launchd
│   └── com.goodoracle.agent.plist     # macOS launchd config
└── docs/
    ├── archived-skills/               # Backed-up old trading skills
    └── plans/
        └── 2026-02-07-goodoracle-mvp.md  # This file
```

**LOC target:** ~1,500 lines core framework. If you can't understand the entire codebase in 10 minutes, it's too complex.

---

## Phase 1: Project Foundation

### Task 1: Create Project Scaffold

**Files:**
- Create: `pyproject.toml`
- Create: `.gitignore`
- Create: `.env.example`
- Create: `goodoracle/__init__.py`
- Create: `tests/conftest.py`

**Step 1: Create directory structure**

```bash
cd /Users/petr.kubelka/git_projects/goodoracle
mkdir -p goodoracle/tools profiles/trading profiles/playground memory/daily tests scripts docs/plans data logs
```

**Step 2: Write pyproject.toml**

```toml
[project]
name = "goodoracle"
version = "0.1.0"
description = "Lightweight autonomous agent framework wrapping Claude Code CLI"
requires-python = ">=3.12"
dependencies = [
    "python-telegram-bot>=21.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "ruff>=0.4",
]

[project.scripts]
goodoracle = "goodoracle.cli:main"

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
```

**Step 3: Write .gitignore**

```
__pycache__/
*.pyc
.env
*.egg-info/
dist/
.venv/
data/*.db
logs/
```

**Step 4: Write .env.example**

```bash
# Telegram
TELEGRAM_BOT_TOKEN=your-bot-token
TELEGRAM_CHAT_ID=your-chat-id

# Grafana (optional, for trading profile)
GRAFANA_CLOUD_PROM_URL=
GRAFANA_CLOUD_LOKI_URL=
GRAFANA_CLOUD_API_KEY=
```

**Step 5: Write goodoracle/__init__.py**

```python
"""GoodOracle — lightweight autonomous agent wrapping Claude Code CLI."""

__version__ = "0.1.0"
```

**Step 6: Write tests/conftest.py**

```python
"""Shared test fixtures."""

from pathlib import Path
import pytest


@pytest.fixture
def project_root(tmp_path: Path) -> Path:
    """Create a minimal project structure for testing."""
    (tmp_path / "profiles" / "test").mkdir(parents=True)
    (tmp_path / "profiles" / "test" / "identity.md").write_text("You are a test agent.")
    (tmp_path / "memory").mkdir()
    (tmp_path / "memory" / "daily").mkdir()
    (tmp_path / "data").mkdir()
    return tmp_path
```

**Step 7: Write Makefile**

```makefile
.PHONY: install test lint playground smoke evaluate clean

install:
	python -m venv .venv
	.venv/bin/pip install -e ".[dev]"

test:
	.venv/bin/pytest -v

lint:
	.venv/bin/ruff check .

# Run a single task with the playground profile — REAL Claude, no mocks
playground:
	.venv/bin/goodoracle task --profile playground "Read the file CLAUDE.md in this project directory. Count the number of second-level headings (##). Respond with: TELEGRAM: Found N headings in CLAUDE.md. MEMORY: CLAUDE.md has N second-level headings."

# Full end-to-end smoke test — verifies entire pipeline with real Claude CLI
smoke:
	./scripts/smoke-test.sh

# Run trading evaluation cycle
evaluate:
	.venv/bin/goodoracle evaluate --profile trading

# Show task history from the mailbox
history:
	sqlite3 data/goodoracle.db "SELECT id, status, source, substr(content, 1, 60) as task, substr(result, 1, 60) as result, created_at FROM tasks ORDER BY id DESC LIMIT 10;"

# Show current memory state
memory:
	@echo "=== Long-Term Memory ==="
	@cat memory/MEMORY.md 2>/dev/null || echo "(empty)"
	@echo ""
	@echo "=== Today ==="
	@cat memory/daily/$$(date +%Y-%m-%d).md 2>/dev/null || echo "(no entries today)"

# Reset all state (memory + db) for fresh testing
clean:
	rm -f data/goodoracle.db
	rm -f memory/MEMORY.md
	rm -f memory/daily/*.md
	@echo "State cleaned. Ready for fresh run."
```

**Step 8: Create placeholders**

```bash
touch memory/daily/.gitkeep
touch memory/MEMORY.md
```

**Step 9: Install and verify**

Run: `cd /Users/petr.kubelka/git_projects/goodoracle && make install`
Expected: Successfully installed goodoracle-0.1.0

**Step 10: Commit**

```bash
git add -A
git commit -m "feat: project scaffold with pyproject.toml and directory structure"
```

---

### Task 2: Write CLAUDE.md

**Files:**
- Create: `CLAUDE.md`

**Step 1: Write CLAUDE.md**

```markdown
# GoodOracle

## What Is This?

Lightweight autonomous agent framework (~1,500 LOC) wrapping Claude Code CLI.
Runs in a loop, processes tasks from a mailbox, maintains persistent memory,
communicates via Telegram. Domain-agnostic — trading is the first profile.

## Non-Negotiable Rules

1. **Always Opus 4.6** — every Claude CLI call MUST use `--model claude-opus-4-6`. No exceptions. No overrides.
2. **Small codebase** — target ~1,500 LOC. If you can't understand it in 10 minutes, it's too complex.
3. **Domain agnostic** — no trading logic in core. Profiles add domain behavior.
4. **File-based memory** — human-readable Markdown, git-trackable.
5. **Single process** — no microservices, no message queues, no abstraction layers.

## Architecture

```
Telegram / Cron / CLI
        |
    Mailbox (SQLite queue)
        |
    Agent Loop
        |
    Context Builder (identity + memory + task)
        |
    Claude Code CLI (--print --model claude-opus-4-6)
        |
    Result Parser (MEMORY: and TELEGRAM: directives)
        |
    Memory Update + Telegram Response
```

## Running

```bash
# Interactive: poll Telegram for messages, process them
goodoracle run --profile trading

# One-shot evaluation (for cron, every 6h)
goodoracle evaluate --profile trading

# Single task
goodoracle task --profile trading "Analyze BTC market conditions"
```

## Development

```bash
# Install
pip install -e ".[dev]"

# Test
pytest -v

# Lint
ruff check .
```

## Key Design Decisions

### Why Claude Code CLI, not API?
Claude Code CLI gives access to the full tool ecosystem (Bash, Read, Write, Edit,
Grep, Glob). The agent can read files, run commands, edit code — all through a
single CLI call. No need to reimplement tools.

### Why mailbox pattern?
Everything is a task: Telegram messages, cron triggers, manual CLI invocations.
One queue, one processing loop. Simple to debug, simple to extend.

### Why profiles?
The same agent framework can be a trading analyst, a code reviewer, a research
assistant. Profile = identity + tools + memory context. Swap profiles, swap behavior.

### Why file-based memory?
- Human-readable (you can open MEMORY.md and read it)
- Git-trackable (see what the agent learned over time)
- Editable (fix mistakes by editing Markdown)
- Simple (no vector DB, no embeddings, no complexity)

## Response Directives

Claude's response is parsed for directives:
- Lines starting with `MEMORY:` → appended to long-term memory
- Lines starting with `TELEGRAM:` → sent to Telegram
- Everything else → stored as task result

## Profile Structure

```
profiles/<name>/
├── identity.md      # Who the agent is
├── tools.md         # What tools are available
└── evaluation.md    # Template for scheduled evaluation cycles
```

## Superpowers Skills Workflow

For development on this project:
- Before coding: `superpowers:writing-plans` + `superpowers:executing-plans`
- For features: `superpowers:test-driven-development` (TDD always)
- For debugging: `superpowers:systematic-debugging`
- After implementation: `superpowers:verification-before-completion`
```

**Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "docs: add CLAUDE.md project manifest"
```

---

### Task 3: Configuration System

**Files:**
- Create: `goodoracle/config.py`
- Create: `tests/test_config.py`

**Step 1: Write the failing test**

```python
# tests/test_config.py
import os
from pathlib import Path

from goodoracle.config import Config


def test_load_config_defaults():
    config = Config(profile="trading", project_root=Path("/tmp/test-goodoracle"))
    assert config.profile == "trading"
    assert config.model == "claude-opus-4-6"


def test_load_config_from_env(monkeypatch):
    monkeypatch.setenv("TELEGRAM_BOT_TOKEN", "test-token")
    monkeypatch.setenv("TELEGRAM_CHAT_ID", "12345")
    config = Config(profile="trading", project_root=Path("/tmp/test-goodoracle"))
    assert config.telegram_bot_token == "test-token"
    assert config.telegram_chat_id == "12345"


def test_profile_dir(tmp_path):
    profiles_dir = tmp_path / "profiles" / "trading"
    profiles_dir.mkdir(parents=True)
    config = Config(profile="trading", project_root=tmp_path)
    assert config.profile_dir == profiles_dir


def test_model_always_opus():
    """Model must ALWAYS be claude-opus-4-6, no override possible."""
    config = Config(profile="trading", project_root=Path("/tmp/test"))
    assert config.model == "claude-opus-4-6"
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_config.py -v`
Expected: FAIL with "ModuleNotFoundError"

**Step 3: Write minimal implementation**

```python
# goodoracle/config.py
"""Configuration — loads profile and environment variables."""

import os
from dataclasses import dataclass, field
from pathlib import Path


@dataclass
class Config:
    profile: str
    project_root: Path = field(default_factory=lambda: Path(__file__).parent.parent)
    model: str = field(default="claude-opus-4-6", init=False)

    @property
    def profile_dir(self) -> Path:
        return self.project_root / "profiles" / self.profile

    @property
    def memory_dir(self) -> Path:
        return self.project_root / "memory"

    @property
    def db_path(self) -> Path:
        return self.project_root / "data" / "goodoracle.db"

    @property
    def telegram_bot_token(self) -> str:
        return os.environ.get("TELEGRAM_BOT_TOKEN", "")

    @property
    def telegram_chat_id(self) -> str:
        return os.environ.get("TELEGRAM_CHAT_ID", "")
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_config.py -v`
Expected: PASS (4 passed)

**Step 5: Commit**

```bash
git add goodoracle/config.py tests/test_config.py
git commit -m "feat: configuration system with profile support"
```

---

### Task 4: Claude Code CLI Wrapper

**Files:**
- Create: `goodoracle/claude.py`
- Create: `tests/test_claude.py`

**Step 1: Write the failing test**

```python
# tests/test_claude.py
import subprocess
from unittest.mock import patch, MagicMock

from goodoracle.claude import invoke_claude, ClaudeError


def test_invoke_claude_basic():
    """Test basic Claude CLI invocation with mocked subprocess."""
    mock_result = MagicMock()
    mock_result.returncode = 0
    mock_result.stdout = "Hello, world!"
    mock_result.stderr = ""

    with patch("subprocess.run", return_value=mock_result) as mock_run:
        result = invoke_claude("Say hello")
        assert result == "Hello, world!"

        # Verify opus-4-6 is ALWAYS enforced
        cmd = mock_run.call_args[0][0]
        assert "--model" in cmd
        assert "claude-opus-4-6" in cmd


def test_invoke_claude_with_context():
    """Test that context is piped as stdin."""
    mock_result = MagicMock()
    mock_result.returncode = 0
    mock_result.stdout = "Analysis complete"
    mock_result.stderr = ""

    with patch("subprocess.run", return_value=mock_result) as mock_run:
        invoke_claude("Analyze this", context="some data")
        assert mock_run.call_args[1]["input"] == "some data"


def test_invoke_claude_with_system_prompt():
    """Test system prompt is passed correctly."""
    mock_result = MagicMock()
    mock_result.returncode = 0
    mock_result.stdout = "Done"
    mock_result.stderr = ""

    with patch("subprocess.run", return_value=mock_result) as mock_run:
        invoke_claude("Do something", system_prompt="You are a trader")
        cmd = mock_run.call_args[0][0]
        assert "--system-prompt" in cmd


def test_invoke_claude_enforces_opus():
    """Model flag must ALWAYS be claude-opus-4-6. This is non-negotiable."""
    mock_result = MagicMock()
    mock_result.returncode = 0
    mock_result.stdout = "ok"
    mock_result.stderr = ""

    with patch("subprocess.run", return_value=mock_result) as mock_run:
        invoke_claude("test")
        cmd = mock_run.call_args[0][0]
        model_idx = cmd.index("--model")
        assert cmd[model_idx + 1] == "claude-opus-4-6"


def test_invoke_claude_timeout():
    """Test timeout handling."""
    with patch("subprocess.run", side_effect=subprocess.TimeoutExpired("claude", 300)):
        try:
            invoke_claude("slow task", timeout=300)
            assert False, "Should have raised"
        except ClaudeError as e:
            assert "timeout" in str(e).lower()


def test_invoke_claude_nonzero_exit():
    """Test non-zero exit code handling."""
    mock_result = MagicMock()
    mock_result.returncode = 1
    mock_result.stdout = ""
    mock_result.stderr = "Error: something went wrong"

    with patch("subprocess.run", return_value=mock_result):
        try:
            invoke_claude("bad prompt")
            assert False, "Should have raised"
        except ClaudeError as e:
            assert "something went wrong" in str(e)
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_claude.py -v`
Expected: FAIL with "ModuleNotFoundError"

**Step 3: Write minimal implementation**

```python
# goodoracle/claude.py
"""Claude Code CLI wrapper — ALWAYS enforces Opus 4.6."""

import subprocess


class ClaudeError(Exception):
    """Raised when Claude CLI invocation fails."""


MODEL = "claude-opus-4-6"
DEFAULT_TIMEOUT = 600  # 10 minutes


def invoke_claude(
    prompt: str,
    *,
    context: str | None = None,
    system_prompt: str | None = None,
    timeout: int = DEFAULT_TIMEOUT,
    output_format: str | None = None,
) -> str:
    """Invoke Claude Code CLI and return the output.

    IMPORTANT: Model is ALWAYS claude-opus-4-6. No override possible.

    Args:
        prompt: The task/question for Claude.
        context: Optional text piped as stdin.
        system_prompt: Optional system prompt.
        timeout: Timeout in seconds (default 600).
        output_format: Optional output format ("json").

    Returns:
        Claude's text response.

    Raises:
        ClaudeError: On timeout, non-zero exit, or other failures.
    """
    cmd = ["claude", "--print", "--model", MODEL]

    if system_prompt:
        cmd.extend(["--system-prompt", system_prompt])

    if output_format:
        cmd.extend(["--output-format", output_format])

    cmd.append(prompt)

    try:
        result = subprocess.run(
            cmd,
            input=context,
            capture_output=True,
            text=True,
            timeout=timeout,
        )
    except subprocess.TimeoutExpired:
        raise ClaudeError(f"Timeout after {timeout}s")

    if result.returncode != 0:
        raise ClaudeError(result.stderr.strip() or f"Exit code {result.returncode}")

    return result.stdout.strip()
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_claude.py -v`
Expected: PASS (6 passed)

**Step 5: Commit**

```bash
git add goodoracle/claude.py tests/test_claude.py
git commit -m "feat: Claude Code CLI wrapper enforcing Opus 4.6"
```

---

## Phase 2: Memory & Context

### Task 5: Memory System

**Files:**
- Create: `goodoracle/memory.py`
- Create: `tests/test_memory.py`

**Step 1: Write the failing test**

```python
# tests/test_memory.py
from datetime import date
from pathlib import Path

from goodoracle.memory import Memory


def test_read_long_term_empty(tmp_path):
    mem = Memory(memory_dir=tmp_path)
    assert mem.read_long_term() == ""


def test_write_and_read_long_term(tmp_path):
    mem = Memory(memory_dir=tmp_path)
    mem.write_long_term("I learned that BTC markets are efficient.")
    assert "BTC markets are efficient" in mem.read_long_term()


def test_append_long_term(tmp_path):
    mem = Memory(memory_dir=tmp_path)
    mem.write_long_term("Fact 1.")
    mem.append_long_term("Fact 2.")
    content = mem.read_long_term()
    assert "Fact 1." in content
    assert "Fact 2." in content


def test_read_today_empty(tmp_path):
    mem = Memory(memory_dir=tmp_path)
    assert mem.read_today() == ""


def test_append_today(tmp_path):
    mem = Memory(memory_dir=tmp_path)
    mem.append_today("Observation: spread is $1.01")
    content = mem.read_today()
    assert "spread is $1.01" in content


def test_append_today_creates_file_with_header(tmp_path):
    mem = Memory(memory_dir=tmp_path)
    mem.append_today("First entry")
    content = mem.read_today()
    today = date.today().isoformat()
    assert today in content


def test_read_recent_days(tmp_path):
    mem = Memory(memory_dir=tmp_path)
    daily_dir = tmp_path / "daily"
    daily_dir.mkdir(exist_ok=True)
    (daily_dir / "2026-02-06.md").write_text("# 2026-02-06\n\nYesterday note")
    mem.append_today("Today note")
    recent = mem.read_recent(days=7)
    assert "Today note" in recent
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_memory.py -v`
Expected: FAIL with "ModuleNotFoundError"

**Step 3: Write minimal implementation**

```python
# goodoracle/memory.py
"""Memory management — daily observations + long-term knowledge."""

from datetime import date, timedelta
from pathlib import Path


class Memory:
    def __init__(self, memory_dir: Path):
        self.memory_dir = memory_dir
        self.memory_dir.mkdir(parents=True, exist_ok=True)
        self.daily_dir = memory_dir / "daily"
        self.daily_dir.mkdir(exist_ok=True)

    @property
    def _long_term_path(self) -> Path:
        return self.memory_dir / "MEMORY.md"

    def _daily_path(self, d: date | None = None) -> Path:
        d = d or date.today()
        return self.daily_dir / f"{d.isoformat()}.md"

    def read_long_term(self) -> str:
        path = self._long_term_path
        return path.read_text() if path.exists() else ""

    def write_long_term(self, content: str) -> None:
        self._long_term_path.write_text(content)

    def append_long_term(self, content: str) -> None:
        existing = self.read_long_term()
        separator = "\n\n" if existing else ""
        self._long_term_path.write_text(existing + separator + content)

    def read_today(self) -> str:
        path = self._daily_path()
        return path.read_text() if path.exists() else ""

    def append_today(self, content: str) -> None:
        path = self._daily_path()
        if not path.exists():
            header = f"# {date.today().isoformat()}\n\n"
            path.write_text(header + content + "\n")
        else:
            existing = path.read_text()
            path.write_text(existing + "\n" + content + "\n")

    def read_recent(self, days: int = 7) -> str:
        parts = []
        today = date.today()
        for i in range(days):
            d = today - timedelta(days=i)
            path = self._daily_path(d)
            if path.exists():
                parts.append(path.read_text())
        return "\n\n---\n\n".join(parts)
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_memory.py -v`
Expected: PASS (7 passed)

**Step 5: Commit**

```bash
git add goodoracle/memory.py tests/test_memory.py
git commit -m "feat: memory system with daily + long-term storage"
```

---

### Task 6: Context Builder

**Files:**
- Create: `goodoracle/context.py`
- Create: `tests/test_context.py`

**Step 1: Write the failing test**

```python
# tests/test_context.py
from pathlib import Path

from goodoracle.context import ContextBuilder
from goodoracle.config import Config
from goodoracle.memory import Memory


def test_build_context_includes_identity(tmp_path):
    profile_dir = tmp_path / "profiles" / "test"
    profile_dir.mkdir(parents=True)
    (profile_dir / "identity.md").write_text("You are a helpful analyst.")

    config = Config(profile="test", project_root=tmp_path)
    memory = Memory(memory_dir=tmp_path / "memory")
    ctx = ContextBuilder(config=config, memory=memory)
    system_prompt = ctx.build_system_prompt()
    assert "helpful analyst" in system_prompt


def test_build_context_includes_memory(tmp_path):
    profile_dir = tmp_path / "profiles" / "test"
    profile_dir.mkdir(parents=True)
    (profile_dir / "identity.md").write_text("You are an agent.")

    memory = Memory(memory_dir=tmp_path / "memory")
    memory.write_long_term("Key fact: markets close at 4pm.")
    memory.append_today("Observation: volatility is high.")

    config = Config(profile="test", project_root=tmp_path)
    ctx = ContextBuilder(config=config, memory=memory)
    system_prompt = ctx.build_system_prompt()
    assert "markets close at 4pm" in system_prompt
    assert "volatility is high" in system_prompt


def test_build_context_includes_tools(tmp_path):
    profile_dir = tmp_path / "profiles" / "test"
    profile_dir.mkdir(parents=True)
    (profile_dir / "identity.md").write_text("Agent")
    (profile_dir / "tools.md").write_text("You can query Grafana.")

    config = Config(profile="test", project_root=tmp_path)
    memory = Memory(memory_dir=tmp_path / "memory")
    ctx = ContextBuilder(config=config, memory=memory)
    system_prompt = ctx.build_system_prompt()
    assert "query Grafana" in system_prompt


def test_build_task_prompt(tmp_path):
    profile_dir = tmp_path / "profiles" / "test"
    profile_dir.mkdir(parents=True)
    (profile_dir / "identity.md").write_text("Agent")

    config = Config(profile="test", project_root=tmp_path)
    memory = Memory(memory_dir=tmp_path / "memory")
    ctx = ContextBuilder(config=config, memory=memory)
    prompt = ctx.build_task_prompt("Analyze BTC conditions")
    assert "Analyze BTC conditions" in prompt
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_context.py -v`
Expected: FAIL

**Step 3: Write minimal implementation**

```python
# goodoracle/context.py
"""Context builder — assembles prompts from identity, memory, and tools."""

from goodoracle.config import Config
from goodoracle.memory import Memory


class ContextBuilder:
    def __init__(self, config: Config, memory: Memory):
        self.config = config
        self.memory = memory

    def _read_profile_file(self, name: str) -> str:
        path = self.config.profile_dir / name
        return path.read_text() if path.exists() else ""

    def build_system_prompt(self) -> str:
        sections = []

        identity = self._read_profile_file("identity.md")
        if identity:
            sections.append(f"## Identity\n\n{identity}")

        tools = self._read_profile_file("tools.md")
        if tools:
            sections.append(f"## Available Tools\n\n{tools}")

        long_term = self.memory.read_long_term()
        if long_term:
            sections.append(f"## Long-Term Memory\n\n{long_term}")

        recent = self.memory.read_recent(days=7)
        if recent:
            sections.append(f"## Recent Observations\n\n{recent}")

        return "\n\n---\n\n".join(sections)

    def build_task_prompt(self, task: str) -> str:
        return (
            f"## Task\n\n{task}\n\n"
            "## Instructions\n\n"
            "1. Analyze the task using your identity, memory, and tools.\n"
            "2. Take action or provide analysis.\n"
            "3. State any observations worth remembering (prefix with MEMORY:).\n"
            "4. State any messages to send to the user (prefix with TELEGRAM:).\n"
        )
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_context.py -v`
Expected: PASS (4 passed)

**Step 5: Commit**

```bash
git add goodoracle/context.py tests/test_context.py
git commit -m "feat: context builder assembles prompt from profile + memory"
```

---

## Phase 3: Agent Core

### Task 7: Mailbox (Task Queue)

**Files:**
- Create: `goodoracle/mailbox.py`
- Create: `tests/test_mailbox.py`

**Step 1: Write the failing test**

```python
# tests/test_mailbox.py
from goodoracle.mailbox import Mailbox, TaskStatus


def test_enqueue_and_dequeue(tmp_path):
    db_path = tmp_path / "test.db"
    mb = Mailbox(db_path)
    task_id = mb.enqueue("Analyze BTC market", source="telegram")
    assert task_id > 0

    task = mb.dequeue()
    assert task is not None
    assert task["content"] == "Analyze BTC market"
    assert task["source"] == "telegram"
    assert task["status"] == TaskStatus.PROCESSING


def test_dequeue_empty(tmp_path):
    db_path = tmp_path / "test.db"
    mb = Mailbox(db_path)
    assert mb.dequeue() is None


def test_complete_task(tmp_path):
    db_path = tmp_path / "test.db"
    mb = Mailbox(db_path)
    mb.enqueue("Do something", source="cli")
    task = mb.dequeue()
    mb.complete(task["id"], result="Done successfully")

    history = mb.history(limit=1)
    assert len(history) == 1
    assert history[0]["status"] == TaskStatus.DONE
    assert history[0]["result"] == "Done successfully"


def test_fail_task(tmp_path):
    db_path = tmp_path / "test.db"
    mb = Mailbox(db_path)
    mb.enqueue("Bad task", source="cron")
    task = mb.dequeue()
    mb.fail(task["id"], error="Something broke")

    history = mb.history(limit=1)
    assert history[0]["status"] == TaskStatus.FAILED


def test_fifo_order(tmp_path):
    db_path = tmp_path / "test.db"
    mb = Mailbox(db_path)
    mb.enqueue("First", source="cli")
    mb.enqueue("Second", source="cli")

    task1 = mb.dequeue()
    task2 = mb.dequeue()
    assert task1["content"] == "First"
    assert task2["content"] == "Second"
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_mailbox.py -v`
Expected: FAIL

**Step 3: Write minimal implementation**

```python
# goodoracle/mailbox.py
"""Mailbox — SQLite-backed task queue."""

import sqlite3
from datetime import datetime, timezone
from pathlib import Path


class TaskStatus:
    PENDING = "pending"
    PROCESSING = "processing"
    DONE = "done"
    FAILED = "failed"


class Mailbox:
    def __init__(self, db_path: Path):
        db_path.parent.mkdir(parents=True, exist_ok=True)
        self.conn = sqlite3.connect(str(db_path))
        self.conn.row_factory = sqlite3.Row
        self._migrate()

    def _migrate(self) -> None:
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS tasks (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                content TEXT NOT NULL,
                source TEXT NOT NULL,
                status TEXT NOT NULL DEFAULT 'pending',
                result TEXT,
                created_at TEXT NOT NULL,
                updated_at TEXT NOT NULL
            )
        """)
        self.conn.commit()

    def enqueue(self, content: str, source: str) -> int:
        now = datetime.now(timezone.utc).isoformat()
        cursor = self.conn.execute(
            "INSERT INTO tasks (content, source, status, created_at, updated_at) VALUES (?, ?, ?, ?, ?)",
            (content, source, TaskStatus.PENDING, now, now),
        )
        self.conn.commit()
        return cursor.lastrowid

    def dequeue(self) -> dict | None:
        row = self.conn.execute(
            "SELECT * FROM tasks WHERE status = ? ORDER BY id ASC LIMIT 1",
            (TaskStatus.PENDING,),
        ).fetchone()
        if row is None:
            return None
        now = datetime.now(timezone.utc).isoformat()
        self.conn.execute(
            "UPDATE tasks SET status = ?, updated_at = ? WHERE id = ?",
            (TaskStatus.PROCESSING, now, row["id"]),
        )
        self.conn.commit()
        return dict(row) | {"status": TaskStatus.PROCESSING}

    def complete(self, task_id: int, result: str = "") -> None:
        now = datetime.now(timezone.utc).isoformat()
        self.conn.execute(
            "UPDATE tasks SET status = ?, result = ?, updated_at = ? WHERE id = ?",
            (TaskStatus.DONE, result, now, task_id),
        )
        self.conn.commit()

    def fail(self, task_id: int, error: str = "") -> None:
        now = datetime.now(timezone.utc).isoformat()
        self.conn.execute(
            "UPDATE tasks SET status = ?, result = ?, updated_at = ? WHERE id = ?",
            (TaskStatus.FAILED, error, now, task_id),
        )
        self.conn.commit()

    def history(self, limit: int = 10) -> list[dict]:
        rows = self.conn.execute(
            "SELECT * FROM tasks ORDER BY id DESC LIMIT ?", (limit,)
        ).fetchall()
        return [dict(r) for r in rows]
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_mailbox.py -v`
Expected: PASS (5 passed)

**Step 5: Commit**

```bash
git add goodoracle/mailbox.py tests/test_mailbox.py
git commit -m "feat: SQLite-backed mailbox task queue"
```

---

### Task 8: Tool System

**Files:**
- Create: `goodoracle/tools/__init__.py`
- Create: `goodoracle/tools/base.py`
- Create: `goodoracle/tools/shell.py`
- Create: `tests/test_tools.py`

**Step 1: Write the failing test**

```python
# tests/test_tools.py
from goodoracle.tools.base import Tool, ToolRegistry
from goodoracle.tools.shell import ShellTool


def test_tool_registry():
    registry = ToolRegistry()
    tool = ShellTool()
    registry.register(tool)
    assert "shell" in registry.list_tools()
    assert registry.get("shell") is tool


def test_tool_registry_describe():
    registry = ToolRegistry()
    registry.register(ShellTool())
    desc = registry.describe_all()
    assert "shell" in desc.lower()


def test_shell_tool_execute():
    tool = ShellTool()
    result = tool.execute(command="echo hello")
    assert "hello" in result


def test_shell_tool_metadata():
    tool = ShellTool()
    assert tool.name == "shell"
    assert tool.description
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_tools.py -v`
Expected: FAIL

**Step 3: Write minimal implementation**

```python
# goodoracle/tools/base.py
"""Tool interface and registry."""

from abc import ABC, abstractmethod


class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...

    @property
    @abstractmethod
    def description(self) -> str: ...

    @abstractmethod
    def execute(self, **kwargs) -> str: ...


class ToolRegistry:
    def __init__(self):
        self._tools: dict[str, Tool] = {}

    def register(self, tool: Tool) -> None:
        self._tools[tool.name] = tool

    def get(self, name: str) -> Tool | None:
        return self._tools.get(name)

    def list_tools(self) -> list[str]:
        return list(self._tools.keys())

    def describe_all(self) -> str:
        lines = []
        for tool in self._tools.values():
            lines.append(f"- **{tool.name}**: {tool.description}")
        return "\n".join(lines)
```

```python
# goodoracle/tools/shell.py
"""Shell command execution tool."""

import subprocess
from goodoracle.tools.base import Tool


class ShellTool(Tool):
    @property
    def name(self) -> str:
        return "shell"

    @property
    def description(self) -> str:
        return "Execute a shell command and return stdout."

    def execute(self, **kwargs) -> str:
        command = kwargs.get("command", "")
        try:
            result = subprocess.run(
                command, shell=True, capture_output=True, text=True, timeout=30
            )
            return result.stdout.strip() or result.stderr.strip()
        except subprocess.TimeoutExpired:
            return "ERROR: Command timed out after 30s"
```

```python
# goodoracle/tools/__init__.py
"""Tool registry and built-in tools."""

from goodoracle.tools.base import Tool, ToolRegistry
from goodoracle.tools.shell import ShellTool

__all__ = ["Tool", "ToolRegistry", "ShellTool"]
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_tools.py -v`
Expected: PASS (4 passed)

**Step 5: Commit**

```bash
git add goodoracle/tools/ tests/test_tools.py
git commit -m "feat: tool system with registry and shell tool"
```

---

### Task 9: Agent Core Loop

**Files:**
- Create: `goodoracle/agent.py`
- Create: `tests/test_agent.py`

**Step 1: Write the failing test**

```python
# tests/test_agent.py
from pathlib import Path
from unittest.mock import patch

from goodoracle.agent import Agent
from goodoracle.config import Config


def _setup_agent(tmp_path: Path) -> Agent:
    """Create an agent with a minimal profile."""
    profile_dir = tmp_path / "profiles" / "test"
    profile_dir.mkdir(parents=True)
    (profile_dir / "identity.md").write_text("You are a test agent.")
    (tmp_path / "data").mkdir(exist_ok=True)

    config = Config(profile="test", project_root=tmp_path)
    return Agent(config)


def test_agent_process_task(tmp_path):
    agent = _setup_agent(tmp_path)
    agent.mailbox.enqueue("What is 2+2?", source="test")

    with patch("goodoracle.claude.invoke_claude", return_value="The answer is 4."):
        processed = agent.process_next()
        assert processed is True


def test_agent_process_empty_queue(tmp_path):
    agent = _setup_agent(tmp_path)
    processed = agent.process_next()
    assert processed is False


def test_agent_updates_memory_on_memory_prefix(tmp_path):
    agent = _setup_agent(tmp_path)
    agent.mailbox.enqueue("Learn something", source="test")

    response = "Analysis done.\nMEMORY: BTC spread is consistently $1.01"
    with patch("goodoracle.claude.invoke_claude", return_value=response):
        agent.process_next()

    long_term = agent.memory.read_long_term()
    assert "BTC spread" in long_term


def test_agent_captures_telegram_messages(tmp_path):
    agent = _setup_agent(tmp_path)
    agent.mailbox.enqueue("Report status", source="test")

    response = "Here's the report.\nTELEGRAM: System is healthy, no action needed."
    with patch("goodoracle.claude.invoke_claude", return_value=response):
        agent.process_next()

    assert len(agent.pending_messages) == 1
    assert "System is healthy" in agent.pending_messages[0]
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_agent.py -v`
Expected: FAIL

**Step 3: Write minimal implementation**

```python
# goodoracle/agent.py
"""Core agent loop — dequeue, think, act."""

import logging
from goodoracle.claude import invoke_claude, ClaudeError
from goodoracle.config import Config
from goodoracle.context import ContextBuilder
from goodoracle.mailbox import Mailbox
from goodoracle.memory import Memory

log = logging.getLogger(__name__)


class Agent:
    def __init__(self, config: Config):
        self.config = config
        self.memory = Memory(config.memory_dir)
        self.mailbox = Mailbox(config.db_path)
        self.context = ContextBuilder(config, self.memory)
        self.pending_messages: list[str] = []

    def process_next(self) -> bool:
        """Process the next task from the mailbox. Returns True if a task was processed."""
        task = self.mailbox.dequeue()
        if task is None:
            return False

        log.info("Processing task %d: %s", task["id"], task["content"][:80])

        try:
            system_prompt = self.context.build_system_prompt()
            task_prompt = self.context.build_task_prompt(task["content"])
            response = invoke_claude(task_prompt, system_prompt=system_prompt)
            self._process_response(response)
            self.mailbox.complete(task["id"], result=response[:500])
            self.memory.append_today(f"Task: {task['content'][:100]}\nResult: {response[:200]}")
            log.info("Task %d completed", task["id"])
        except ClaudeError as e:
            self.mailbox.fail(task["id"], error=str(e))
            log.error("Task %d failed: %s", task["id"], e)

        return True

    def _process_response(self, response: str) -> None:
        """Extract MEMORY: and TELEGRAM: directives from Claude's response."""
        for line in response.split("\n"):
            stripped = line.strip()
            if stripped.startswith("MEMORY:"):
                content = stripped[len("MEMORY:"):].strip()
                self.memory.append_long_term(content)
            elif stripped.startswith("TELEGRAM:"):
                content = stripped[len("TELEGRAM:"):].strip()
                self.pending_messages.append(content)

    def run_all(self) -> int:
        """Process all pending tasks. Returns count of tasks processed."""
        count = 0
        while self.process_next():
            count += 1
        return count
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_agent.py -v`
Expected: PASS (4 passed)

**Step 5: Commit**

```bash
git add goodoracle/agent.py tests/test_agent.py
git commit -m "feat: core agent loop with memory + telegram extraction"
```

---

## Phase 4: Communication & CLI

### Task 10: Telegram Integration

**Files:**
- Create: `goodoracle/telegram_bot.py`
- Create: `tests/test_telegram.py`

**Step 1: Write the failing test**

```python
# tests/test_telegram.py
from unittest.mock import AsyncMock
import pytest

from goodoracle.telegram_bot import TelegramBot


def test_telegram_bot_init():
    bot = TelegramBot(token="fake-token", chat_id="12345")
    assert bot.chat_id == "12345"


@pytest.mark.asyncio
async def test_send_message():
    bot = TelegramBot(token="fake-token", chat_id="12345")
    mock_bot = AsyncMock()
    bot._bot = mock_bot

    await bot.send("Hello from GoodOracle")
    mock_bot.send_message.assert_called_once_with(
        chat_id="12345", text="Hello from GoodOracle"
    )


def test_truncate_long_message():
    bot = TelegramBot(token="fake-token", chat_id="12345")
    long_msg = "x" * 5000
    truncated = bot._truncate(long_msg)
    assert len(truncated) <= 4096
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_telegram.py -v`
Expected: FAIL

**Step 3: Write minimal implementation**

```python
# goodoracle/telegram_bot.py
"""Telegram integration — send and receive messages."""

import asyncio
import logging
from telegram import Bot

log = logging.getLogger(__name__)

MAX_MESSAGE_LENGTH = 4096


class TelegramBot:
    def __init__(self, token: str, chat_id: str):
        self.token = token
        self.chat_id = chat_id
        self._bot = Bot(token=token)

    async def send(self, text: str) -> None:
        """Send a message to the configured chat."""
        text = self._truncate(text)
        try:
            await self._bot.send_message(chat_id=self.chat_id, text=text)
        except Exception as e:
            log.error("Failed to send Telegram message: %s", e)

    def send_sync(self, text: str) -> None:
        """Synchronous wrapper for send."""
        asyncio.run(self.send(text))

    @staticmethod
    def _truncate(text: str) -> str:
        if len(text) <= MAX_MESSAGE_LENGTH:
            return text
        return text[: MAX_MESSAGE_LENGTH - 20] + "\n\n... (truncated)"
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_telegram.py -v`
Expected: PASS (3 passed)

**Step 5: Commit**

```bash
git add goodoracle/telegram_bot.py tests/test_telegram.py
git commit -m "feat: Telegram bot integration for sending messages"
```

---

### Task 11: CLI Entry Point

**Files:**
- Create: `goodoracle/cli.py`
- Create: `goodoracle/__main__.py`
- Create: `tests/test_cli.py`

**Step 1: Write the failing test**

```python
# tests/test_cli.py
from goodoracle.cli import create_parser


def test_parser_task_command():
    parser = create_parser()
    args = parser.parse_args(["task", "--profile", "trading", "Analyze BTC"])
    assert args.command == "task"
    assert args.profile == "trading"
    assert args.prompt == "Analyze BTC"


def test_parser_evaluate_command():
    parser = create_parser()
    args = parser.parse_args(["evaluate", "--profile", "trading"])
    assert args.command == "evaluate"
    assert args.profile == "trading"


def test_parser_run_command():
    parser = create_parser()
    args = parser.parse_args(["run", "--profile", "trading"])
    assert args.command == "run"


def test_parser_default_profile():
    parser = create_parser()
    args = parser.parse_args(["task", "Do something"])
    assert args.profile == "trading"  # default
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_cli.py -v`
Expected: FAIL

**Step 3: Write minimal implementation**

```python
# goodoracle/cli.py
"""CLI entry point — run, evaluate, task."""

import argparse
import logging
from pathlib import Path

from goodoracle.agent import Agent
from goodoracle.config import Config
from goodoracle.telegram_bot import TelegramBot


def create_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(prog="goodoracle", description="Autonomous agent framework")

    sub = parser.add_subparsers(dest="command")

    # goodoracle task "prompt"
    task_cmd = sub.add_parser("task", help="Process a single task")
    task_cmd.add_argument("prompt", help="Task to process")
    task_cmd.add_argument("--profile", default="trading")

    # goodoracle evaluate
    eval_cmd = sub.add_parser("evaluate", help="Run evaluation cycle")
    eval_cmd.add_argument("--profile", default="trading")

    # goodoracle run
    run_cmd = sub.add_parser("run", help="Run in polling mode")
    run_cmd.add_argument("--profile", default="trading")

    return parser


def main() -> None:
    logging.basicConfig(
        level=logging.INFO,
        format="%(asctime)s %(name)s %(levelname)s %(message)s",
    )
    parser = create_parser()
    args = parser.parse_args()

    project_root = Path(__file__).parent.parent
    config = Config(profile=args.profile, project_root=project_root)
    agent = Agent(config)

    if args.command == "task":
        agent.mailbox.enqueue(args.prompt, source="cli")
        agent.process_next()
        _send_pending(config, agent)

    elif args.command == "evaluate":
        evaluation_path = config.profile_dir / "evaluation.md"
        if evaluation_path.exists():
            prompt = evaluation_path.read_text()
        else:
            prompt = "Run a full evaluation of the current system state. Report findings."
        agent.mailbox.enqueue(prompt, source="cron")
        agent.process_next()
        _send_pending(config, agent)

    elif args.command == "run":
        print("Polling mode not yet implemented. Use 'task' or 'evaluate'.")

    else:
        parser.print_help()


def _send_pending(config: Config, agent: Agent) -> None:
    if agent.pending_messages and config.telegram_bot_token:
        bot = TelegramBot(token=config.telegram_bot_token, chat_id=config.telegram_chat_id)
        for msg in agent.pending_messages:
            bot.send_sync(msg)
        agent.pending_messages.clear()
```

```python
# goodoracle/__main__.py
"""Allow running as: python -m goodoracle"""

from goodoracle.cli import main

main()
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_cli.py -v`
Expected: PASS (4 passed)

**Step 5: Commit**

```bash
git add goodoracle/cli.py goodoracle/__main__.py tests/test_cli.py
git commit -m "feat: CLI entry point with task, evaluate, and run commands"
```

---

## Phase 5: Scheduling & Automation

### Task 12: Cron/LaunchD Setup

**Files:**
- Create: `scripts/com.goodoracle.agent.plist`
- Create: `scripts/install-schedule.sh`

**Step 1: Write launchd plist**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.goodoracle.agent</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/petr.kubelka/git_projects/goodoracle/.venv/bin/goodoracle</string>
        <string>evaluate</string>
        <string>--profile</string>
        <string>trading</string>
    </array>
    <key>WorkingDirectory</key>
    <string>/Users/petr.kubelka/git_projects/goodoracle</string>
    <key>StartInterval</key>
    <integer>21600</integer>
    <key>StandardOutPath</key>
    <string>/Users/petr.kubelka/git_projects/goodoracle/logs/agent.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/petr.kubelka/git_projects/goodoracle/logs/agent-error.log</string>
    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/Users/petr.kubelka/.local/bin</string>
    </dict>
</dict>
</plist>
```

**Step 2: Write install script**

```bash
#!/usr/bin/env bash
# scripts/install-schedule.sh — Install goodoracle as a macOS launchd service
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PROJECT_DIR="$(dirname "$SCRIPT_DIR")"
PLIST_NAME="com.goodoracle.agent"
PLIST_SRC="$SCRIPT_DIR/$PLIST_NAME.plist"
PLIST_DST="$HOME/Library/LaunchAgents/$PLIST_NAME.plist"

echo "Installing goodoracle scheduled agent..."

mkdir -p "$PROJECT_DIR/logs"

cp "$PLIST_SRC" "$PLIST_DST"

launchctl unload "$PLIST_DST" 2>/dev/null || true
launchctl load "$PLIST_DST"

echo "Installed. Agent will run every 6 hours."
echo "  Check status:  launchctl list | grep goodoracle"
echo "  Run now:        launchctl start $PLIST_NAME"
echo "  Uninstall:      launchctl unload $PLIST_DST && rm $PLIST_DST"
```

**Step 3: Make executable and commit**

```bash
chmod +x scripts/install-schedule.sh
git add scripts/
git commit -m "feat: launchd scheduling for 6-hour evaluation cycle"
```

---

## Phase 6: Trading Integration

### Task 13: Trading Profile

**Files:**
- Create: `profiles/trading/identity.md`
- Create: `profiles/trading/tools.md`
- Create: `profiles/trading/evaluation.md`

**Step 1: Write identity.md** (migrated from trading-architect skill thinking)

```markdown
# Trading Oracle

You are an autonomous trading analyst managing a Polymarket prediction market system.

## Your Mission

Maximize profit on Polymarket prediction markets. You think in expected value:

    EV = (win_rate * avg_win) - (loss_rate * avg_loss) - fees - slippage

## Your System

- **Infrastructure:** Rust trading bots on AWS (t3.micro, eu-west-2)
- **Markets:** Polymarket BTC 15-minute prediction markets (expanding to ETH and others)
- **Data:** Binance real-time feed + Polymarket CLOB order book
- **Monitoring:** Grafana Cloud (Prometheus metrics + Loki logs)
- **Project root:** /Users/petr.kubelka/git_projects/trading
- **CLAUDE.md:** /Users/petr.kubelka/git_projects/trading/CLAUDE.md

## How You Think

1. **Hypothesis-driven:** Every strategy is a falsifiable hypothesis. Define pass/fail criteria BEFORE testing.
2. **Evidence-based:** Decisions come from data, not intuition. Query Grafana, read the database, check logs.
3. **ELI5 communication:** Explain everything simply. Numbers first, then interpretation.
4. **Risk-aware:** Know your edge, know when it decays, know when to stop.
5. **Two tracks:** Development (implementation roadmap) and Analysis (hypothesis testing). Track both.

## Your Counterparties

- Market makers (tight spreads, fast execution)
- LLM bots (herd behavior, exploitable via mean reversion)
- Retail humans (slow, emotional, predictable)
- Other arb bots (competing for same edge)

## Key Files to Read

- Hypotheses: /Users/petr.kubelka/git_projects/trading/docs/HYPOTHESES.md
- Journal: /Users/petr.kubelka/git_projects/trading/docs/TRADING-JOURNAL.md
- Config: /Users/petr.kubelka/git_projects/trading/config/default.toml
- Status: /Users/petr.kubelka/git_projects/trading/docs/plans/paper_trading_mvp/STATUS.md
- Implementation plan: /Users/petr.kubelka/git_projects/trading/docs/implementation-plan.md
```

**Step 2: Write tools.md**

```markdown
# Available Tools

## Grafana Queries

Query live infrastructure metrics using the Grafana API.

```bash
source /Users/petr.kubelka/git_projects/trading/.env.agent
/Users/petr.kubelka/git_projects/trading/scripts/query-grafana.sh pnl
/Users/petr.kubelka/git_projects/trading/scripts/query-grafana.sh latency
/Users/petr.kubelka/git_projects/trading/scripts/query-grafana.sh health
/Users/petr.kubelka/git_projects/trading/scripts/query-grafana.sh errors
```

## SSH to Server

```bash
ssh -i /Users/petr.kubelka/git_projects/trading/trading-bot.pem ubuntu@$(cat /Users/petr.kubelka/git_projects/trading/deploy/.server-info | head -1)
```

Check process health: `systemctl status bot-arb latency-harness`
Check logs: `journalctl -u bot-arb --since "6 hours ago" --no-pager | tail -50`

## SQLite Database

Production data: `/Users/petr.kubelka/git_projects/trading/data/trading.db`

Useful queries:
```bash
sqlite3 /Users/petr.kubelka/git_projects/trading/data/trading.db "SELECT COUNT(*) FROM binance_ticks;"
sqlite3 /Users/petr.kubelka/git_projects/trading/data/trading.db "SELECT COUNT(*) FROM polymarket_books;"
sqlite3 /Users/petr.kubelka/git_projects/trading/data/trading.db "SELECT * FROM paper_trades ORDER BY id DESC LIMIT 5;"
```

## File System

Read and modify project files:
- `docs/HYPOTHESES.md` — hypothesis tracker (read + update)
- `docs/TRADING-JOURNAL.md` — session journal (append findings)
- `config/default.toml` — trading bot configuration (read, suggest changes)

## Web Search

Search for market news, competitor analysis, strategy research.
```

**Step 3: Write evaluation.md**

```markdown
# Evaluation Cycle

Run this evaluation every 6 hours. Report findings via Telegram.

## Steps

### 1. Health Check
- Are bot processes alive? (SSH or Grafana health query)
- Are data feeds flowing? (Binance ticks count, Polymarket books count)
- Any errors in last 6 hours? (Grafana Loki error query)
- Memory/CPU stable? (Grafana process metrics)

### 2. Performance Review
- Paper PnL since last evaluation
- Number of opportunities detected vs. acted on
- Current spread levels (combined YES+NO ask)
- Fill rate for paper trades

### 3. Hypothesis Update
- Read current hypotheses from docs/HYPOTHESES.md
- Check if new evidence supports state transitions
- Propose new hypotheses if market conditions changed
- Update HYPOTHESES.md if warranted

### 4. Market Analysis
- Current BTC volatility and direction
- Polymarket spread patterns (are they tightening or widening?)
- Any new market types worth exploring? (ETH, other events)
- External factors (exchange news, regulatory changes)

### 5. Roadmap Check
- What was the last development task completed?
- What should be worked on next? (check STATUS.md and implementation-plan.md)
- Any blocked tasks that need attention?
- Priority: what gives the highest EV improvement per hour of dev work?

### 6. Recommendations
For each recommendation, provide:
- **What:** Specific action to take
- **Why (ELI5):** Simple explanation with numbers
- **Priority:** High/Medium/Low
- **Effort:** Rough estimate (quick fix / half day / full day)
- **Risk:** What could go wrong

## Output Format

1. Prefix Telegram-bound summary with TELEGRAM: (keep under 500 words, lead with most important finding)
2. Prefix new knowledge with MEMORY: (facts worth remembering across sessions)
3. Update TRADING-JOURNAL.md with detailed findings
4. Update HYPOTHESES.md if hypothesis states changed
```

**Step 4: Commit**

```bash
git add profiles/
git commit -m "feat: trading profile with identity, tools, and evaluation cycle"
```

---

### Task 14: Grafana Query Tool

**Files:**
- Create: `goodoracle/tools/grafana.py`
- Create: `tests/test_grafana.py`

**Step 1: Write the failing test**

```python
# tests/test_grafana.py
from unittest.mock import patch, MagicMock

from goodoracle.tools.grafana import GrafanaTool


def test_grafana_tool_metadata():
    tool = GrafanaTool(script_path="/fake/path", env_file="/fake/.env")
    assert tool.name == "grafana"
    assert "query" in tool.description.lower()


def test_grafana_tool_execute():
    tool = GrafanaTool(script_path="/fake/path", env_file="/fake/.env")

    mock_result = MagicMock()
    mock_result.stdout = "pnl_total: $0.00"
    mock_result.returncode = 0

    with patch("subprocess.run", return_value=mock_result):
        result = tool.execute(query="pnl")
        assert "$0.00" in result
```

**Step 2: Run test to verify it fails**

Run: `pytest tests/test_grafana.py -v`
Expected: FAIL

**Step 3: Write minimal implementation**

```python
# goodoracle/tools/grafana.py
"""Grafana query tool — wraps query-grafana.sh."""

import subprocess
from goodoracle.tools.base import Tool


class GrafanaTool(Tool):
    def __init__(self, script_path: str, env_file: str):
        self.script_path = script_path
        self.env_file = env_file

    @property
    def name(self) -> str:
        return "grafana"

    @property
    def description(self) -> str:
        return "Query Grafana for trading metrics (pnl, latency, health, errors)."

    def execute(self, **kwargs) -> str:
        query = kwargs.get("query", "health")
        try:
            result = subprocess.run(
                ["bash", "-c", f"source {self.env_file} && {self.script_path} {query}"],
                capture_output=True,
                text=True,
                timeout=30,
            )
            return result.stdout.strip() or result.stderr.strip()
        except subprocess.TimeoutExpired:
            return "ERROR: Grafana query timed out"
```

**Step 4: Run test to verify it passes**

Run: `pytest tests/test_grafana.py -v`
Expected: PASS (2 passed)

**Step 5: Commit**

```bash
git add goodoracle/tools/grafana.py tests/test_grafana.py
git commit -m "feat: Grafana query tool for trading metrics"
```

---

## Phase 7: Skills & Project Cleanup

### Task 15: Create goodoracle-dev Skill

This Claude Code skill bridges interactive development sessions with goodoracle's autonomous analysis.

**Files:**
- Create: `~/.claude/skills/goodoracle-dev/SKILL.md`

**Step 1: Write the skill**

```markdown
---
name: goodoracle-dev
description: Development assistant informed by GoodOracle's analysis. Reads the oracle's latest evaluation, memory, and hypotheses to guide implementation work on monitored projects.
---

# GoodOracle Development Assistant

## Overview

You are working on a project that GoodOracle (an autonomous agent) is monitoring and analyzing.
Before implementing anything, check what the oracle knows and recommends.

**Announce at start:** "I'm using goodoracle-dev to check what the oracle recommends."

## When to Use

- Starting an implementation session on any GoodOracle-monitored project
- Picking what to work on next
- Implementing a recommendation from the oracle's evaluation

## Process

### Step 1: Read Oracle State

Read these files to understand current state:

1. **Long-term memory:** `/Users/petr.kubelka/git_projects/goodoracle/memory/MEMORY.md`
2. **Recent observations:** Check `/Users/petr.kubelka/git_projects/goodoracle/memory/daily/` for last 3 days
3. **Task history:**
   ```bash
   sqlite3 /Users/petr.kubelka/git_projects/goodoracle/data/goodoracle.db \
     "SELECT content, result, status, created_at FROM tasks ORDER BY id DESC LIMIT 5;"
   ```

### Step 2: Read Project State

For the **trading** project:
1. **Hypotheses:** `/Users/petr.kubelka/git_projects/trading/docs/HYPOTHESES.md`
2. **Journal:** `/Users/petr.kubelka/git_projects/trading/docs/TRADING-JOURNAL.md`
3. **Status:** `/Users/petr.kubelka/git_projects/trading/docs/plans/paper_trading_mvp/STATUS.md`

### Step 3: Present Prioritized Work

Based on oracle state, present:
1. What the oracle most recently recommended (from task results)
2. What hypotheses need attention (from HYPOTHESES.md)
3. What implementation tasks are pending (from STATUS.md)
4. Your own assessment of highest-EV work

Let the developer choose what to work on.

### Step 4: Implement with TDD

Use superpowers skills for implementation:
- `superpowers:test-driven-development` for all coding
- `superpowers:writing-plans` if the task is complex
- `superpowers:verification-before-completion` before claiming done

### Step 5: Update Oracle Memory

After completing work, update GoodOracle's daily memory:

```bash
echo "## Implementation Session $(date +%H:%M)

- **Task:** [what was done]
- **Result:** [outcome]
- **Next:** [what should happen next]
" >> /Users/petr.kubelka/git_projects/goodoracle/memory/daily/$(date +%Y-%m-%d).md
```

Also update the project's own journal/status docs as appropriate.

## Remember

- Oracle's analysis may be hours old — verify assumptions against current state
- Don't blindly follow oracle recommendations — use your judgment
- Always update oracle memory so the next evaluation cycle has fresh context
```

**Step 2: Verify skill appears**

Run a new Claude Code session and check that `goodoracle-dev` appears in available skills.

---

### Task 16: Archive Old Trading Skills

**Step 1: Back up existing skills to goodoracle repo**

```bash
mkdir -p /Users/petr.kubelka/git_projects/goodoracle/docs/archived-skills
cp -r /Users/petr.kubelka/.claude/skills/trading-architect /Users/petr.kubelka/git_projects/goodoracle/docs/archived-skills/
cp -r /Users/petr.kubelka/.claude/skills/trading-guidance /Users/petr.kubelka/git_projects/goodoracle/docs/archived-skills/
cp -r /Users/petr.kubelka/.claude/skills/trading-hypotheses /Users/petr.kubelka/git_projects/goodoracle/docs/archived-skills/
cp -r /Users/petr.kubelka/.claude/skills/trading-system-checkup /Users/petr.kubelka/git_projects/goodoracle/docs/archived-skills/
```

**Step 2: Remove old skills**

```bash
rm -rf /Users/petr.kubelka/.claude/skills/trading-architect
rm -rf /Users/petr.kubelka/.claude/skills/trading-guidance
rm -rf /Users/petr.kubelka/.claude/skills/trading-hypotheses
rm -rf /Users/petr.kubelka/.claude/skills/trading-system-checkup
```

**Step 3: Verify only goodoracle-dev remains**

```bash
ls /Users/petr.kubelka/.claude/skills/
# Expected: goodoracle-dev/
```

**Step 4: Commit archived skills**

```bash
cd /Users/petr.kubelka/git_projects/goodoracle
git add docs/archived-skills/
git commit -m "docs: archive old trading skills (replaced by goodoracle agent)"
```

---

### Task 17: Update Trading Project CLAUDE.md

**Files:**
- Modify: `/Users/petr.kubelka/git_projects/trading/CLAUDE.md`

**Step 1: Add goodoracle reference section at the top**

Add this after the first heading:

```markdown
## Agent Orchestration

This project is monitored by **GoodOracle** (`/Users/petr.kubelka/git_projects/goodoracle`).

- **Autonomous evaluation:** GoodOracle runs every 6h via launchd, queries Grafana, analyzes hypotheses, sends Telegram reports
- **Development guidance:** Use `goodoracle-dev` skill at session start to see what the oracle recommends
- **Oracle memory:** `/Users/petr.kubelka/git_projects/goodoracle/memory/`
- **Manual trigger:** `cd ~/git_projects/goodoracle && goodoracle evaluate --profile trading`

The old `trading-architect`, `trading-guidance`, `trading-hypotheses`, and `trading-system-checkup` skills have been replaced by goodoracle. Archives in `goodoracle/docs/archived-skills/`.
```

**Step 2: Simplify the skills workflow section**

Replace references to "every session starts with trading-architect" with:
"Every session starts with `goodoracle-dev` skill which reads the oracle's latest analysis."

**Step 3: Commit**

```bash
cd /Users/petr.kubelka/git_projects/trading
git add CLAUDE.md
git commit -m "docs: reference GoodOracle for autonomous orchestration"
```

---

## Phase 8: Playground & Real Integration Testing

**Philosophy: No mocks for integration tests. Every test runs real Claude CLI, writes real memory, touches real SQLite. Mocks are only acceptable in unit tests for pure logic (config parsing, memory file I/O, mailbox SQL). The smoke test is the final proof that the system works.**

### Task 18: Playground Profile

The playground profile is a safe, self-contained test environment. No external dependencies (no Grafana, no SSH, no Telegram needed). Just Claude + filesystem.

**Files:**
- Create: `profiles/playground/identity.md`
- Create: `profiles/playground/tools.md`
- Create: `profiles/playground/evaluation.md`

**Step 1: Write playground identity.md**

```markdown
# Playground Agent

You are a test agent for verifying the GoodOracle framework works correctly.

## Rules

1. Always respond concisely and precisely.
2. Always include at least one MEMORY: line with a factual observation from your work.
3. Always include at least one TELEGRAM: line summarizing your response in one sentence.
4. When asked to read files, read them and report exact data (line counts, headings, etc.).
5. Never make up data — if a file doesn't exist, say so.

## Your Project

You are running inside: /Users/petr.kubelka/git_projects/goodoracle
This is the GoodOracle autonomous agent framework.

Key files you can read:
- CLAUDE.md — project manifest
- pyproject.toml — project config
- memory/MEMORY.md — your long-term memory
- memory/daily/ — your daily observations
```

**Step 2: Write playground tools.md**

```markdown
# Available Tools

## File System

You can read any file in the project directory:
- `/Users/petr.kubelka/git_projects/goodoracle/CLAUDE.md`
- `/Users/petr.kubelka/git_projects/goodoracle/pyproject.toml`
- `/Users/petr.kubelka/git_projects/goodoracle/memory/MEMORY.md`

## Shell

You can run simple shell commands to inspect the project:
```bash
ls /Users/petr.kubelka/git_projects/goodoracle/
wc -l /Users/petr.kubelka/git_projects/goodoracle/CLAUDE.md
sqlite3 /Users/petr.kubelka/git_projects/goodoracle/data/goodoracle.db "SELECT COUNT(*) FROM tasks;"
```
```

**Step 3: Write playground evaluation.md**

```markdown
# Playground Evaluation

Run a self-check of the GoodOracle framework. This verifies memory, file access, and response directives all work.

## Steps

1. Read CLAUDE.md and count the number of second-level headings (## lines).
2. Read memory/MEMORY.md and summarize what long-term knowledge exists (or note it's empty).
3. Check if data/goodoracle.db exists. If so, count total tasks with:
   `sqlite3 /Users/petr.kubelka/git_projects/goodoracle/data/goodoracle.db "SELECT COUNT(*) FROM tasks;"`
4. Read today's daily memory file if it exists.
5. Summarize findings.

## Output

- MEMORY: Playground evaluation ran successfully. Found N headings in CLAUDE.md, M tasks in mailbox, K lines in long-term memory.
- TELEGRAM: Playground self-check passed. CLAUDE.md has N headings, M tasks processed, system operational.
```

**Step 4: Commit**

```bash
git add profiles/playground/
git commit -m "feat: playground profile for real integration testing"
```

---

### Task 19: Smoke Test Script

The smoke test runs the REAL full pipeline: Claude CLI is invoked, memory is written, mailbox is updated. No mocks. If `claude` isn't on PATH, the test fails — that's intentional.

**Files:**
- Create: `scripts/smoke-test.sh`

**Step 1: Write the smoke test**

```bash
#!/usr/bin/env bash
# scripts/smoke-test.sh — Full end-to-end verification with REAL Claude CLI
#
# This script tests the entire goodoracle pipeline:
#   1. Claude CLI is available and responds
#   2. A task can be enqueued, processed, and completed
#   3. Memory is updated (both daily and long-term)
#   4. The evaluation cycle works
#   5. Task history is queryable
#
# NO MOCKS. If something fails here, it's broken for real.

set -euo pipefail

PROJECT_DIR="$(cd "$(dirname "$0")/.." && pwd)"
cd "$PROJECT_DIR"

GOODORACLE=".venv/bin/goodoracle"
PASS=0
FAIL=0

pass() { echo "  ✅ $1"; PASS=$((PASS + 1)); }
fail() { echo "  ❌ $1"; FAIL=$((FAIL + 1)); }

echo "═══════════════════════════════════════════"
echo "  GoodOracle Smoke Test (REAL, NO MOCKS)"
echo "═══════════════════════════════════════════"
echo ""

# Clean state for reproducible test
echo "Cleaning previous state..."
make clean 2>/dev/null || true
echo ""

# ─── Test 1: Claude CLI available ──────────────────
echo "1. Checking Claude CLI..."
if claude --version > /dev/null 2>&1; then
    pass "Claude CLI found: $(claude --version 2>&1 | head -1)"
else
    fail "Claude CLI not on PATH"
    echo "   Cannot continue without Claude CLI. Install it first."
    exit 1
fi
echo ""

# ─── Test 2: goodoracle CLI installed ──────────────
echo "2. Checking goodoracle CLI..."
if $GOODORACLE --help > /dev/null 2>&1; then
    pass "goodoracle CLI installed"
else
    fail "goodoracle not installed. Run: make install"
    exit 1
fi
echo ""

# ─── Test 3: Simple task with playground profile ───
echo "3. Running playground task (REAL Claude call)..."
echo "   This calls Claude Opus 4.6 — may take 30-60 seconds..."
$GOODORACLE task --profile playground \
    "Read the file CLAUDE.md in /Users/petr.kubelka/git_projects/goodoracle/ and count the second-level headings (lines starting with ##). Respond with exactly: MEMORY: CLAUDE.md has N second-level headings. TELEGRAM: Counted N headings in CLAUDE.md."

# Verify memory was updated
if [ -f "memory/daily/$(date +%Y-%m-%d).md" ]; then
    pass "Daily memory file created"
else
    fail "Daily memory file NOT created"
fi

if grep -q "CLAUDE.md" "memory/MEMORY.md" 2>/dev/null; then
    pass "Long-term memory updated with CLAUDE.md observation"
else
    fail "Long-term memory NOT updated (expected CLAUDE.md mention)"
fi
echo ""

# ─── Test 4: Task recorded in mailbox ──────────────
echo "4. Checking mailbox database..."
if [ -f "data/goodoracle.db" ]; then
    TASK_COUNT=$(sqlite3 data/goodoracle.db "SELECT COUNT(*) FROM tasks WHERE status = 'done';")
    if [ "$TASK_COUNT" -ge 1 ]; then
        pass "Found $TASK_COUNT completed task(s) in mailbox"
    else
        fail "No completed tasks found in mailbox"
    fi
else
    fail "Database not created at data/goodoracle.db"
fi
echo ""

# ─── Test 5: Evaluation cycle ─────────────────────
echo "5. Running playground evaluation (REAL Claude call)..."
echo "   Another real Claude call — 30-60 seconds..."
$GOODORACLE evaluate --profile playground

EVAL_COUNT=$(sqlite3 data/goodoracle.db "SELECT COUNT(*) FROM tasks WHERE source = 'cron' AND status = 'done';")
if [ "$EVAL_COUNT" -ge 1 ]; then
    pass "Evaluation task completed successfully"
else
    fail "Evaluation task did not complete"
fi
echo ""

# ─── Test 6: Second task accumulates memory ────────
echo "6. Running second task to verify memory accumulation..."
$GOODORACLE task --profile playground \
    "Read memory/MEMORY.md and tell me how many lines of long-term memory exist. Respond with: MEMORY: Memory system is accumulating correctly. TELEGRAM: Memory has N lines of knowledge."

MEMORY_LINES=$(wc -l < memory/MEMORY.md 2>/dev/null || echo "0")
if [ "$MEMORY_LINES" -ge 2 ]; then
    pass "Memory accumulating: $MEMORY_LINES lines in long-term memory"
else
    fail "Memory not accumulating (only $MEMORY_LINES lines)"
fi
echo ""

# ─── Test 7: Task history queryable ────────────────
echo "7. Verifying task history..."
TOTAL_TASKS=$(sqlite3 data/goodoracle.db "SELECT COUNT(*) FROM tasks;")
DONE_TASKS=$(sqlite3 data/goodoracle.db "SELECT COUNT(*) FROM tasks WHERE status = 'done';")
FAILED_TASKS=$(sqlite3 data/goodoracle.db "SELECT COUNT(*) FROM tasks WHERE status = 'failed';")

if [ "$TOTAL_TASKS" -ge 3 ]; then
    pass "Task history: $TOTAL_TASKS total, $DONE_TASKS done, $FAILED_TASKS failed"
else
    fail "Expected at least 3 tasks, found $TOTAL_TASKS"
fi
echo ""

# ─── Summary ──────────────────────────────────────
echo "═══════════════════════════════════════════"
echo "  Results: $PASS passed, $FAIL failed"
echo "═══════════════════════════════════════════"
echo ""

# Show state for inspection
echo "📋 Current memory state:"
echo "--- Long-term (memory/MEMORY.md) ---"
cat memory/MEMORY.md 2>/dev/null || echo "(empty)"
echo ""
echo "--- Today (memory/daily/$(date +%Y-%m-%d).md) ---"
cat "memory/daily/$(date +%Y-%m-%d).md" 2>/dev/null || echo "(empty)"
echo ""
echo "📬 Task history:"
sqlite3 -header -column data/goodoracle.db \
    "SELECT id, status, source, substr(content, 1, 50) as task, created_at FROM tasks ORDER BY id;"
echo ""

if [ "$FAIL" -gt 0 ]; then
    echo "⚠️  Some tests failed. Check output above."
    exit 1
else
    echo "All smoke tests passed. GoodOracle is operational."
    exit 0
fi
```

**Step 2: Make executable**

```bash
chmod +x scripts/smoke-test.sh
```

**Step 3: Run the smoke test for real**

Run: `make smoke`
Expected: 7/7 tests pass with real Claude CLI invocations. Takes ~2-3 minutes total (3 real Claude calls).

**Step 4: Commit**

```bash
git add scripts/smoke-test.sh
git commit -m "feat: real E2E smoke test — no mocks, real Claude CLI"
```

---

### Task 20: Final Push & Verification

**Step 1: Run unit tests**

```bash
make test
```

Expected: All unit tests pass.

**Step 2: Run smoke test**

```bash
make smoke
```

Expected: All 7 E2E tests pass with real Claude calls.

**Step 3: Inspect state after smoke test**

```bash
make memory   # Show what the agent learned
make history  # Show task execution history
```

Verify:
- Long-term memory contains observations from smoke test tasks
- Daily memory has entries with timestamps
- All tasks show status "done" in history

**Step 4: Push to GitHub**

```bash
git add -A
git commit -m "feat: GoodOracle MVP — autonomous agent framework wrapping Claude Code CLI"
git push origin main
```

**Step 5: Test from clean clone (ultimate verification)**

```bash
cd /tmp
git clone git@github.com:KSonny4/goodoracle.git goodoracle-test
cd goodoracle-test
make install
make smoke
```

If this passes, the system is truly production-ready.

---

## Future Enhancements (Post-MVP)

These are NOT part of this plan but should be tracked:

1. **Telegram polling mode** (`goodoracle run`): Real-time interactive Q&A via Telegram bot with `python-telegram-bot`'s Application/Updater for webhook or long-polling.

2. **Multi-message Telegram output**: Break long responses into multiple messages instead of truncating.

3. **Telegram input queuing**: Messages sent to the bot are queued in the mailbox automatically.

4. **Additional profiles**: `profiles/code-review/`, `profiles/research/` for non-trading use cases.

5. **Grafana annotations**: Mark evaluation timestamps on Grafana dashboards via API.

6. **Response streaming**: Stream Claude's output to Telegram in real-time instead of waiting for completion.

7. **Context window management**: If system prompt + memory exceeds Claude's context, intelligently summarize/compress older memory.

8. **Web dashboard**: Simple local web UI to view task history, memory, and agent status.

---

## Architecture Decision: Why Claude Code CLI, Not API?

| Aspect | Claude Code CLI | Direct API |
|--------|----------------|------------|
| Tool ecosystem | Full (Bash, Read, Write, Edit, Grep, Glob) | Must reimplement |
| File operations | Native | Must implement |
| Shell commands | Native | Must implement |
| Auth management | Handled by CLI | Must manage API keys |
| Model selection | `--model claude-opus-4-6` | API parameter |
| Cost tracking | CLI handles it | Must track tokens |
| Complexity | ~50 LOC wrapper | ~500+ LOC client |
| Limitation | Subprocess overhead, no streaming | Full control |

**Decision:** CLI wrapper wins for our use case. The ~50 LOC in `claude.py` gives us the entire Claude Code tool ecosystem for free. We accept the subprocess overhead (seconds, not milliseconds) because our evaluation cycle runs every 6 hours, not every 6 milliseconds.

---

## Skills Migration Summary

| Old Skill | What Happens | Where It Goes |
|-----------|-------------|---------------|
| `trading-architect` | **Replaced** by goodoracle agent + trading profile | `profiles/trading/identity.md` + `evaluation.md` |
| `trading-guidance` | **Absorbed** into evaluation cycle | `profiles/trading/evaluation.md` recommendations section |
| `trading-hypotheses` | **Absorbed** into oracle memory + evaluation | Oracle reads/updates `HYPOTHESES.md` during evaluation |
| `trading-system-checkup` | **Absorbed** into evaluation health check | `profiles/trading/evaluation.md` health check section |
| **(new)** `goodoracle-dev` | **Bridge** between oracle and developer | `~/.claude/skills/goodoracle-dev/SKILL.md` |
