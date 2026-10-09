**Title:** Fix Bluetooth on MediaTek controllers (CFI-12xx, fw 13.60): scan, pair and stream work

Hi! On my PS5 (CFI-1216A, fw 13.60, Relapse) the **1.0.2 release doesn't start at all**: no toast, no page, no tile, same as #1. A build of the **current `main`** does start (page and tile work), so `main` already fixes that and a release would help the people in #1. After that, though, **Scan found nothing** and **Connect always failed with a page timeout (0x04)**. My console has a **MediaTek `0e8d:3603`** Bluetooth controller (HCI 5.2, manufacturer 0x0046), so newer FAT models have the MediaTek chip too, not only the Pro.

### Cause
The PS5 keeps its own page scan running almost all the time (interlaced, window `0x14e` every `0x29c`). On this chip that leaves almost no radio time for HearBridge to connect or scan.

| | Before | After |
|---|---|---|
| Connect | 12–64 s or timeout | ~1 s |
| Scan | 0–1 results | finds the buds |
| Pair | failed | works (0.6 s) |
| Stream | none | 48 kHz SBC, 0 drops |

Tested with raw HCI commands: with `Write_Scan_Enable(0)` a page takes ~1 s; with the system's value (`2`) it usually times out.

### Changes (3 commits)
1. **Pause the PS5's page scan while we connect, pair or scan.** `hci_scan_pause()` / `hci_scan_resume()` in `hci_cmd.c` turn it off and restore the old value right after, so the DualSense keeps working.
2. **Use the right bulk OUT pipe.** This chip has two (`0x01`, `0x02`). `0x01` stalls, so ACL now uses the one matching the bulk IN (`0x02`), with `0x01` as spare. New test in `tests/test_diag.c`.
3. **Optional HCI debug.** Create `/data/hearbridge/hci_debug` to enable `/api/hcilog` (HCI trace) and `/api/hci` (raw commands, token required). Off by default. Might help with the Pro (#2).

### Notes
- The PS5 still grabs ~2–5% of the HCI events (it reads the same endpoint). Your timed refill handles it, with no drops. I couldn't add more reads: fw 13.60 only allows 62 USB transfers (the 63rd `USB_FS_OPEN` fails with ENOMEM).
- `make test` all OK. `make ps5` with SDK v0.43 OK, no new imports.
- Version number not changed.
- Tested with SOUNDPEATS Air6 HS and oraimo SpaceBuds N.
