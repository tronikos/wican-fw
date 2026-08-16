# AGENTS.md — WiCAN firmware

Orientation for coding agents. `README.md` is the user-facing doc; this file is
the map plus the things that are easy to get wrong.

## Layout

| Path | What |
|---|---|
| `main/` | **all firmware C, flat** — ~48 files, no per-module subdirs |
| `components/` | 3 local components: `debug_logs/`, `filesystem/`, `ha_webhooks/` |
| `managed_components/` | IDF component-manager deps (littlefs, mdns) — generated |
| `vehicle_profiles/` | 40 per-manufacturer dirs of profile JSON (source of truth) |
| `.vehicle_profiles/` | Node tooling that merges those + `params.json` + `schema.json` |
| `vehicle_profiles.json` | **generated** merged artifact, committed |
| `docs/` | Nuxt docs site; `docs/content/0.Config/6.Automate/4.Supported_Parameters.md` is generated |
| `tools/wican-log-viewer/` | separate Rust CLI |

There is **no `vehicle_profiles.c`**. Profiles are pure data, and none of it is
compiled in:

* `vehicle_profiles.json` and `.vehicle_profiles/params.json` are **fetched over
  HTTPS from `meatpiHQ/wican-fw` `main`** by the web UI when the user picks a car
  (`main/homepage_full.html`). A profile added on a branch or a fork therefore has
  no effect on any device until it is merged upstream, and there is no point
  carrying one on a firmware branch.
* A profile the user uploads in the Automate tab is stored as `car_data.json` on
  LittleFS; standard PIDs live in `auto_pid.json`. `load_all_pids()`
  (`main/autopid.c`) reads those at boot.

Note the builtin database format has no `period` field, so a car picked from the
dropdown polls every parameter at the web UI's 5 s default. Only an uploaded
profile can set per-parameter poll intervals.

### The files that matter

- **`main/elm327.c`** (~1260 lines) — ELM327 command emulation and the CAN
  request/response loop (`elm327_request`). `elm327.h` is a tiny public API.
- **`main/autopid.c`** (~2740 lines) — profile loading, the polling task,
  response parsing (`parse_elm327_response`), expression evaluation, MQTT/HA
  publishing. Much the largest and least defensive file in the tree.
- **`main/comm_server.c`** — the TCP/UDP server. This is the path Car Scanner
  and other ELM327 clients use over WiFi.
- **`main/main.c`** — task wiring and `send_to_host()`, the single funnel from
  every protocol handler onto the TX queue.
- **`main/expression_parser.c`** — the profile expression language.

## Build

ESP-IDF, `idf.py build`. No unit tests exist anywhere in the repo
(`.vehicle_profiles/package.json` has `"test": "... exit 1"`), and **no workflow
builds the firmware** — the 4 workflows in `.github/workflows/` only build and
validate profile JSON, deploy docs, and build the Rust log viewer. C changes are
verified by building and flashing, nothing else.

- Hardware variant is selected in the root `CMakeLists.txt` (`set(HARDWARE_VER
  ${WICAN_V300})` today; V210 / USB_V100 / PRO are commented out).
- `git describe` is embedded as `-DGIT_SHA`, and the artifact name strips every
  `.` — `v4.21-47-gabc1234` becomes `wican-fw_obd_v421-47-gabc1234.bin`.
- **`sdkconfig` is checked in — never let it regenerate.** A regenerated one can
  silently drop `CONFIG_HTTPD_WS_SUPPORT=y` and break `config_server.c`.

### The Node tooling

`.vehicle_profiles/merge.js` (`npm run build`) does two things:

- `process_params()` (`src/params.js`) — sorts `params.json` in place,
  regenerates the `propertyNames` enum in `schema.json`, and writes
  `docs/.../4.Supported_Parameters.md`.
- `process_cars()` (`src/cars.js`) — merges `vehicle_profiles/**` into
  `vehicle_profiles.json`. `comment` keys are stripped (`PARAMS_TO_IGNORE`).

**Both use CWD-relative paths — run them from inside `.vehicle_profiles/`.**

`params.json` is the single source of truth for a parameter's unit / class /
min / max / description. A profile may only use names that exist there.

## Traps

### The odd-length PID trailing digit

In a profile, a `pid` string with an **odd** number of characters has its last
character consumed by `elm327_process_cmd`/`elm327_request` as the *expected
number of response frames*, not as hex. `22028C1` requests `22028C` and expects
1 frame. The hint is capped at 9; responses longer than that must carry no digit.

A **wrong** hint is expensive. In `elm327_request()` the `req_expected_rsp`
branch is checked *before* the ISO-TP-complete early exit, so if the hint is
never reached the request waits out the full `ATST` window.

`ATST` is parsed **base 16** (`elm327_set_timeout`), so `ATST96` is 0x96 = 150,
and the timeout is `150 * 4.096` = **614 ms** — not 393 ms. Several comments and
commit messages in the history got this wrong.

### `elm327_should_receive()` accepts a range when `ATCRA` is unset

With no receive filter it accepts **any** id in `0x7E8..0x7EF` (or
`0x18DAF100..0x18DAF1FF` extended), i.e. several ECUs answering one functional
`0x7DF` request. Anything that stops reading early must be gated on
`elm327_config.rx_address_is_set` or it will truncate multi-ECU responses.

### `send_to_host()` funnels through a 65-byte buffer

`xdev_buffer.ucElement` is `uint8_t[DEV_BUFFER_LENGTH]` = 65 (`main/types.h`),
and the buffer inside `send_to_host()` is `static` — shared across every
protocol handler. Callers pass up to 128 bytes. Anything queued for the host has
to respect that limit.

### `parse_elm327_response()` indexes raw, PCI bytes included

Profile expressions address the raw ELM327 text buffer with `ATH1` on, so `B0`
is the ISO-TP PCI byte and, on a multi-frame response, `B8` / `B16` / `B24` are
consecutive-frame PCI bytes sitting in the middle of the data. Profiles work
around this by skipping those indices (e.g. `((B5<<24)|(B6<<16)|(B7<<8)|B9)`).

This is the contract. **Do not add ISO-TP deframing** — it would silently break
every existing profile's byte offsets.

### `pid_init` is `;`-separated and becomes `\r`-separated

`load_all_pids()` rewrites `;` to `\r`; `send_commands()` then iterates on
`strchr(cmd_start, '\r')`. A `pid_init` whose final command has no terminator is
dropped — which is why many profiles in the tree end their `pid_init` with `;`.

### Out-of-range parameter values are ignored, not clamped

`autopid.c` logs "below min … ignoring" / "above max … ignoring" and drops the
sample. A profile that emits values outside its `params.json` min/max silently
produces no data at all, which looks identical to an unsupported PID.

### `OBD_ELM327` and `AUTO_PID` are mutually exclusive

`main.c` picks one, and `tcp_server_init()` is skipped entirely in `AUTO_PID`
mode. So `autopid_task` holding `elm327_lock()` across its whole sweep looks
alarming but cannot contend with a TCP client. Conversely the WiFi ELM327 path
takes **no** lock — `elm327_lock` is used only by `autopid.c`.

## Known rough edges

Unfixed on `main` at the time of writing; grep the symbol, line numbers drift.

`main/autopid.c`
- `autopid_find_standard_pid()` and `send_commands()` drain `autopidQueue`
  without freeing each response's `priority_data` heap buffer.
- `parse_elm327_response()`: `strncpy` into `char header_str[9]` before
  `header_length` is validated; unchecked `malloc`/`realloc` for
  `lowest_header_data` (and the `p = realloc(p, n)` leak-on-failure pattern);
  `response->data[k]` unbounded against `BUFFER_SIZE`; `priority_data[0]`
  logged without a NULL check.
- `parse_json_file()`: unchecked `ftell` (−1 → `buffer[-1] = 0`) and `malloc`.
- `load_all_pids()`: many `item ? strdup(item->valuestring) : …` without a
  `valuestring != NULL` check — a numeric JSON value crashes. Some sites already
  get this right, so grep before assuming.
- `standard_init` is `strdup`'d inside the per-PID loop and leaks each iteration.
- `strchr(name, '-') + 1` with no NULL check on names lacking a `-`.

`main/elm327.c`
- `elm327_lock()` uses `pdMS_TO_TICKS(portMAX_DELAY)`, which overflows in 32-bit
  to a finite ~71-minute timeout, and discards the return value — on expiry the
  caller proceeds unlocked.
- A 1-character command leaves `cmd` empty after the frame-count digit is
  stripped, transmitting a zero-length frame.
- `ATSP` stores an unvalidated protocol character.

`main/comm_server.c`
- `sock` is a bare `static int` closed while the RX task may be blocked in
  `recv()` on it — use-after-close across a reconnect.
- `xSemaphoreGive` sits outside the matching `if (xSemaphoreTake(...))` in
  places.
- `goto CLEAN_UP;` in `tcp_server_task` makes the code after it unreachable, and
  the global `char rx_buffer[128];` exists only for that dead code.

`main/CMakeLists.txt`
- `SRCS` lists `autopid.c` and `wc_timer.c` twice and contains a stray `""`.

## Conventions

- Tabs in `elm327.c`, `main.c`, `comm_server.c`; 4 spaces in `autopid.c`. Match
  the file you are in.
- `#define TAG __func__` in several files, so `ESP_LOGx` tags are function names.
- One concern per branch — the fix branches here are each intended to stand alone
  as an upstream PR.
