# roBa BLE pointer latency investigation

Date: 2026-06-25

## Repository and revisions

- Config repository branch: `fix/roba-ble-pointer-latency`
- User config manifest: `config/west.yml`
- ZMK project: `zmkfirmware/zmk`, revision `v0.3-branch`
- Checked ZMK commit: `acfd8e5ea76cf23ad1c9b6b99848f97a95224257`
- Zephyr from ZMK manifest: `zmkfirmware/zephyr`, revision `v3.5.0+zmk-fixes`
- PMW3610 driver: `kumamuk-git/zmk-pmw3610-driver`, revision `main`
- Checked PMW3610 commit: `5e04553ab803d24405bd45621a41310ea3050e59`

## Current roBa_R configuration

`boards/shields/roBa/Kconfig.defconfig` makes `roBa_R` the split central:

- `ZMK_SPLIT=y`
- `ZMK_SPLIT_ROLE_CENTRAL=y`

`boards/shields/roBa/roBa_R.overlay` enables the PMW3610 on the right half:

- `compatible = "pixart,pmw3610"`
- SPI0, CS `gpio0 9`, IRQ `gpio0 2`
- `trackball_listener` listens to `&trackball`
- `zip_temp_layer 4 30000` is applied as the input processor

Because the trackball device is on `roBa_R` and `roBa_R` is central, pointer input is generated on the central MCU and then sent to the host. It does not need to cross the split peripheral-to-central BLE link first.

Current `roBa_R.conf` pointer-related settings:

- `CONFIG_ZMK_POINTING=y`
- `CONFIG_ZMK_BLE=y`
- `CONFIG_SPI=y`
- `CONFIG_INPUT=y`
- `CONFIG_PMW3610=y`
- `CONFIG_PMW3610_CPI=800`
- `CONFIG_PMW3610_CPI_DIVIDOR=1`
- `CONFIG_PMW3610_ORIENTATION_180=y`
- `CONFIG_PMW3610_SCROLL_TICK=10`
- `CONFIG_PMW3610_INVERT_X=n`
- `CONFIG_PMW3610_INVERT_SCROLL_X=y`
- `CONFIG_PMW3610_RUN_DOWNSHIFT_TIME_MS=3264`
- `CONFIG_PMW3610_REST1_SAMPLE_TIME_MS=20`
- `CONFIG_PMW3610_POLLING_RATE_250=y`
- `CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS=30000`
- `CONFIG_PMW3610_MOVEMENT_THRESHOLD=2`
- `CONFIG_PMW3610_SMART_ALGORITHM=y`

## PMW3610 polling modes

The PMW3610 driver Kconfig defines a `choice` with exactly these symbols:

- `CONFIG_PMW3610_POLLING_RATE_250`
- `CONFIG_PMW3610_POLLING_RATE_125`
- `CONFIG_PMW3610_POLLING_RATE_125_SW`

The choice default is `PMW3610_POLLING_RATE_250`.

Implementation details from `src/pmw3610.h` and `src/pmw3610.c`:

- 250Hz and 125_SW both use sensor polling register value `0x0D`.
- Hardware 125Hz uses sensor polling register value `0x00`.
- 125_SW reads at the 250Hz sensor setting, stores one sample in `last_x/last_y`, and on the next sample adds the stored values to the current `x/y`.
- 125_SW then reports the summed movement once and clears the stored values.
- In MOVE/SNIPE mode the driver emits `INPUT_REL_X` with `sync=false`, then `INPUT_REL_Y` with `sync=true`.
- Therefore one PMW3610 motion handling pass becomes one ZMK mouse report containing both X and Y.
- At 250Hz the upper layer can receive up to about 250 synced pointer reports per second when motion is continuous.
- Scroll mode accumulates movement in `scroll_delta_x/y` and emits wheel events only after `CONFIG_PMW3610_SCROLL_TICK`.

## ZMK input to HID path

The current ZMK revision handles pointer events in `app/src/pointing/input_listener.c`.

- `INPUT_REL_X` and `INPUT_REL_Y` are accumulated in listener-local `data->mouse.data`.
- On an event with `evt->sync`, ZMK calls `zmk_hid_mouse_movement_set(x, y)`.
- It then calls `zmk_endpoints_send_mouse_report()`.
- After sending, it clears movement and scroll values back to zero.

`app/src/endpoints.c` chooses the transport:

- USB calls `zmk_usb_hid_send_mouse_report()`.
- BLE calls `zmk_hog_send_mouse_report(&mouse_report->body)`.

## BLE HID-over-GATT queue behavior

In `app/src/hog.c`:

- Mouse queue: `K_MSGQ_DEFINE(zmk_hog_mouse_msgq, sizeof(struct zmk_hid_mouse_report_body), CONFIG_ZMK_BLE_MOUSE_REPORT_QUEUE_SIZE, 4)`
- `CONFIG_ZMK_BLE_MOUSE_REPORT_QUEUE_SIZE` exists and defaults to `20`.
- `zmk_hog_send_mouse_report()` uses `k_msgq_put(..., K_MSEC(100))`.
- If the queue is still full after waiting, the code removes one old mouse report with `k_msgq_get(..., K_NO_WAIT)` and recursively retries the new report.
- The send worker drains queued reports with `k_msgq_get(..., K_NO_WAIT)` and calls `bt_gatt_notify_cb()`.
- The callback parameter does not install a completion callback in this revision, so this code does not record per-report BLE notification completion timing.
- There is no mouse-specific coalescing before the queue. Each synced ZMK mouse report becomes one queued BLE mouse report.

This queue behavior can create latency when reports are produced faster than the host/controller path consumes them. Increasing the queue size alone would risk keeping more old movement reports alive for longer, so it is not a first-line fix.

## BLE connection parameters

In ZMK `app/Kconfig` for this revision:

- `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN` exists.
- It changes connection stability-oriented defaults; notably `CONFIG_BT_CTLR_PHY_2M` defaults to `n` when this experimental option is enabled.
- It does not directly lower the HID mouse report queue delay or implement movement coalescing.

ZMK sets these BLE peripheral preferred defaults:

- `CONFIG_BT_PERIPHERAL_PREF_MIN_INT=6`
- `CONFIG_BT_PERIPHERAL_PREF_MAX_INT=12`
- `CONFIG_BT_PERIPHERAL_PREF_LATENCY=30`
- `CONFIG_BT_PERIPHERAL_PREF_TIMEOUT=400`

Zephyr `subsys/bluetooth/host/Kconfig.gatt` confirms:

- interval values are in 1.25ms units
- valid normal range starts at 6
- value 6 means 7.5ms
- latency is in connection intervals
- timeout is in 10ms units

Zephyr exposes these values through the GAP Peripheral Preferred Connection Parameters characteristic. The host may ignore or reject them; Windows is not guaranteed to adopt the requested interval.

## Low-risk change candidates

1. Add comparison build configs for PMW3610 polling modes.
   - Safe because the PMW3610 driver Kconfig choice confirms all three symbols exist.
   - Keep CPI, orientation, scroll, automouse, and smart algorithm unchanged.

2. Add a separate 250Hz + low-latency BLE comparison build.
   - Use only confirmed Zephyr/ZMK symbols:
     - `CONFIG_BT_PERIPHERAL_PREF_MIN_INT=6`
     - `CONFIG_BT_PERIPHERAL_PREF_MAX_INT=6`
     - `CONFIG_BT_PERIPHERAL_PREF_LATENCY=0`
   - Do not add `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN` for this test because it is not directly targeted at pointer latency and disables 2M PHY by default.
   - Do not change `CONFIG_ZMK_BLE_MOUSE_REPORT_QUEUE_SIZE` yet because queue behavior has no movement coalescing.

## Open questions and measurement gaps

- Actual Windows connection interval after pairing is not measured yet.
- Actual BLE notification scheduling delay is not measured because the current HOG path has no completion timestamp logging.
- No firmware build has been run locally in this workspace yet.
- If low-latency connection parameters do not improve behavior enough, the likely robust fix is a mouse movement coalescer before the BLE HOG mouse queue, preserving accumulated relative movement while avoiding long queues of old reports.

