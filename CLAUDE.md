# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Plover is a free, open-source stenography engine that translates stenotype input into text output. It supports multiple steno machines (keyboard, TX Bolt, Gemini PR, etc.) and runs on Linux, macOS, and Windows. Python 3.10+ required.

## Development Setup

```bash
python -m venv .venv
source .venv/bin/activate
pip install -c reqs/constraints.txt -r reqs/dev.txt
pre-commit install
```

CMake is required to build hidapi (for Plover HID support).

## Common Commands

### Testing
```bash
tox                              # run full test suite (default)
tox -e test                      # same as above, explicit
tox -e test -- test/test_steno.py          # run a single test file
tox -e test -- test/test_steno.py -k "test_name"  # run a single test
tox -e test -- test/gui_qt       # run GUI tests
```

The tox dev environment lives in `.tox/dev/` and can be activated directly.

### Linting
```bash
pre-commit run --all-files       # run ruff linter + formatter
ruff check --fix .               # lint with auto-fix
ruff format .                    # format code
```

Linter: Ruff (v0.14.8), configured via `.pre-commit-config.yaml`.

### Running from Source
```bash
tox -e launch -- -l debug        # launch Plover with debug logging
```

### Building Distributions
```bash
tox -e setup -- bdist_appimage   # Linux AppImage
tox -e setup -- bdist_app        # macOS app bundle
tox -e setup -- bdist_dmg        # macOS disk image
tox -e setup -- bdist_win        # Windows portable
```

## Architecture

Plover's engine processes steno input through a four-phase pipeline:

1. **Capture** (`plover/machine/`): Machine plugins receive hardware input and map physical keys to logical steno keys via a Keymap. Platform-specific keyboard capture lives in `plover/machine/keyboard_capture/`.

2. **Translation** (`plover/translation.py`): The `Translator` maintains a 100-stroke buffer and performs greedy longest-match lookup against a priority-ordered dictionary stack. Macros (e.g., undo, retro operations) execute here.

3. **Formatting** (`plover/formatting.py`): The `Formatter` processes meta operators (`{...}` syntax) and generates `Action` objects that describe output changes (text, backspaces, key combos) and state changes (capitalization, spacing).

4. **Rendering** (`plover/output/`): Platform-specific output handlers emit keystrokes to the OS. OS abstraction lives in `plover/oslayer/` with subdirectories for linux, osx, and windows.

### Key Components

- **Engine** (`plover/engine.py`): Central orchestrator connecting all four phases. Manages configuration, dictionary loading, machine state, and a hooks system for external listeners. Thread-safe.
- **Registry** (`plover/registry.py`): Plugin discovery via Python entry points. 10 extension point groups defined in `setup.cfg` under `[options.entry_points]`.
- **Steno** (`plover/steno.py`): `Stroke` data model built on `plover_stroke` C extension.
- **Dictionary system** (`plover/dictionary/`, `plover/steno_dictionary.py`): Supports JSON and RTF/CRE formats. Dictionary stack with enable/disable and priority ordering.
- **Configuration** (`plover/config.py`): Hierarchical config with sections for machine, system, dictionaries, appearance, logging, output.
- **GUI** (`plover/gui_qt/`): PySide6-based Qt interface, decoupled from engine logic.

### Plugin System

Plugins are discovered via setuptools entry points. The groups (`setup.cfg`):
- `plover.command`, `plover.dictionary`, `plover.gui`, `plover.machine`
- `plover.macro`, `plover.meta`, `plover.system`
- `plover.gui.qt.tool`, `plover.gui.qt.machine_option`

Platform-specific variants use `plover.{linux,osx,windows}.*` naming.

## Code Style

- PEP 8 for new code (project is not yet fully PEP 8 compliant)
- Prefer classes; decouple UI from logic
- Keep whitespace-only changes in separate commits from substantive changes

## PR Workflow

PRs must include a towncrier changelog fragment in `news.d/`:
- Named `<section>/<pr_number>.<category>.md`
- Sections: `feature`, `bugfix`, `api`
- Categories: `core`, `dict`, `ui`, `linux`, `osx`, `windows`, `break`, `dnr`, `new`
- Example: `news.d/bugfix/1041.ui.md`
