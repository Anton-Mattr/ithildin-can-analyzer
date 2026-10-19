# macOS Port Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship ithildin on macOS (Apple silicon) as a DMG, with PEAK adapters via the MacCAN PCBUSB library and candleLight/gs_usb adapters via libusb.

**Architecture:** The application code already runs on macOS (522/523 tests pass locally on arm64). The port adds a `gs_usb` interface end to end (schema → prober → enrichment → bus open), makes the adapter prober probe a per-platform backend set (macOS: `pcan`, `gs_usb` only), fixes PCAN enrichment for the reduced config dicts python-can returns on Darwin, and adds a macOS packaging/CI lane (PyInstaller `.app` → DMG, later signed and notarized).

**Tech Stack:** python-can 4.6 (`pcan`, `gs_usb` interfaces), `gs_usb` 0.3.x + `pyusb` + libusb-1.0, MacCAN PCBUSB (`libPCBUSB.dylib`, user-installed), PySide6, PyInstaller ≥ 6.10, `hdiutil`, `codesign`/`notarytool`.

**Spec:** No separate spec document. The requirements are the user's decision of 2026-10-19 ("port to Mac; PEAK via MacCAN and gs_usb is good enough") plus the findings below.

## Findings this plan rests on (verified 2026-10-19 on this Mac)

- `uv run pytest`: 522 passed, 1 failed (`tests/test_transmit_controls.py::test_list_name_is_elided_to_keep_clear_of_the_badge`). The tabular font resolves to Menlo, and Menlo's advance widths are fractional (`QFontMetricsF` 171.875 vs `QFontMetrics` 172 for the test string).
- python-can `pcan` on Darwin loads `PCBUSB` via `find_library`. `/usr/local/lib/libPCBUSB.0.13.dylib` is installed here. On Darwin, `PcanBus._detect_available_configs()` returns only `{"interface", "channel", "supports_fd"}` for *available* channels. It omits `device_id`, `device_name` and `channel_condition`, so `_enrich_pcan` today collapses every adapter into a single "PCAN device" whose serial is `None`.
- python-can's `gs_usb` interface has **no** `_detect_available_configs`. Detection must call `gs_usb.gs_usb.GsUsb.scan()` itself. `GsUsbBus(channel=i)` opens scan index `i`.
- `GsUsbBus.__init__` calls `self.gs_usb.start()` with default flags. `GS_CAN_MODE_LISTEN_ONLY` exists in `gs_usb.constants`, but python-can exposes no kwarg for it.
- The resources tree contains no `app_icon.*`. The spec's icon logic is already optional.
- The repo is `tobias-hallberg/ithildin-can-analyzer` (not a Mattr repo). Work on `feature/macos-port` off `main`.

## Global Constraints

- Python ≥ 3.12. Release builds stay on 3.12 (the `cantact` extra caps the shipped build; see `release.yml`).
- macOS target: **arm64 only**, macOS 13+. Build on GitHub `macos-14` runners. Intel Macs are out of scope.
- macOS adapter set: exactly `pcan` and `gs_usb`. Windows/Linux probe sets are unchanged.
- PCBUSB is **not bundled**. Users install it from mac-can.com. The `.app` must find it at `/usr/local/lib`. libusb **is** bundled inside the `.app`.
- `gs_usb` is a new `SupportedInterface` value on every platform, so a project saved on a Mac loads elsewhere. It is probed only on macOS: on Linux the kernel exposes those adapters as socketcan.
- `gs_usb` is **not** listen-only capable in this plan (not added to `LISTEN_ONLY_INTERFACES`). The Create-bus dialog already locks the switch for such interfaces.
- No new comments beyond non-obvious *why*. `ruff check src tests` and `mypy src` (strict) stay clean.
- Commit message trailer: `Co-Authored-By: Claude Opus 5.5 (1M context) <noreply@anthropic.com>`.

## Review Focus

1. **PCBUSB not installed:** the user plugs in a PEAK adapter on a Mac without the library. Expected: no crash and no stall; the dock shows no PEAK devices and points the user to the install instructions. Pinned in Task 3 (`test_pcan_probe_without_pcbusb_returns_no_devices`).
2. **Two identical single-channel PEAK USB adapters on macOS.** Expected: two separate device rows, each with one channel (`PCAN_USBBUS1`, `PCAN_USBBUS2`), not one merged "PCAN device". Pinned in Task 3.
3. **gs_usb adapter unplugged between probe and Connect:** the scan index no longer exists. Expected: a `BusReader.error` banner/report, not a hang or traceback. Pinned in Task 1 (`test_gs_usb_open_missing_device_reports_error`).
4. **Project with `listen_only = true` on a gs_usb bus** (hand-edited `.itp`). Expected: validation rejects it with the same message as socketcan/ixxat, so it never opens a bus that ACKs while the user believes it is silent. Pinned in Task 1.
5. **libusb missing on the user's machine (no Homebrew):** the bundled dylib must be the one loaded. Expected: the gs_usb probe still finds devices. Pinned in Task 5's verification step, and run on a Mac without Homebrew in Task 7.

---

### Task 1: `gs_usb` interface in schema, dependencies and bus open

**Files:**
- Modify: `src/ithildin/workspace/schema.py:33-43` (add `"gs_usb"` to `SupportedInterface`; leave `LISTEN_ONLY_INTERFACES` unchanged)
- Modify: `src/ithildin/core/bus.py:149-166` (docstring of `_apply_listen_only_kwargs`: name `gs_usb` beside socketcan/ixxat as having no silent kwarg)
- Modify: `pyproject.toml`: add `"gs-usb>=0.3.1; sys_platform == 'darwin'"` to `dependencies`; add a mypy override `module = ["gs_usb.*", "usb.*"]`, `ignore_missing_imports = true`
- Modify: `packaging/ithildin.spec` hiddenimports: add `"can.interfaces.gs_usb"`, `"gs_usb"`, `"usb.backend.libusb1"`
- Test: `tests/test_workspace.py`, `tests/test_bus.py`

**Interfaces:**
- Produces: `SupportedInterface` includes `"gs_usb"`. A `BusConfig(interface="gs_usb", channel=<int scan index>, bitrate=…)` is valid and opens `can.Bus(interface="gs_usb", channel=i, bitrate=…)`.

- [ ] **Step 1: Write failing tests**

```python
# tests/test_workspace.py
def test_gs_usb_bus_config_round_trips() -> None:
    cfg = BusConfig(name="bench", interface="gs_usb", channel=0, bitrate=500_000)
    assert BusConfig.model_validate(cfg.model_dump()) == cfg

def test_gs_usb_rejects_listen_only() -> None:
    with pytest.raises(ValidationError):
        BusConfig(name="bench", interface="gs_usb", channel=0, bitrate=500_000, listen_only=True)

# tests/test_bus.py
def test_gs_usb_open_missing_device_reports_error(qtbot, monkeypatch) -> None:
    # monkeypatch can.Bus to raise can.CanInitializationError("Cannot find device 0. Devices found: 0")
    # start a BusController for the gs_usb config; assert its `error` signal fires (qtbot.waitSignal)
    # with a message containing "Cannot find device 0", and no unhandled exception.
```

Match the existing `test_bus.py` pattern for the error-path test (how other open failures are asserted). If an existing test already covers "open raises → error signal" generically, parametrize it over `"gs_usb"` instead of adding a new one.

- [ ] **Step 2:** `uv run pytest tests/test_workspace.py tests/test_bus.py -k gs_usb -v`. Expected: FAIL (the literal rejects `"gs_usb"`).
- [ ] **Step 3:** Make the schema, pyproject, spec and docstring changes listed above, then run `uv lock` and `uv sync`.
- [ ] **Step 4:** `uv run pytest -k gs_usb -v` → PASS. `uv run mypy src` and `uv run ruff check src tests` → clean.
- [ ] **Step 5:** Commit `Add gs_usb as a supported CAN interface`.

### Task 2: Per-platform probe set and gs_usb detection/enrichment

**Files:**
- Modify: `src/ithildin/ui/adapter_prober.py:41-49, 149-160, 259-275`
- Modify: `src/ithildin/ui/adapter_enrich.py` (vendor/transport maps at ~87-97, new enricher, `_ENRICHERS`)
- Test: `tests/test_adapter_prober.py`, `tests/test_adapter_enrich.py`

**Interfaces:**
- Consumes: `"gs_usb"` in `SupportedInterface` (Task 1).
- Produces:
  - `_MACOS_BACKENDS: tuple[str, ...] = ("pcan", "gs_usb")` and `def supported_backends(platform: str = sys.platform) -> tuple[str, ...]`, which returns `_MACOS_BACKENDS` for `"darwin"` and `_SUPPORTED_BACKENDS` otherwise. `AdapterProber.run` iterates `supported_backends()`.
  - `def _detect_gs_usb() -> list[dict[str, object]]`: lazily imports `gs_usb.gs_usb.GsUsb` and returns `[{"interface": "gs_usb", "channel": i, "serial": <dev.serial_number or None>, "device_name": <product string or None>} for i, dev in enumerate(GsUsb.scan())]`. On `ImportError`, or the `usb.core.NoBackendError` that pyusb raises when libusb is absent, it returns `[]`. `_detect_configs` routes `backend == "gs_usb"` to it.
  - `_enrich_gs_usb`: one `DeviceInfo` per config, with `vendor="candleLight"`, `model=device_name or "gs_usb device"`, `serial=serial`, `transport="USB"`, `capabilities=("CAN 2.0",)` and one `ChannelInfo(channel=i, label="CAN", status="free")`. It is registered in `_ENRICHERS["gs_usb"]`.

- [ ] **Step 1: Write failing tests**

```python
def test_supported_backends_on_macos_is_pcan_and_gs_usb() -> None:
    assert supported_backends("darwin") == ("pcan", "gs_usb")

def test_supported_backends_elsewhere_unchanged() -> None:
    assert supported_backends("win32") == ("pcan", "kvaser", "cantact", "socketcan", "ixxat")
    assert supported_backends("linux") == supported_backends("win32")

def test_detect_gs_usb_lists_each_scanned_device(monkeypatch) -> None:
    # install a fake `gs_usb.gs_usb` module in sys.modules whose GsUsb.scan() returns
    # two objects with serial_number "A1" / "B2"
    assert [c["channel"] for c in _detect_configs("gs_usb")] == [0, 1]
    assert [c["serial"] for c in _detect_configs("gs_usb")] == ["A1", "B2"]

def test_detect_gs_usb_without_libusb_returns_empty(monkeypatch) -> None:
    # fake GsUsb.scan raises usb.core.NoBackendError (fake the class too)
    assert _detect_configs("gs_usb") == []

def test_enrich_gs_usb_one_device_per_adapter() -> None:
    devs = enrich("gs_usb", [{"interface": "gs_usb", "channel": 0, "serial": "A1", "device_name": "candleLight USB to CAN adapter"},
                             {"interface": "gs_usb", "channel": 1, "serial": "B2", "device_name": None}])
    assert [(d.serial, d.model, d.channels[0].channel) for d in devs] == [
        ("A1", "candleLight USB to CAN adapter", 0), ("B2", "gs_usb device", 1)]
```

- [ ] **Step 2:** `uv run pytest tests/test_adapter_prober.py tests/test_adapter_enrich.py -v` → the new tests FAIL.
- [ ] **Step 3:** Implement the interfaces above. Read the real attribute names from the installed `gs_usb` 0.3.1 `GsUsb` class (serial/product). If `serial_number` triggers a USB string read that can raise, catch it per device and use `None`.
- [ ] **Step 4:** The tests pass, and the whole `tests/test_adapter_prober.py` stays green (existing tests may assume `_SUPPORTED_BACKENDS` on darwin; patch `sys.platform` or `supported_backends` there rather than weakening them).
- [ ] **Step 5:** Commit `Probe pcan and gs_usb on macOS`.

### Task 3: PCAN enrichment for Darwin config dicts

**Files:**
- Modify: `src/ithildin/ui/adapter_enrich.py:168-212` (`_enrich_pcan`)
- Test: `tests/test_adapter_enrich.py`, `tests/test_adapter_prober.py`

**Interfaces:**
- Produces: when a pcan config has no `device_id`, it becomes its own device, grouped by `channel`. Existing grouping by `device_id` is unchanged when the key is present.

- [ ] **Step 1: Write failing tests**

```python
def test_pcan_darwin_configs_are_one_device_per_channel() -> None:
    devs = enrich("pcan", [{"interface": "pcan", "channel": "PCAN_USBBUS1", "supports_fd": False},
                           {"interface": "pcan", "channel": "PCAN_USBBUS2", "supports_fd": True}])
    assert [[c.channel for c in d.channels] for d in devs] == [["PCAN_USBBUS1"], ["PCAN_USBBUS2"]]
    assert devs[1].capabilities == ("CAN 2.0", "CAN FD")
    assert all(d.status == "available" for d in devs)

def test_pcan_probe_without_pcbusb_returns_no_devices(monkeypatch) -> None:
    # monkeypatch ctypes.util.find_library to return None for "PCBUSB"
    # (python-can's PCANBasic() then raises OSError and detection returns [])
    assert _detect_and_enrich("pcan") == []
```

- [ ] **Step 2:** Run → the first test FAILS (a single merged device); the second should already pass, which confirms review-focus item 1.
- [ ] **Step 3:** In `_enrich_pcan`, key each config by `cfg.get("device_id")`, or by `("channel", cfg.get("channel"))` when `device_id` is absent.
- [ ] **Step 4:** `uv run pytest tests/test_adapter_enrich.py -v` → PASS.
- [ ] **Step 5:** Commit `Show each macOS PCAN channel as its own device`.

### Task 4: Fractional font metrics in `fit_name_and_id`

**Files:**
- Modify: `src/ithildin/ui/transmit/list_delegate.py:327-339` (or the test, if debugging shows the test's arithmetic is the defect)
- Test: `tests/test_transmit_controls.py:130-140`

- [ ] **Step 1:** Use superpowers:systematic-debugging. Under pytest, record `fm.horizontalAdvance` and `QFontMetricsF.horizontalAdvance` for `full`, `ident` and `"MMM…"`, plus the `room` computed inside `fit_name_and_id`, and find the pixel where elision kicks in. (In a bare script with Menlo the widths are 172 vs 171.875 and the name does not elide, so the failing condition is pytest-specific: look at font size/DPI from `qapp` or the app default font.)
- [ ] **Step 2:** Decide where the bug is. If a name that visibly fits gets elided in the real Transmit list on macOS, fix `fit_name_and_id` (e.g. compute `room` with `QFontMetricsF` and `math.floor`, so rounding errors favour the name). If only the test's expectation is off by rounding, fix the test's `roomy` with `math.ceil` of the float widths.
- [ ] **Step 3:** `uv run pytest -q` → 523 passed on macOS. Then break it on purpose: subtract 1 px from `roomy` and confirm the test fails, then restore it.
- [ ] **Step 4:** Commit.

### Task 5: macOS app bundle, DMG script and CI lanes

**Files:**
- Modify: `packaging/ithildin.spec`: on `sys.platform == "darwin"` add a `BUNDLE(coll, name="Ithildin.app", bundle_identifier="se.ithildin.analyzer", info_plist={"CFBundleShortVersionString": <version from pyproject>, "NSHighResolutionCapable": True, "LSMinimumSystemVersion": "13.0"})`. Use `.icns` if `RESOURCES / "app_icon.icns"` exists, else no icon. Ensure libusb is collected: rely on pyinstaller-hooks-contrib's `usb` hook, and if `find dist -name 'libusb-1.0*.dylib'` comes back empty, add the dylib to `binaries` explicitly.
- Create: `packaging/macos/build-dmg.sh` (mirrors `packaging/linux/build-appimage.sh`): reads the version from `pyproject.toml`, runs `hdiutil create -volname Ithildin -srcfolder dist/Ithildin.app -ov -format UDZO dist/Ithildin-<version>-arm64.dmg`, and adds an `/Applications` symlink to the staging folder.
- Modify: `.github/workflows/test.yml` matrix `os`: add `macos-14`.
- Modify: `.github/workflows/release.yml`: add a `macos` job (`runs-on: macos-14`, `UV_PYTHON: "3.12"`, `uv sync --group packaging --extra cantact`, PyInstaller, `bash packaging/macos/build-dmg.sh`, upload `dist/Ithildin-*-arm64.dmg`). Add it to the `release` job's `needs` and `files`.
- Modify: `packaging/ithildin.spec` header comment: say the spec is shared by Windows, Linux and macOS.

**Interfaces:**
- Produces: `dist/Ithildin.app` and `dist/Ithildin-<version>-arm64.dmg` (unsigned).

- [ ] **Step 1:** `uv run pyinstaller packaging/ithildin.spec --noconfirm --clean` → `dist/Ithildin.app` exists.
- [ ] **Step 2:** `find dist/Ithildin.app -name 'libusb-1.0*.dylib'` lists a bundled dylib. `open dist/Ithildin.app` launches the main window (no console traceback in `Console.app`).
- [ ] **Step 3:** Prove the bundled libusb is the one used (review-focus item 5): run `DYLD_PRINT_LIBRARIES=1 dist/Ithildin.app/Contents/MacOS/ithildin 2>&1 | grep libusb` with the dock probing, and check that the path is inside the `.app`.
- [ ] **Step 4:** Prove PCBUSB loads from `/usr/local/lib` inside the frozen app: a connected PEAK adapter appears in the dock. Without an adapter, temporarily rename `/usr/local/lib/libPCBUSB.dylib` and confirm the probe completes with no PEAK rows and no hang; then restore it.
- [ ] **Step 5:** `bash packaging/macos/build-dmg.sh` → the DMG mounts and shows `Ithildin.app` + `Applications`.
- [ ] **Step 6:** Push the branch and check that `tests` passes on `macos-14` for 3.12 and 3.13. Trigger `release` via `workflow_dispatch` and check that the macOS artifact is uploaded.
- [ ] **Step 7:** Commit `Build a macOS app bundle and DMG`.

### Task 6: Code signing and notarization *(blocked on an Apple Developer ID)*

Do not start this task until the owner confirms an Apple Developer Program membership and has added the secrets.

**Files:**
- Create: `packaging/macos/entitlements.plist` with `com.apple.security.cs.disable-library-validation = true` (required so the hardened-runtime app can `dlopen` the separately signed `/usr/local/lib/libPCBUSB.dylib`) and `com.apple.security.cs.allow-unsigned-executable-memory = true` (PyInstaller/ctypes).
- Modify: `packaging/ithildin.spec`: pass `codesign_identity` and `entitlements_file` from the env vars `ITHILDIN_CODESIGN_IDENTITY` / `ITHILDIN_ENTITLEMENTS`, falling back to `None`.
- Modify: `packaging/macos/build-dmg.sh`: when `ITHILDIN_CODESIGN_IDENTITY` is set, sign the DMG, run `xcrun notarytool submit --wait`, then `xcrun stapler staple`.
- Modify: `release.yml` macOS job: import the `.p12` from the secrets `MACOS_CERT_P12` / `MACOS_CERT_PASSWORD` into a temporary keychain, and pass `APPLE_ID`, `APPLE_TEAM_ID` and `APPLE_APP_PASSWORD` to notarytool.

- [ ] **Step 1:** Build signed locally → `codesign --verify --deep --strict dist/Ithildin.app` passes and `spctl -a -vv dist/Ithildin.app` says `accepted, source=Notarized Developer ID` (after notarization).
- [ ] **Step 2:** On the signed build, a PEAK adapter still enumerates. Then remove the `disable-library-validation` entitlement, rebuild, and confirm the PEAK probe now fails. This proves the entitlement is load-bearing. Restore it.
- [ ] **Step 3:** Commit `Sign and notarize the macOS build`.

### Task 7: Docs and hardware verification

**Files:**
- Modify: `README.md:78` (remove macOS from the non-goals; add a macOS install section: DMG, install PCBUSB from https://www.mac-can.com, supported adapters PEAK PCAN-USB family and candleLight/gs_usb, gs_usb has no listen-only, and the unsigned-build Gatekeeper workaround if Task 6 has not landed)
- Modify: `CLAUDE.md` (drop "Not macOS in v1"; in the vendor-backends invariant, document `supported_backends()` and the macOS set, and update the file-locations row "Add or re-enable a CAN backend" to mention `supported_backends`)
- Modify: `CHANGELOG.md` (Unreleased: macOS support)

- [ ] **Step 1:** Write the docs changes.
- [ ] **Step 2:** Run the hardware checklist on the built `.app` and record the results in the PR description:
  - PEAK PCAN-USB: probe → connect at 500 kbit/s → frames in Trace, decoded in Monitor/Plot. Transmit one frame from the Transmit window and confirm it on a second node.
  - PEAK listen-only: connect with listen-only on, and confirm on the bus (second node or scope) that the PEAK no longer ACKs. If PCBUSB ignores `PCAN_LISTEN_ONLY`, **stop and report**, because this is safety-relevant (it would mean removing `pcan` from listen-only on darwin).
  - gs_usb (candleLight): probe → connect → receive and transmit, as above. Confirm the Create-bus dialog shows listen-only locked.
  - Unplug each adapter mid-capture → the error is reported in the banner/Report dock, and the app stays responsive.
  - UI smoke: menu bar (Settings/About land in the app menu), ⌘ shortcuts, MDI maximize/restore chrome, light/dark theme, plot rendering (`_gl_plot.py`) on Retina.
  - The DMG on a second Mac without Homebrew: the app launches and gs_usb enumerates.
- [ ] **Step 3:** Commit, open the PR from `feature/macos-port` → `main`.

---

**Effort estimate:** Tasks 1–5 take about 3–4 days. Task 6 takes about 1 day once the Apple account exists. Task 7 takes about 1–2 days, mostly hardware time. Total: about 1–1.5 weeks.
