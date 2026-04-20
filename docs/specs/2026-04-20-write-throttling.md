# Write throttling for filter_monitor.py

## Background

The original code calls `_persist()` on every timer tick (default: every 60 seconds), unconditionally. On Raspberry Pi systems with SD cards that experience write stalls, this blocks Klipper's reactor and can freeze prints.

## Design

### Throttle

Add `PERSIST_INTERVAL = 300` at module level (5 minutes, in seconds). Not user-configurable — this is an implementation detail, not a tuning knob.

Add `self.last_persist_time = 0.0` in `__init__`.

`_persist(force=False)` skips the write unless `force=True` or `time.time() - self.last_persist_time >= PERSIST_INTERVAL`. Updates `last_persist_time` after a successful write.

**Rationale for wall-clock throttle:** The CSV stores `filter_last_reset`, `filter_runtime`, `filter_total_runtime`, and `filter_reset_count`. Calendar time is derived at runtime from `filter_last_reset`, so idle periods produce no new data to write. A simple wall-clock throttle is sufficient — no dirty-check or runtime-delta calculation needed. Losing up to 5 minutes of runtime data is acceptable against a 50-hour filter budget.

### Force persist wiring

Certain events must write immediately regardless of the throttle:

- **Shutdown and restart** — `_update(stop_timer=True)` is the terminal write path. Pass `force=True` to `_persist()` when `stop_timer=True`.
- **Reset** — a filter reset is a deliberate user action that must be durable. If Klipper crashes within the throttle window after a reset, the reset would be lost and the filter re-counted as unserviced.

`_update()` gains a `force_persist=False` parameter that passes through to `_persist()`. `_reset_filter()` calls `_update(force_persist=True)`. No changes to `cmd_RESET_FILTER`.

## Decisions log

**No concurrent update guard.** Klipper's reactor is single-threaded. The only re-entrancy path (`expiry_gcode` → `RESET_FILTER` → `_update()`) is harmless — the second `_monitor()` call adds ~0 runtime, `_notify()` does not fire (guarded by `event_time is not None`), and the throttle prevents a double write.

**No adaptive throttle.** A throttle expressed as a fraction of `max_runtime_hours` would auto-scale with filter lifetime, but adds complexity with no current use case.

**No `persist_interval` config option.** Users shouldn't need to reason about write frequency.
