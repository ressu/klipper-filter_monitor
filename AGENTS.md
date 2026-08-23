# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Gemini CLI, and others) when working with code in this repository.

## Project Overview

This is a Klipper 3D printer firmware plugin (`filter_monitor.py`) that monitors air filter runtime and expiry. It is a fork of [blankhaus/klipper-filter_monitor](https://github.com/blankhaus/klipper-filter_monitor). The plugin tracks how long a fan has been active and triggers notifications/G-code when runtime or age thresholds are exceeded.

## Development Commands

```bash
# Lint the plugin
uv run pylint filter_monitor.py

# Type-check the plugin (uses stubs/ for Klipper type definitions)
uv run pyrefly check filter_monitor.py
```

Both commands must exit cleanly with no errors. Type errors are never acceptable — the stubs exist specifically to validate the plugin against Klipper internals.

There is no build step. The plugin runs inside Klipper's Python environment on the printer.

## Architecture

**Single-file plugin**: All logic lives in `filter_monitor.py`. Klipper discovers it via `load_config_prefix()` at the bottom of the file, which maps `[filter_monitor <name>]` config sections to `FilterMonitor` instances.

**Klipper integration pattern**:
- `FilterMonitor.__init__` registers event handlers (`klippy:connect`, `klippy:ready`, `klippy:shutdown`, etc.) and G-code mux commands (`FILTER_STATS`, `RESET_FILTER`)
- `_handle_connect` resolves the fan object from config; fan lookup differs by type — `heater_generic` uses `heaters.lookup_heater()`, all others use `printer.lookup_object()`
- `_handle_ready` starts a recurring reactor timer (`_monitor_event`) that calls `_update()` every `interval` seconds
- `_monitor()` checks `fan.last_pwm_value` for heaters or `fan.get_status()["speed"]` for fan types, accumulates runtime, and computes `filter_percent_r` as the minimum of runtime-based and age-based remaining capacity

**State persistence**: Filter state (last reset time, runtime, total runtime, reset count) is written as a single CSV row to `~/printer_data/config/plugins/filter_monitor/<name>.csv` on every `_update()` call.

**Type stubs**: `stubs/klippy/__init__.pyi` and `stubs/extras/__init__.pyi` provide type definitions for Klipper internals (generated via stubgen). These are only used for static analysis — Klipper is not imported at type-check time. The `TYPE_CHECKING` guard at the top of `filter_monitor.py` handles this.

**Multi-instance support**: `FILTER_STATS` and `RESET_FILTER` are mux commands keyed by `NAME`. When called without `NAME` and only one instance exists, it targets that instance automatically.

## Documentation Conventions

- Design docs for non-trivial changes go in `docs/specs/` and are committed.
- Implementation plans are transient working notes, not committed to the repo.

## Klipper Plugin Conventions

- `load_config_prefix(config)` (not `load_config`) is used so multiple `[filter_monitor X]` sections can coexist
- Errors must be raised via `self.printer.command_error(msg)` — not Python exceptions — to integrate with Klipper's error system
- Fan speed is checked as `status["speed"]` (float 0.0–1.0) for fan types; heaters use `fan.last_pwm_value` directly
- Reactor timers return the next wake time from their callback; returning `self.reactor.NEVER` stops the timer
