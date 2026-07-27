# USB stack conflict — fix report and migration notes

Status: local notes, not for committing. Companion to `PR-draft.md`.
Written 2026-07-13.

## The failure

The nrf52840dongle and xiao_ble receiver builds fail to link with the
current SDK pin (NCS v3.3.0-preview3-branch, which carries Zephyr 4.3):

```
ld.bfd: usbd_cdc_acm.c:1381: multiple definition of `__device_dts_ord_117';
        cdc_acm.c:1229: first defined here
```

The firmware enables the legacy USB device stack
(`CONFIG_USB_DEVICE_STACK=y` in prj.conf). In Zephyr 4.3, these two
boards — and only these two of the boards built in CI — source
`boards/common/usb/Kconfig.cdc_acm_serial.defconfig`, which defaults
`USB_DEVICE_STACK_NEXT=y` for their CDC ACM console. With both stacks
enabled, the legacy `cdc_acm.c` and the new `usbd_cdc_acm.c` each
instantiate a driver for the same `zephyr,cdc-acm-uart` devicetree node,
and the link fails. The other boards get their CDC console from this
repo's overlays rather than board defaults, so they only ever have the
legacy stack.

CI timeline (verified against upstream run history):

- CI workflow added 2025-10-29 (2a3a259); oldest surviving run on main
  is 2025-12-08 (c7cd4d7). From then through 2026-01-04, **all** boards
  failed — a separate, broader breakage under the older SDK pin.
- 2026-01-08 (5a3961a, "ci: Bump to SDK v3.2 Branch"): from this run
  onward, exactly nrf52840dongle and xiao_ble fail — the USB stack
  conflict signature. This is the trigger commit.
- Failures are easy to miss because the build job sets
  `continue-on-error: true`. The failed jobs still surface as red X
  check runs on the commit, but the workflow run's own conclusion is
  "success", so the Actions run list stays green while only 7 of 9
  artifacts upload. Mixed signal, not full concealment.

## The immediate fix (the PR)

Set `CONFIG_USB_DEVICE_STACK_NEXT=n` in
`boards/nrf52840dongle_nrf52840.conf` and `boards/xiao_ble.conf`. The
board-level enable is a Kconfig `default`, not a `select`, so the app
config overrides it cleanly and both boards behave exactly as before the
SDK bump. Verified green on the fork:
https://github.com/biscuitvixen/SlimeVR-Tracker-nRF-Receiver/actions/runs/29236774651

## Why this is a stopgap

The legacy USB device stack was formally deprecated in Zephyr 4.3 — the
version the current pin ships — and is slated for **removal in Zephyr
4.5**. The firmware will be forced onto the new stack (`usb_device_next`
/ usbd API) within roughly two Zephyr releases.

The stack choice is app-wide, not per-board: `src/hid.c` is written
entirely against the legacy API, and the two stacks cannot coexist in
one image (the link collision above is exactly that). So dongle and xiao
cannot take the modern stack for their console while HID stays legacy —
it is all-legacy (this fix) or a whole-app migration.

## Migration assessment: legacy → usb_device_next

The USB surface of this firmware is small: one HID transport file, a
passive console, and configuration. The port is contained.

### src/hid.c (the only real work)

| Legacy (current) | New stack (usbd) |
|---|---|
| `usb_hid_register_device()` | `hid_device_register()` |
| `hid_int_ep_write()` | `hid_device_submit_report()` |
| `hid_int_ep_read()` polled on a timer | `output_report` callback (data is delivered, no polling) |
| `int_in_ready` hid_ops callback | `input_report_done` callback |
| `usb_enable(status_cb)` + `usb_dc_status_code` | `USBD_DEVICE_DEFINE` + `usbd_msg_cb` messages |
| `device_get_binding("HID_0")` | `zephyr,hid-device` devicetree node |
| `CONFIG_USB_DEVICE_VID/PID` | VID/PID passed to `USBD_DEVICE_DEFINE` (0x1209/0x7690) |
| `CONFIG_USB_HID_POLL_INTERVAL_MS=1` | `in-polling-period-us = <1000>` devicetree property |

The HID report descriptor and wire format are unchanged — the same byte
array is handed to the new API, so the SlimeVR server sees an identical
device. `CONFIG_USB_HID_BOOT_PROTOCOL` looks vestigial (the code sets
`HID_BOOT_IFACE_CODE_NONE`) and can likely be dropped; the new-stack
equivalent is the `protocol-code = "none"` devicetree property.

The whole mapping above was verified against the Zephyr v4.3.0 source
(`include/zephyr/usb/class/usbd_hid.h`, the `zephyr,hid-device` binding,
and the hid-keyboard sample), not just the docs. Details confirmed:

- `hid_device_submit_report()` is asynchronous when an
  `input_report_done` callback is provided, mirroring the current
  `int_in_ready` design; without the callback it blocks.
- The interrupt OUT endpoint (`CONFIG_ENABLE_HID_INT_OUT_EP=y` in
  prj.conf today) maps to declaring `out-report-size` in the
  `zephyr,hid-device` node. Without that property, output reports
  arrive via `set_report()` on the control pipe instead of the
  `output_report` callback.
- `in-polling-period-us` is a required binding property; there is also
  a runtime setter (`hid_device_set_in_polling()`, behind
  `CONFIG_USBD_HID_SET_POLLING_PERIOD`).
- `USBD_DEVICE_DEFINE(name, udc, vid, pid)` takes VID/PID directly;
  manufacturer/product strings become `USBD_DESC_MANUFACTURER_DEFINE` /
  `USBD_DESC_PRODUCT_DEFINE`.

Two gotchas found during verification, not obvious from the docs:

1. **Console CDC context ownership.** On the new stack, the board's CDC
   ACM console auto-initializes its own usbd device context
   (`CDC_ACM_SERIAL_INITIALIZE_AT_BOOT=y`, set by the same board
   fragment that causes the current conflict). Only one context can own
   the UDC, so an app-defined composite device (HID + CDC console) must
   disable that auto-init and register the console's CDC instance into
   its own context. Same spirit as today's
   `CONFIG_USB_DEVICE_INITIALIZE_AT_BOOT=n`, but it has to be handled
   deliberately, per board.
2. **Report buffer alignment.** `hid_device_submit_report()` requires
   an aligned report buffer (the sample uses `UDC_STATIC_BUF_DEFINE`).
   The current `ep_report_buffer` is an array of `__packed` 16-byte
   structs with no alignment guarantee, so the buffer handling in
   hid.c needs a small change. Callbacks also run in the USB stack
   thread and must not block (the current handlers already comply).

### Console / CDC — nearly free

`src/console.c` talks to whatever `zephyr,console` points at; the
existing `zephyr,cdc-acm-uart` overlay nodes work on the new stack via
`CONFIG_USBD_CDC_ACM_CLASS`. For dongle and xiao this is what the board
defaults already want to do. One wart: `console.c` prints
`CONFIG_USB_DEVICE_MANUFACTURER/PRODUCT`, which are legacy-only Kconfig
symbols set per-board in `boards/*.conf`; those strings need to move to
app-level defines or new-stack string descriptors.

### Config

The ~10 legacy `CONFIG_USB_*` options in prj.conf and the per-board
manufacturer/product strings get replaced by their usbd equivalents; the
two-line board fix from the PR gets deleted.

### Risk assessment

- Controller driver: not a risk. These boards' CDC console already runs
  on the new stack by default upstream, so `udc_nrf` is proven on this
  hardware.
- Needs hardware validation: the 1 ms interrupt polling behavior, the
  4-reports-per-64-byte-transfer batching under load, and
  latency/dropped-report behavior versus the current code. This is the
  part that cannot be signed off from CI alone — CI proves it links and
  boots, not that tracker data streams correctly at rate.
- Coordination: the SlimeVR tracker firmware repo shares the same
  legacy-stack pattern; upstream may want to migrate both together.
  Worth asking on the SlimeVR Discord whether sctanf already has plans.

### Effort estimate

Roughly a day of focused work (hid.c rewrite, overlay/Kconfig swap)
plus hardware testing across the board matrix.

## References

- The board Kconfig fragment that triggers the conflict (permalink at
  the v4.3.0 tag):
  https://github.com/zephyrproject-rtos/zephyr/blob/v4.3.0/boards/common/usb/Kconfig.cdc_acm_serial.defconfig
- Zephyr 4.3 release notes, exact wording: "The legacy stack is now
  deprecated and will be removed in Zephyr 4.5."
- Zephyr 4.3 release announcement (new stack replaces legacy):
  https://www.zephyrproject.org/zephyr-4-3-is-here-whats-new/
- Zephyr 4.3.0 release notes:
  https://docs.zephyrproject.org/latest/releases/release-notes-4.3.html
- Legacy USB device support (deprecated):
  https://docs.zephyrproject.org/latest/services/connectivity/usb/device/usb_device.html
- USBD HID device API:
  https://docs.zephyrproject.org/latest/doxygen/html/group__usbd__hid__device.html
- HID keyboard sample on the new stack:
  https://docs.zephyrproject.org/latest/samples/subsys/usb/hid-keyboard/README.html
- MCUboot migration issue confirming removal target of Zephyr 4.5:
  https://github.com/mcu-tools/mcuboot/issues/2596
- nRF Connect SDK v3.3.0 release notes:
  https://nrfconnectdocs.nordicsemi.com/ncs/3.3.0/nrf/releases_and_maturity/releases/release-notes-3.3.0.html
