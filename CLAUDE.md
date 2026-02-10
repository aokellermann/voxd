# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Voxd is a voice-to-text dictation application for Linux. It runs in the background, records audio via sounddevice, transcribes with whisper.cpp (local/offline), optionally post-processes text with LLMs (local or cloud), and types the result into the focused application via ydotool (Wayland) or xdotool (X11).

## Build & Development

**Build system:** Hatchling. Source layout with `src/voxd/` as the package.

```bash
# Install in editable mode (inside a venv)
pip install -e .

# Run the application
voxd              # main entry point (GUI by default)
voxd --cli        # CLI mode
voxd --tray       # system tray daemon
voxd --flux       # VAD-driven continuous dictation
voxd --trigger-record  # send IPC signal to running daemon

# Model management
voxd-model        # separate CLI for downloading whisper models
```

**Full installer:** `./setup.sh` handles system deps, venv creation, whisper.cpp build, ydotool setup, and desktop launchers. Idempotent and distro-aware (apt/dnf/pacman/zypper).

## Testing

```bash
pytest                    # run all tests
pytest tests/test_config.py  # run a single test file
pytest -k test_name       # run a specific test by name
```

Tests use a custom `isolate_xdg_dirs` fixture (in `tests/conftest.py`) that mocks XDG directories and sets Qt to headless mode, so tests never touch real user files.

## Architecture

### Processing Pipeline

```
AudioRecorder (sounddevice) → AudioPreprocessor (normalize/clip detect)
  → WhisperTranscriber (whisper-cli subprocess)
  → AIPP (optional LLM post-processing: llama.cpp, ollama, openai, anthropic, xai)
  → SimulatedTyper (ydotool/xdotool) + ClipboardManager (wl-copy/xclip)
```

The main orchestration loop lives in `src/voxd/core/voxd_core.py`.

### UI Modes

Four independent UI frontends share the same core pipeline:

- **GUI** (`gui/gui_main.py`) — PyQt6 frameless window with record/status display
- **CLI** (`cli/cli_main.py`) — interactive terminal with commands: r(ecord), rh(ands-free), l(og), cfg, x(exit), h(elp)
- **Tray** (`tray/tray_main.py`) — system tray icon, background daemon mode
- **Flux** (`flux/flux_main.py`) — Voice Activity Detection engine for continuous hands-free dictation, with tuner GUI (`flux_tuner.py`)

### IPC

A Unix socket server (`utils/ipc_server.py`) lets external hotkeys trigger recording on a running daemon via `voxd --trigger-record` (client in `utils/ipc_client.py`).

### Configuration

YAML-based config at `~/.config/voxd/config.yaml` with 100+ settings. Managed by `AppConfig` in `core/config.py`. Defaults ship in `src/voxd/defaults/config.yaml`.

### Path Layout (XDG-compliant)

- Config: `$XDG_CONFIG_HOME/voxd/`
- Models: `$XDG_DATA_HOME/voxd/models/`
- Output/logs: `$XDG_DATA_HOME/voxd/output/`

Path resolution is centralized in `src/voxd/paths.py`.

### External Binaries

- **whisper.cpp** (`whisper-cli`) — auto-built by `utils/whisper_auto.py` if missing
- **llama.cpp** — optional, for local LLM post-processing
- **ydotool** — required on Wayland for simulated typing

## Packaging

Uses **nfpm** (`packaging/nfpm.yaml`) to produce deb, rpm, and Arch packages for amd64 and arm64. CI workflow in `.github/workflows/release-packages.yml` builds packages on tag push via manual dispatch.

## Key Conventions

- Python 3.9+ compatibility required
- Entry points defined in `pyproject.toml`: `voxd` → `voxd.__main__:main`, `voxd-model` → `voxd.models:_cli`
- Version in `pyproject.toml` is a placeholder (`"mr.batman"`) — CI bumps it from the git tag during release builds
- 99+ languages supported via ISO 639-1 codes (`utils/languages.py`)
