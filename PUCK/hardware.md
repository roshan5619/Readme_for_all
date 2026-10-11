# Hardware build and Raspberry Pi setup

> **Status:** the software has been developed and tested on a laptop (WSL Ubuntu). The Raspberry Pi steps below are
> written for Raspberry Pi 4 Model B 4 GB + Raspberry Pi OS 64-bit (Bookworm) but **have not yet been run on the
> physical hardware**. Treat Pi compatibility as unverified until the checklist at the end passes.

## Bill of materials (low-budget build)

| Part | Purpose | Approx. cost |
|---|---|---|
| Raspberry Pi 4 Model B, 4 GB (owned) | Runs the simulator, API and Studio | – |
| Waveshare UPS HAT (B) | 2×18650 battery pack, 5 V to the Pi via GPIO, INA219 fuel gauge (I2C 0x42) | ~$25 |
| 2 × protected 18650 Li-ion cells (same model/batch) | Battery, ~2–3 h at typical load | ~$12 |
| Charger for the HAT (8.4 V, usually included) | Recharging the pack | (included) |
| PAM8302 mono amplifier + 8 Ω 0.5–1 W speaker | "You are safe to remove now" voice, ready/low-battery chimes | ~$5 |
| USB-C data cable + **VBUS-blocking** USB-C adapter | Data link to the laptop without back-feeding power (see R1) | ~$5 |
| 32 GB microSD (A1/A2) | OS + software | ~$8 |
| Enclosure (3D-printed or off-the-shelf with HAT clearance) + heatsink | Puck housing, passive cooling | ~$5–10 |
| **Total new spend** | | **≈ $55–70** |

8 qubits need only 4 KiB of state, so any Pi 4 (even 2 GB) is sufficient for the MVP; the 4 GB model leaves room
for the 20-qubit roadmap (16 MiB state).

## Wiring

```
              ┌───────────── Waveshare UPS HAT (B) ─────────────┐
 charger ───▶ │ DC in (8.4 V)   2×18650   INA219 (I2C 0x42)      │
              └───────┬──────────────────────────────┬──────────┘
                      │ 40-pin header: 5 V, GND,     │
                      │ SDA1 (GPIO2), SCL1 (GPIO3)   │
              ┌───────▼──────────────────────────────▼──────────┐
              │                Raspberry Pi 4B                  │
 laptop ◀───▶ │ USB-C port (data only, via VBUS blocker)        │
              │ 3.5 mm jack ── L/GND ──▶ PAM8302 IN+/IN−        │
              │ 5 V (pin 2/4), GND (pin 6) ──▶ PAM8302 VIN/GND  │──▶ speaker
              └─────────────────────────────────────────────────┘
```

* Stack the UPS HAT on the GPIO header (it powers the Pi through the 5 V pins and exposes I2C).
* PAM8302: `A+` to the jack's left channel, `A−` to jack ground, `VIN`/`GND` to Pi 5 V/GND, speaker on the output.
  (A small powered 3.5 mm speaker also works.)
* Do **not** connect a charger to the Pi's USB-C port while the HAT is powering it.

### Power and charging — important caveats

* **R1 — back-feeding.** The Pi 4's USB-C port is also its power input; the HAT feeds the same 5 V rail through the
  GPIO header. Plugging an ordinary USB-C cable into a laptop connects the laptop's VBUS to that rail. Use a
  USB-C adapter/cable with VBUS removed for the data link, and verify with a USB power meter before first use.
  Gadget mode on the Pi 4 is expected to enumerate without VBUS sensing, but **this must be confirmed on your
  hardware** (`diagnose.sh` shows the UDC state).
* **R2 — charging.** The UPS HAT (B) charges from its own DC input, *not* from the laptop's USB port. Plugging into
  the laptop therefore does not charge the Puck. If laptop-powered charging is required, the upgrade path is a HAT
  with USB-C PD input (e.g. Waveshare UPS HAT (E)), which would need a new battery driver.
* The Pi 4 has no power button: switch the HAT on, wait for the **ready chime** (~30–45 s boot), then plug in.
* Low battery: below 15 % the Puck plays a warning; below 8 % for three readings it ejects and powers off by itself.

## Software installation on the Pi

1. Flash **Raspberry Pi OS (64-bit), Bookworm** with Raspberry Pi Imager. In the Imager settings set a hostname
   (e.g. `quanverge-puck`), user, Wi-Fi and enable SSH.
2. Build the Studio UI on your laptop if `apps/studio/dist/` is missing (`cd apps/studio && npm install && npm run build`).
   The Pi never needs Node.js.
3. Copy the repository to the Pi (`git clone`, or `scp -r Quantum_Puck pi@quanverge-puck.local:`).
4. Install:
   ```bash
   cd Quantum_Puck
   sudo ./device/scripts/install.sh --hostname quanverge-puck
   sudo reboot      # first install only: enables USB gadget mode and I2C
   ```
   The installer checks the architecture/OS/Python, installs packages, builds the C engine with Cortex-A72 tuning,
   runs the engine self-test, creates the `quanverge` service user, generates sounds (the spoken prompt via
   `espeak-ng`), writes `/etc/quanverge/puck.toml` and a random API token in `/etc/quanverge/puck.env`, enables
   I2C, configures the USB gadget + DHCP, installs the polkit power-off rule and the systemd services.
5. Verify: `sudo /opt/quanverge/device/scripts/diagnose.sh --sound`.

## Connecting from the laptop

| Link | Address | Auth |
|---|---|---|
| USB-C (gadget network) | `http://192.168.7.1` | not required (trusted link) |
| Wi-Fi / Ethernet | `http://quanverge-puck.local` (mDNS) or the Pi's IP | API token required |

Install the launcher once per laptop so Studio opens automatically when the Puck is plugged in:

```powershell
# Windows (needs Python 3 for Windows)
powershell -ExecutionPolicy Bypass -File host-launcher\install-windows.ps1
```
```bash
# macOS / Linux
./host-launcher/install-unix.sh
```

Windows and macOS do not allow USB devices to auto-run software; the launcher is the supported way to get
"plug in → app opens". Windows binds the RNDIS function automatically through the Microsoft OS descriptors;
macOS and Linux use CDC-ECM. The Puck hands out `192.168.7.2–20` by DHCP and advertises no gateway or DNS, so the
laptop keeps its normal internet connection.

Only expose the Puck on networks you trust. Never port-forward it to the internet.

## Operations

| Task | Command |
|---|---|
| Status | `systemctl status quanverge-api quanverge-power quanverge-gadget` |
| Logs | `journalctl -u quanverge-api -f` |
| Restart after a crash | automatic (`Restart=on-failure`); manually: `sudo systemctl restart quanverge-api` |
| API token | `sudo cat /etc/quanverge/puck.env` (rotate: edit, then restart the service) |
| Change limits | edit `/etc/quanverge/puck.toml`, `sudo systemctl restart quanverge-api` |
| Re-run USB gadget | `sudo systemctl restart quanverge-gadget dnsmasq` |
| Update software | pull the repo, `sudo ./device/scripts/install.sh` (keeps data and token) |
| Backup data | copy `/var/lib/quanverge/puck.sqlite3` (stop the service or use `sqlite3 .backup`) |

**Recovery:** if the Puck lost power without ejecting, SQLite's WAL journal restores the database on next start and
any interrupted job is marked `INTERRUPTED`. If the service keeps failing, `diagnose.sh` and
`journalctl -u quanverge-api -b` identify the cause; reinstalling is idempotent.

## Hardware acceptance checklist (to be run on the Pi)

- [ ] `install.sh` completes on Pi 4B 4 GB / Bookworm 64-bit
- [ ] `diagnose.sh` reports no FAIL
- [ ] `pytest tests/` passes on the Pi (`pip install -r requirements-dev.txt`; PennyLane/Qiskit optional)
- [ ] USB-C link enumerates on Windows (RNDIS) and macOS/Linux (ECM); laptop gets 192.168.7.x
- [ ] No back-feeding through the data cable (USB meter shows ~0 A from the laptop)
- [ ] Launcher opens Studio within ~5 s of plugging in
- [ ] Battery % and charging state match a multimeter within ±5 %
- [ ] Eject: voice prompt from the Puck speaker, then clean power-off; data present after reboot
- [ ] Low-battery automatic shutdown triggers
- [ ] `benchmarks/run_bench.py` results recorded in `benchmarks/results/`
- [ ] Wi-Fi access requires the token; USB access does not
