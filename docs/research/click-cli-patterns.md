# Click CLI Patterns for Large Applications

Research findings for reorganizing `studyctl` (1272-line `cli.py`) and absorbing 12 Typer-based commands from `agent-session-tools`.

**Sources**: Click 8.3.x official docs (Context7), rich-click GitHub, current codebase analysis.

---

## Table of Contents

1. [Current State Analysis](#1-current-state-analysis)
2. [Multi-Module CLI Architecture](#2-multi-module-cli-architecture)
3. [Lazy Loading for Fast Startup](#3-lazy-loading-for-fast-startup)
4. [Plugin Pattern (File-Based Discovery)](#4-plugin-pattern-file-based-discovery)
5. [Typer-to-Click Conversion Guide](#5-typer-to-click-conversion-guide)
6. [Shared Options and Context](#6-shared-options-and-context)
7. [Sharing Logic Between CLI and FastAPI](#7-sharing-logic-between-cli-and-fastapi)
8. [Click + Rich Integration](#8-click--rich-integration)
9. [Error Handling Patterns](#9-error-handling-patterns)
10. [Testing Patterns](#10-testing-patterns)
11. [Recommended File Layout](#11-recommended-file-layout)
12. [Migration Plan](#12-migration-plan)
13. [Real-World Examples](#13-real-world-examples)

---

## 1. Current State Analysis

### studyctl CLI (Click-based, 1272 lines)

Current command tree:

```
studyctl
  sync          # top-level command
  status        # top-level command
  audio         # top-level command
  topics        # top-level command
  dedup         # top-level command
  review        # top-level command
  struggles     # top-level command
  wins          # top-level command
  streaks       # top-level command
  resume        # top-level command
  progress-map  # top-level command
  teachback     # top-level command
  teachback-history  # top-level command
  state         # group
    push / pull / status / init
  config        # group
    init / show
  schedule      # group
    install / remove / list / add / delete / blocks
  bridge        # group
    add / list
  docs          # group (implied from code)
    list / open / search
  web           # top-level command
  tui           # top-level command
```

**Problems**: All commands in one file. 15+ top-level imports loaded on every invocation. No logical grouping for related commands (teachback + teachback-history should be under a group).

### agent-session-tools (Typer-based, 6 separate entry points)

```
session-sync       # typer.Typer() in sync.py
session-query      # typer.Typer() in query_sessions.py (+ sub-apps: profiles, vscode)
session-export     # typer.Typer() in export_sessions.py
session-maint      # typer.Typer() in maintenance.py
tutor-checkpoint   # standalone function in tutor_checkpoint.py
study-speak        # typer.Typer() in speak.py
```

---

## 2. Multi-Module CLI Architecture

### Pattern: One Module Per Command Group

The standard Click pattern splits each command group into its own Python module, then registers them with the root group.

```
src/studyctl/
    cli/
        __init__.py      # root group + lazy registration
        _root.py         # top-level commands (sync, status, topics)
        _content.py      # sync, audio, dedup commands
        _state.py        # state push/pull/status/init
        _config.py       # config init/show
        _schedule.py     # schedule install/remove/list/add/delete/blocks
        _review.py       # review, struggles, wins, streaks, resume
        _progress.py     # teachback, teachback-history, bridge, progress-map
        _docs.py         # docs list/open/search
        _session.py      # absorbed from agent-session-tools
        _speak.py        # absorbed from agent-session-tools
```

### Root `__init__.py` — The Wiring Point

```python
"""studyctl CLI — the main entry point."""
from __future__ import annotations

import click

from ._lazy import LazyGroup


@click.group(
    cls=LazyGroup,
    lazy_subcommands={
        # Each value is "module_path.command_object"
        "sync":     "studyctl.cli._content.sync",
        "audio":    "studyctl.cli._content.audio",
        "dedup":    "studyctl.cli._content.dedup",
        "status":   "studyctl.cli._content.status",
        "topics":   "studyctl.cli._content.topics",
        "state":    "studyctl.cli._state.state_group",
        "config":   "studyctl.cli._config.config_group",
        "schedule": "studyctl.cli._schedule.schedule_group",
        "review":   "studyctl.cli._review.review",
        "struggles":"studyctl.cli._review.struggles",
        "wins":     "studyctl.cli._review.wins",
        "streaks":  "studyctl.cli._review.streaks",
        "resume":   "studyctl.cli._review.resume",
        "teachback":"studyctl.cli._progress.teachback_group",
        "bridge":   "studyctl.cli._progress.bridge_group",
        "progress-map": "studyctl.cli._progress.progress_map",
        "session":  "studyctl.cli._session.session_group",
        "speak":    "studyctl.cli._speak.speak",
        "docs":     "studyctl.cli._docs.docs_group",
        "web":      "studyctl.cli._web.web",
        "tui":      "studyctl.cli._tui.tui",
    },
)
@click.version_option()
def cli() -> None:
    """AuDHD study pipeline: sync, review, and track learning."""


# Entry point: studyctl = "studyctl.cli:cli"
```

### Individual Command Module Pattern

Each module is self-contained with its own imports:

```python
# src/studyctl/cli/_state.py
"""State sync commands — push/pull across machines."""
from __future__ import annotations

import click
from rich.console import Console

console = Console()


@click.group(name="state")
def state_group() -> None:
    """Cross-machine state sync (via Obsidian vault)."""


@state_group.command(name="push")
@click.argument("remote", required=False)
def state_push(remote: str | None) -> None:
    """Push local progress to remote machine(s)."""
    from ..shared import push_state  # <-- lazy import inside command

    try:
        pushed = push_state(remote)
    except FileNotFoundError as e:
        raise click.ClickException(f"{e}\nRun 'studyctl state init' first") from None
    if pushed:
        for f in pushed:
            console.print(f"[green]{f}[/green]")
    else:
        console.print("[dim]Everything up to date[/dim]")
```

**Key insight**: Business logic imports happen INSIDE command functions, not at module top level. This keeps `studyctl --help` fast because Python only imports the module for the command you actually run.

---

## 3. Lazy Loading for Fast Startup

### LazyGroup (from Click official docs)

This is the **recommended approach** from Click's own documentation:

```python
# src/studyctl/cli/_lazy.py
"""LazyGroup — defer subcommand imports until actually needed."""
from __future__ import annotations

import importlib

import click


class LazyGroup(click.Group):
    """A Click group that lazily loads subcommands on first access.

    Usage:
        @click.group(cls=LazyGroup, lazy_subcommands={
            "sub": "mypackage.commands.sub_module.sub_command",
        })
        def cli():
            pass

    The dict maps command names to dotted import paths.
    The last component is the attribute name on the module.
    """

    def __init__(
        self,
        *args: object,
        lazy_subcommands: dict[str, str] | None = None,
        **kwargs: object,
    ) -> None:
        super().__init__(*args, **kwargs)
        self.lazy_subcommands = lazy_subcommands or {}

    def list_commands(self, ctx: click.Context) -> list[str]:
        """Merge eagerly-registered and lazy command names."""
        base = super().list_commands(ctx)
        lazy = sorted(self.lazy_subcommands.keys())
        return base + lazy

    def get_command(self, ctx: click.Context, cmd_name: str) -> click.Command | None:
        if cmd_name in self.lazy_subcommands:
            return self._lazy_load(cmd_name)
        return super().get_command(ctx, cmd_name)

    def _lazy_load(self, cmd_name: str) -> click.Command:
        import_path = self.lazy_subcommands[cmd_name]
        modname, cmd_object_name = import_path.rsplit(".", 1)
        mod = importlib.import_module(modname)
        cmd_object = getattr(mod, cmd_object_name)
        if not isinstance(cmd_object, click.BaseCommand):
            raise ValueError(
                f"Lazy loading of {import_path!r} failed: "
                f"expected click.BaseCommand, got {type(cmd_object).__name__}"
            )
        return cmd_object
```

### What Triggers Lazy Loading

Per Click docs, lazy loading is triggered by:

1. **Command resolution** — `studyctl state push` loads only `_state.py`
2. **Help text rendering** — `studyctl --help` loads ALL modules to get docstrings
3. **Shell completion** — Tab completion loads modules to discover subcommands

**Mitigation for help**: Put the help string in `lazy_subcommands` dict or use a `LazyGroup` that stores help strings separately. For most CLIs, the `--help` cost is acceptable.

### Nested Lazy Groups

Sub-groups can also be lazy. For example, `studyctl session` can lazily load its own subcommands:

```python
# src/studyctl/cli/_session.py
@click.group(
    name="session",
    cls=LazyGroup,
    lazy_subcommands={
        "query":  "studyctl.cli._session_query.query",
        "export": "studyctl.cli._session_export.export_cmd",
        "sync":   "studyctl.cli._session_sync.sync_cmd",
        "maint":  "studyctl.cli._session_maint.maint_group",
    },
)
def session_group() -> None:
    """Manage agent session database."""
```

---

## 4. Plugin Pattern (File-Based Discovery)

For extensible CLIs where commands can be added by dropping files in a folder:

```python
import importlib.util
import os
import click


class PluginGroup(click.Group):
    """Discovers commands from .py files in a folder."""

    def __init__(self, name=None, plugin_folder="commands", **kwargs):
        super().__init__(name=name, **kwargs)
        self.plugin_folder = plugin_folder

    def list_commands(self, ctx):
        rv = []
        for filename in os.listdir(self.plugin_folder):
            if filename.endswith(".py") and not filename.startswith("_"):
                rv.append(filename[:-3])
        rv.sort()
        return rv

    def get_command(self, ctx, name):
        path = os.path.join(self.plugin_folder, f"{name}.py")
        if not os.path.isfile(path):
            return None
        spec = importlib.util.spec_from_file_location(name, path)
        module = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(module)
        return module.cli
```

**When to use**: When you want third-party or user-defined commands. For studyctl, the `LazyGroup` pattern is more appropriate since all commands are known at development time.

---

## 5. Typer-to-Click Conversion Guide

Typer is a thin wrapper around Click. Every Typer app compiles down to Click objects internally. The conversion is mostly mechanical.

### Mapping Table

| Typer Pattern | Click Equivalent |
|---|---|
| `app = typer.Typer()` | `@click.group()` |
| `@app.command()` | `@group.command()` |
| `app.add_typer(sub, name="x")` | `group.add_command(sub_group, "x")` |
| `typer.Argument(help="...")` | `@click.argument("name")` |
| `typer.Option("--flag", help="...")` | `@click.option("--flag", help="...")` |
| `Annotated[str, typer.Argument(...)]` | `@click.argument("name")` on function |
| `Annotated[str, typer.Option(...)]` | `@click.option(...)` on function |
| `typer.echo()` | `click.echo()` |
| `typer.Exit(code=1)` | `raise SystemExit(1)` or `ctx.exit(1)` |
| `rich_markup_mode="rich"` | Use `rich-click` or Rich Console directly |
| `add_completion=True` | Click has built-in shell completion |

### Concrete Conversion Example

**Before (Typer)**:

```python
import typer
from typing import Annotated

app = typer.Typer(name="session-sync", help="Sync sessions.db between machines.")

@app.command()
def push(
    remote: Annotated[str | None, typer.Argument(help="Target remote")] = None,
    dry_run: Annotated[bool, typer.Option("--dry-run", help="Show plan")] = False,
) -> None:
    """Push sessions to a remote machine."""
    ...

def main():
    app()
```

**After (Click)**:

```python
import click

@click.command()
@click.argument("remote", required=False)
@click.option("--dry-run", is_flag=True, help="Show plan")
def push(remote: str | None, dry_run: bool) -> None:
    """Push sessions to a remote machine."""
    ...
```

### Key Differences to Watch For

1. **Type annotations**: Typer uses `Annotated[type, metadata]` for params. Click uses decorators. Strip annotations, add decorators.

2. **Boolean flags**: Typer auto-detects `bool` params as flags. Click needs explicit `is_flag=True` or `--flag/--no-flag`.

3. **Enum choices**: Typer auto-converts `Enum` params. Click needs `type=click.Choice([...])`.

4. **Default from function signature**: Typer reads defaults from function signatures. Click decorators carry their own defaults (which override function defaults).

5. **Return values**: Neither framework uses return values (they go to stdout). No change needed.

6. **Sub-apps**: `app.add_typer(sub_app, name="x")` becomes `group.add_command(sub_group, "x")`.

### Strategy: Wrap Instead of Rewrite

Since Typer IS Click underneath, you can actually mix them. A Typer app exposes its Click group via the internal `typer.main.get_command()`:

```python
import typer
from typer.main import get_command

# This converts a Typer app to a Click Group
typer_app = typer.Typer(...)
click_group = get_command(typer_app)

# Now register it as a Click subcommand
cli.add_command(click_group, "session-sync")
```

**However**, this approach couples you to Typer's internals and makes the dependency mandatory. For studyctl, a clean conversion to pure Click is recommended because:
- studyctl already uses Click
- Typer adds an unnecessary dependency
- The conversion is straightforward and mechanical
- Pure Click gives you full control over lazy loading

---

## 6. Shared Options and Context

### Pattern 1: `click.pass_context` with `ctx.obj`

Store shared state in the Click context object:

```python
from dataclasses import dataclass, field
from pathlib import Path

@dataclass
class AppState:
    """Shared state passed through Click context."""
    verbose: bool = False
    config_path: Path | None = None
    console: Console = field(default_factory=Console)


@click.group(cls=LazyGroup, lazy_subcommands={...})
@click.option("--verbose", "-v", is_flag=True, help="Verbose output")
@click.option("--config", "config_path", type=click.Path(exists=True),
              envvar="STUDYCTL_CONFIG", help="Config file path")
@click.version_option()
@click.pass_context
def cli(ctx: click.Context, verbose: bool, config_path: str | None) -> None:
    """AuDHD study pipeline."""
    ctx.ensure_object(dict)
    ctx.obj = AppState(
        verbose=verbose,
        config_path=Path(config_path) if config_path else None,
    )


# Subcommands access it via @click.pass_obj
@state_group.command(name="push")
@click.argument("remote", required=False)
@click.pass_obj
def state_push(app: AppState, remote: str | None) -> None:
    if app.verbose:
        app.console.print("[dim]Verbose mode[/dim]")
    ...
```

### Pattern 2: Reusable Option Decorators

For options that appear on many commands:

```python
# src/studyctl/cli/_shared.py
import functools
import click


def common_options(f):
    """Decorator that adds --verbose and --config to any command."""
    @click.option("--verbose", "-v", is_flag=True, help="Verbose output")
    @click.option("--config", "config_path", type=click.Path(),
                  envvar="STUDYCTL_CONFIG")
    @functools.wraps(f)
    def wrapper(*args, **kwargs):
        return f(*args, **kwargs)
    return wrapper


def topic_argument(f):
    """Decorator that adds a TOPIC_NAME argument."""
    @click.argument("topic_name", required=False)
    @functools.wraps(f)
    def wrapper(*args, **kwargs):
        return f(*args, **kwargs)
    return wrapper
```

Usage:

```python
@cli.command()
@common_options
@topic_argument
def sync(topic_name: str | None, verbose: bool, config_path: str | None) -> None:
    ...
```

### Pattern 3: `ctx.with_resource` for Cleanup

For resources that need cleanup (database connections, temp files):

```python
@click.group()
@click.pass_context
def cli(ctx):
    ctx.obj = ctx.with_resource(DatabaseConnection())
    # Automatically cleaned up when CLI exits
```

---

## 7. Sharing Logic Between CLI and FastAPI

### The Service Layer Pattern

Business logic lives in service modules, NOT in CLI command functions or FastAPI route handlers. Both entry points call the same functions.

```
src/studyctl/
    services/             # Business logic — no Click or FastAPI imports
        sync_service.py
        review_service.py
        state_service.py
    cli/                  # Click commands — thin wrappers
        _content.py
        _review.py
    web/                  # FastAPI routes — thin wrappers
        server.py
        routes/
            sync.py
            review.py
```

**Service layer** (framework-agnostic):

```python
# src/studyctl/services/review_service.py
from dataclasses import dataclass

@dataclass
class ReviewResult:
    topic: str
    due: bool
    last_studied: str | None
    interval_days: int

def get_due_reviews(days_lookback: int = 30) -> list[ReviewResult]:
    """Pure business logic. No Click, no FastAPI, no Rich."""
    from ..history import spaced_repetition_due
    entries = spaced_repetition_due()
    return [ReviewResult(...) for entry in entries]
```

**CLI wrapper** (thin, handles display):

```python
# src/studyctl/cli/_review.py
@cli.command()
@click.option("--days", "-d", default=30)
def review(days: int) -> None:
    """Show topics due for spaced repetition review."""
    from ..services.review_service import get_due_reviews
    results = get_due_reviews(days)
    # Rich table rendering here
```

**FastAPI wrapper** (thin, handles serialization):

```python
# src/studyctl/web/routes/review.py
from fastapi import APIRouter
from ...services.review_service import get_due_reviews

router = APIRouter()

@router.get("/api/reviews/due")
async def due_reviews(days: int = 30):
    return get_due_reviews(days)
```

### Why This Matters

The current `cli.py` mixes business logic with display logic. Splitting them:
- Makes testing easier (test service without CLI overhead)
- Enables the FastAPI backend to reuse logic
- Keeps CLI commands short (10-20 lines each)

---

## 8. Click + Rich Integration

### Option A: `rich-click` (Drop-in Replacement)

The simplest approach. Replace `import click` with `import rich_click as click`:

```python
import rich_click as click

# Everything else stays the same
# Help pages are automatically styled with Rich
```

Add to dependencies:
```toml
dependencies = [
    "rich-click>=1.8",  # replaces both click and rich deps
]
```

### Option B: Use Rich Directly (Current Approach)

studyctl already uses `Rich.Console` and `Rich.Table`. This is fine and gives full control. The key patterns:

```python
from rich.console import Console
from rich.table import Table

console = Console()

# In commands:
console.print("[green]Success[/green]")
console.print("[red]Error[/red]", style="bold")

# Tables:
table = Table(title="Study Topics")
table.add_column("Name", style="cyan")
table.add_column("Status")
table.add_row("python", "[green]active[/green]")
console.print(table)
```

### Option C: Hybrid (Recommended)

Use `rich-click` for automatic help page styling, and `Rich.Console` for custom output in commands. They are fully compatible.

### Shared Console Instance

Create one console per CLI session, pass it through context:

```python
# src/studyctl/cli/_shared.py
from rich.console import Console

# Module-level singleton (fine for CLIs — single-threaded)
console = Console()
err_console = Console(stderr=True)
```

Import in command modules:

```python
from ._shared import console, err_console
```

---

## 9. Error Handling Patterns

### Click's Exception Hierarchy

```
BaseException
  +-- ClickException          (exit_code attribute, calls show())
  |     +-- UsageError        (malformed command invocation)
  |     +-- BadParameter      (invalid parameter value)
  |     +-- FileError         (file not found/not readable)
  |     +-- NoSuchOption
  |     +-- MissingParameter
  +-- Abort                   (prints "Aborted!" to stderr, exit 1)
```

### Pattern: Use ClickException Instead of SystemExit

The current code uses `raise SystemExit(1)` in several places. This works but bypasses Click's error formatting. Better:

**Before** (current):
```python
console.print("[red]Score must be 5 values[/red]")
raise SystemExit(1)
```

**After** (idiomatic Click):
```python
raise click.ClickException("Score must be 5 comma-separated values (accuracy,own_words,structure,depth,transfer)")
```

Or for parameter-specific errors:
```python
raise click.BadParameter(
    "must be 5 comma-separated integers (1-4)",
    param_hint="'--score'",
)
```

### Pattern: Consistent Error Handling Decorator

```python
# src/studyctl/cli/_shared.py
import functools
import click
from rich.console import Console

err_console = Console(stderr=True)

def handle_errors(f):
    """Catch common exceptions and convert to Click errors."""
    @functools.wraps(f)
    def wrapper(*args, **kwargs):
        try:
            return f(*args, **kwargs)
        except FileNotFoundError as e:
            raise click.ClickException(str(e)) from None
        except PermissionError as e:
            raise click.ClickException(f"Permission denied: {e}") from None
        except KeyboardInterrupt:
            raise click.Abort() from None
    return wrapper
```

Usage:
```python
@state_group.command(name="push")
@click.argument("remote", required=False)
@handle_errors
def state_push(remote: str | None) -> None:
    ...
```

### Exit Code Conventions

| Code | Meaning |
|------|---------|
| 0    | Success |
| 1    | General error (ClickException default) |
| 2    | Usage error (Click auto-handles) |
| 130  | KeyboardInterrupt (SIGINT) |

Set custom exit codes:
```python
raise click.ClickException("Not configured", exit_code=78)  # EX_CONFIG
```

---

## 10. Testing Patterns

### Basic CliRunner Setup

The current test setup is solid. Keep it:

```python
import pytest
from click.testing import CliRunner
from studyctl.cli import cli


@pytest.fixture
def runner() -> CliRunner:
    return CliRunner()


@pytest.fixture(autouse=True)
def _no_db(monkeypatch: pytest.MonkeyPatch) -> None:
    """Prevent tests from touching real database."""
    import studyctl.history as hist
    monkeypatch.setattr(hist, "_find_db", lambda: None)
```

### Testing Subcommands and Groups

```python
class TestStateCommands:
    def test_state_push(self, runner):
        result = runner.invoke(cli, ["state", "push"])
        assert result.exit_code == 0

    def test_state_push_with_remote(self, runner):
        result = runner.invoke(cli, ["state", "push", "macbook"])
        assert result.exit_code == 0

    def test_state_status(self, runner):
        result = runner.invoke(cli, ["state", "status"])
        assert result.exit_code == 0
```

### Testing with Context / Shared State

```python
def test_verbose_mode(runner):
    result = runner.invoke(cli, ["--verbose", "topics"])
    assert result.exit_code == 0

def test_config_override(runner, tmp_path):
    config = tmp_path / "test-config.yaml"
    config.write_text("topics: []")
    result = runner.invoke(cli, ["--config", str(config), "topics"])
    assert result.exit_code == 0
```

### Testing Error Conditions

```python
def test_bad_score_format(runner):
    result = runner.invoke(cli, [
        "teachback", "concept", "-t", "topic",
        "--score", "1,2,3",  # only 3 values, needs 5
        "--type", "micro",
    ])
    assert result.exit_code != 0
    assert "5 comma-separated" in result.output or "5 comma-separated" in (result.exception and str(result.exception) or "")
```

### File System Isolation

```python
def test_config_init_creates_file(runner):
    with runner.isolated_filesystem():
        result = runner.invoke(cli, ["config", "init"], input="y\n")
        assert result.exit_code == 0
```

### Catch Exceptions for Debugging

```python
def test_command_doesnt_crash(runner):
    result = runner.invoke(cli, ["sync", "--dry-run"], catch_exceptions=False)
    # catch_exceptions=False lets pytest show the real traceback
    assert result.exit_code == 0
```

### Testing Absorbed Typer Commands

After conversion, test them exactly like any Click command:

```python
class TestSessionCommands:
    def test_session_query_help(self, runner):
        result = runner.invoke(cli, ["session", "query", "--help"])
        assert result.exit_code == 0
        assert "Query" in result.output

    def test_session_sync_dry_run(self, runner):
        result = runner.invoke(cli, ["session", "sync", "--dry-run"])
        assert result.exit_code == 0
```

---

## 11. Recommended File Layout

```
packages/studyctl/src/studyctl/
    cli/
        __init__.py          # Root group with LazyGroup, entry point
        _lazy.py             # LazyGroup implementation (40 lines)
        _shared.py           # console, err_console, common decorators
        _content.py          # sync, audio, dedup, status, topics (~200 lines)
        _state.py            # state push/pull/status/init (~80 lines)
        _config.py           # config init/show (~120 lines)
        _schedule.py         # schedule install/remove/list/add/delete/blocks (~120 lines)
        _review.py           # review, struggles, wins, streaks, resume (~200 lines)
        _progress.py         # teachback, teachback-history, bridge, progress-map (~250 lines)
        _docs.py             # docs list/open/search (~80 lines)
        _session.py          # absorbed: query, export, sync, maint (~50 lines, lazy sub-group)
        _session_query.py    # absorbed from agent-session-tools
        _session_export.py   # absorbed from agent-session-tools
        _session_sync.py     # absorbed from agent-session-tools
        _session_maint.py    # absorbed from agent-session-tools
        _speak.py            # absorbed: study-speak
        _checkpoint.py       # absorbed: tutor-checkpoint
        _web.py              # web server launch (~30 lines)
        _tui.py              # TUI launch (~30 lines)
    services/                # Business logic (future refactor)
        review_service.py
        sync_service.py
        ...
    history.py               # Existing, unchanged
    config.py                # Existing, unchanged
    ...
```

### Why Underscore Prefix on CLI Modules?

Convention from Click community: `_module.py` signals "internal, not for direct import by users." Only `__init__.py` is the public API. This prevents accidental `from studyctl.cli._state import ...` in non-CLI code.

### Entry Point (unchanged)

```toml
[project.scripts]
studyctl = "studyctl.cli:cli"
```

---

## 12. Migration Plan

### Phase 1: Extract LazyGroup (No Behavior Change)

1. Create `src/studyctl/cli/` package
2. Add `_lazy.py` with `LazyGroup`
3. Move all existing code to `_legacy.py` temporarily
4. Wire `__init__.py` to import from `_legacy.py`
5. Run all tests -- must pass unchanged

### Phase 2: Split by Group (One Group at a Time)

For each group, in this order:
1. `_state.py` (smallest, 4 commands, already a group)
2. `_config.py` (2 commands, already a group)
3. `_schedule.py` (6 commands, already a group)
4. `_progress.py` (bridge group + teachback commands)
5. `_review.py` (review + struggles + wins + streaks + resume)
6. `_content.py` (sync + audio + dedup + status + topics)
7. `_docs.py`, `_web.py`, `_tui.py`

Each extraction:
- Move functions to new module
- Update `__init__.py` lazy_subcommands
- Run tests
- Verify `studyctl --help` still works

### Phase 3: Convert Typer Commands

For each Typer entry point:
1. Create corresponding `_session_*.py` Click module
2. Convert `Annotated[type, typer.Option(...)]` to `@click.option(...)`
3. Convert `typer.Typer()` groups to `@click.group()`
4. Remove `typer` dependency from converted code
5. Create `_session.py` as a lazy sub-group
6. Add tests using `runner.invoke(cli, ["session", "subcommand", ...])`

### Phase 4: Deprecate Standalone Entry Points

Keep old entry points working temporarily:

```python
# In agent-session-tools entry points (deprecated)
def main():
    import warnings
    warnings.warn(
        "session-sync is deprecated. Use 'studyctl session sync' instead.",
        DeprecationWarning, stacklevel=2,
    )
    from studyctl.cli._session_sync import sync_cmd
    sync_cmd()
```

### Phase 5: Service Layer (Future)

Extract business logic from CLI commands into `services/` modules. This enables:
- FastAPI route reuse
- Easier unit testing
- Cleaner separation of concerns

---

## 13. Real-World Examples

### Large Click CLIs Worth Studying

| Project | Commands | Pattern Used |
|---------|----------|-------------|
| **[HTTPie](https://github.com/httpie/cli)** | ~20 | Custom Group + plugin system |
| **[Conda](https://github.com/conda/conda)** | ~25 | Plugin-based with conda plugins |
| **[dbt-core](https://github.com/dbt-labs/dbt-core)** | ~15 | Click groups + multi-file |
| **[Celery](https://github.com/celery/celery)** | ~15 | Click multi-command with lazy loading |
| **[Flask CLI](https://github.com/pallets/flask)** | ~8 | Click AppGroup (like LazyGroup) |
| **[Prefect](https://github.com/PrefectHQ/prefect)** | ~40 | Deep nested Click groups |
| **[Typer itself](https://github.com/tiangolo/typer)** | N/A | Built ON Click Groups internally |

### Prefect's Pattern (40+ Commands)

Prefect organizes commands exactly like the recommended pattern above:

```
prefect/
    cli/
        __init__.py     # Root group
        _types/         # Shared types
        agent.py        # `prefect agent` subgroup
        block.py        # `prefect block` subgroup
        cloud.py        # `prefect cloud` subgroup
        deploy.py       # `prefect deploy` command
        flow.py         # `prefect flow` subgroup
        work_pool.py    # `prefect work-pool` subgroup
        ...
```

### Flask's AppGroup Pattern

Flask extends `click.Group` with `AppGroup` which auto-wraps commands. Similar concept to `LazyGroup` but for plugin discovery:

```python
from flask import Flask
from flask.cli import AppGroup

user_cli = AppGroup("user")

@user_cli.command("create")
@click.argument("name")
def create_user(name):
    ...

app.cli.add_command(user_cli)
# Usage: flask user create admin
```

---

## Summary: Recommendations for studyctl

1. **Use LazyGroup** for the root CLI group and any heavy sub-groups (session)
2. **One file per command group**, underscore-prefixed, max ~250 lines each
3. **Convert Typer to Click** mechanically (it is mostly decorator swaps)
4. **Move business logic imports inside command functions** for startup speed
5. **Use `click.ClickException`** instead of `SystemExit(1)` for consistent error display
6. **Consider `rich-click`** as a drop-in for prettier help pages (optional)
7. **Keep the existing test pattern** (CliRunner + monkeypatch), extend for new groups
8. **Plan for a service layer** to share logic between CLI and FastAPI
9. **Migrate incrementally** -- one group at a time, tests passing at every step
