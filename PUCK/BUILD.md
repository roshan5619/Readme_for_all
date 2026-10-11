# Build guide — Quantum Puck and Quantum Puck Pro

Step-by-step instructions to build both devices: parts, assembly, software installation, connecting a laptop,
and acceptance checks.

| | Quantum Puck | Quantum Puck Pro |
|---|---|---|
| Board | Raspberry Pi 4B (4 GB) | Raspberry Pi 5 (8 GB) |
| Battery | Waveshare UPS HAT (B) | Geekworm X1200 |
| Qubits | 8 | 24 |
| Extras | Quantum Studio | Studio + Notebook by Quanverge Labs (Python on the device) |
| New parts cost | ≈ $55–70 (Pi already owned) | ≈ $140–160 |

> **Status:** the software is fully tested on a laptop ("virtual Puck"). The hardware steps below are written
> from vendor documentation and have **not yet been run on physical boards**. Complete the checklist in
> [Part 4](#part-4--acceptance-checklist-both-devices) before relying on a device.

Related references: [hardware.md](hardware.md) (Puck details), [../quantum_puck_pro/docs/hardware-pi5.md](../quantum_puck_pro/docs/hardware-pi5.md)
(Pro details), [testing.md](testing.md), [benchmarks.md](benchmarks.md).

---

## Part 1 — Prepare the software on a laptop (both devices)

The Raspberry Pi has no Node.js, so it serves a prebuilt copy of the web app. Build it once on the laptop:

```bash
cd Quantum_Puck/apps/studio
npm install
npm run build          # produces apps/studio/dist (committed to the repo)
```

Optional — confirm everything passes before copying to the Pi (WSL / Linux / macOS):

```bash
make -C engine test                                        # C engine: 169 checks
~/.venvs/quanverge-puck/bin/python -m pytest tests          # base Puck
```

Copy the whole `Quantum_Puck` folder to the Pi (Git, USB stick or `scp`, see Part 2). The installer ignores
`node_modules`, `engine/build*` and `var`; the engine is always recompiled on the Pi.

---

## Part 2 — Quantum Puck (Raspberry Pi 4B, 8 qubits)

### 2.1 Parts

| Part | Purpose | Approx. cost |
|---|---|---|
| Raspberry Pi 4 Model B, 4 GB | The computer | (owned) |
| Waveshare UPS HAT (B) | Battery board: powers the Pi via the 5 V header pins, INA219 battery gauge on I2C 0x42 | $25 |
| 2 × protected 18650 Li-ion cells (same model/batch) | Battery, ~2–3 h | $12 |
| 8.4 V charger (normally included with the HAT) | Charging | incl. |
| PAM8302 mono amplifier + 8 Ω 0.5–1 W speaker | Eject voice and chimes | $5 |
| USB-C data cable + **VBUS-blocking** USB-C adapter | Laptop link without back-feeding power | $5 |
| 32 GB microSD card (A1/A2) | Operating system | $8 |
| Case with HAT clearance + heatsink | Enclosure, cooling | $5–10 |

**Tools:** small screwdriver, jumper wires (or soldering iron), USB-C power meter (strongly recommended),
multimeter.

### 2.2 Assembly

1. Attach the heatsink to the Pi's processor.
2. Press the **UPS HAT (B)** onto the 40-pin GPIO header.
3. Insert the two 18650 cells — **check polarity** — and switch the HAT on.
4. Wire the speaker amplifier:

   ```
   Pi 3.5 mm jack  Left   ──► PAM8302 A+
   Pi 3.5 mm jack  GND    ──► PAM8302 A−
   Pi pin 2  (5 V)        ──► PAM8302 VIN
   Pi pin 6  (GND)        ──► PAM8302 GND
   PAM8302 speaker output ──► 8 Ω speaker
   ```

5. Fit everything into the case.

### 2.3 Power rules (read before connecting a laptop)

- **Never connect an ordinary USB-C cable from a laptop to the Pi** while the HAT powers it. Use the
  **VBUS-blocking adapter** so only data passes; confirm with the USB power meter that ≈ 0 A flows from the laptop.
- The UPS HAT (B) charges **only from its own DC charger**, not from the laptop.
- The Pi 4 has no power button: switch the HAT on and wait for the **ready chime** (≈ 30–45 s).

### 2.4 Operating system

1. On the laptop, open **Raspberry Pi Imager** → *Raspberry Pi OS (64-bit)*, Bookworm.
2. In the settings (⚙) set: hostname `quanverge-puck`, user name and password, Wi-Fi, **enable SSH**.
3. Write the card, insert it into the Pi, power on.

### 2.5 Install the Quantum Puck software

Copy the project (example from Windows PowerShell):

```powershell
scp -r C:\path\to\Quantum_Puck <user>@quanverge-puck.local:~/
```

On the Pi (`ssh <user>@quanverge-puck.local`):

```bash
cd ~/Quantum_Puck
sudo ./device/scripts/install.sh --hostname quanverge-puck
sudo reboot                                           # first install only (USB gadget + I2C)
sudo /opt/quanverge/device/scripts/diagnose.sh --sound
```

The installer checks the OS, installs packages, compiles the engine for the Cortex-A72, runs the engine
self-test, creates the `quanverge` service user, generates the sounds (the spoken prompt with `espeak-ng`),
writes `/etc/quanverge/puck.toml`, creates a random API token in `/etc/quanverge/puck.env`, enables I2C and the
USB-C network link, and starts the services.

### 2.6 Connect from a laptop

| Link | Address | Token |
|---|---|---|
| USB-C cable | http://192.168.7.1 | not needed |
| Wi-Fi / Ethernet | http://quanverge-puck.local | required — `sudo cat /etc/quanverge/puck.env` on the Pi, paste on the **Device** page |

Open Studio automatically whenever the Puck is plugged in (install once per laptop):

```powershell
# Windows (needs Python 3 for Windows)
powershell -ExecutionPolicy Bypass -File host-launcher\install-windows.ps1
```

```bash
# macOS / Linux
./host-launcher/install-unix.sh
```

---

## Part 3 — Quantum Puck Pro (Raspberry Pi 5, 24 qubits, notebooks)

### 3.1 Parts

| Part | Purpose | Approx. cost |
|---|---|---|
| Raspberry Pi 5, **8 GB** | Notebooks, Qiskit, 24-qubit simulation (4 × Cortex-A76) | $80 |
| Official **Active Cooler** | Sustained multithreaded runs without throttling | $5 |
| Geekworm **X1200** UPS shield | 2 × 18650, 5.1 V / 5 A to the Pi 5, USB-C PD charging input, MAX17040 gauge (I2C 0x36), power-loss signal (GPIO 6) | $45 |
| 2 × protected 18650 cells (≥ 3000 mAh, same batch) | Battery | $12 |
| 27 W USB-C PD supply (official Pi 5 PSU) | Charges the X1200 | $12 |
| 64 GB A2 microSD card | OS + Qiskit, Aer, PennyLane | $10 |
| **USB audio adapter** + PAM8302 + 8 Ω speaker | Voice/chimes — the Pi 5 has **no 3.5 mm jack** | $8 |
| USB-C data cable + VBUS-blocking adapter | Laptop link | $5 |
| Case with X1200 + cooler clearance | Enclosure | $10–15 |

### 3.2 Assembly

1. Fit the **Active Cooler** to the Pi 5 and plug its cable into the **FAN** header.
2. Mount the Pi 5 on the **X1200** with the supplied standoffs; the pogo pins must touch the pads under the Pi.
3. Insert the 18650 cells — **check polarity**.
4. Connect the **PD charger to the X1200's USB-C port** (not to the Pi's).
5. Audio: plug the USB audio adapter into a Pi USB port; wire its output to the PAM8302 and speaker as in
   section 2.2 (adapter left/ground instead of the Pi jack; VIN/GND from Pi pins 2 and 6).
6. Laptop link: connect the Pi 5's own USB-C port through the **VBUS-blocking adapter** (the X1200 already powers
   the Pi). Pi 5 USB-gadget mode must be verified on your unit; until then use Wi-Fi.

### 3.3 Operating system

As in 2.4, with hostname `quanverge-puck-pro`.

### 3.4 Install the Pro software

Copy the project as in 2.5, then on the Pi:

```bash
cd ~/Quantum_Puck
sudo ./quantum_puck_pro/device/install-pro.sh --hostname quanverge-puck-pro
sudo reboot
sudo /opt/quanverge/device/scripts/diagnose.sh --sound
```

`install-pro.sh` runs the Part 2 installer first, then:

- builds the multithreaded engine for the Cortex-A76 (OpenMP, 28-qubit ceiling),
- creates the notebook environment in `/var/lib/quanverge/kernel-venv` (Qiskit, Qiskit Aer, PennyLane,
  matplotlib, ipykernel — downloading these takes several minutes),
- switches `/etc/quanverge/puck.toml` to `edition = "pro"` (24 qubits, larger limits),
- enables per-notebook memory/CPU limits (systemd user manager with linger),
- selects the X1200 battery driver and enables full USB current for the Pi 5.

### 3.5 Connect

| Link | Address |
|---|---|
| Wi-Fi / Ethernet | http://quanverge-puck-pro.local/#/notebook (token required) |
| USB-C (once gadget mode is verified) | http://192.168.7.1/#/notebook |

---

## Part 4 — Acceptance checklist (both devices)

Run on the real hardware and tick off:

- [ ] Installer finishes; `diagnose.sh` reports **no FAIL**
- [ ] Ready chime plays after boot
- [ ] Quantum Studio opens; a Bell circuit gives only `00` and `11`
- [ ] **Pro:** the Notebook page appears and all 8 notebooks in `Examples` run without errors
- [ ] **Pro:** `/opt/quanverge/engine/build-pro/qpuck info` shows `"threads":4`
- [ ] Battery % on the Device page matches a multimeter-based estimate within ±5 %
- [ ] Plugging/unplugging the charger changes the charging status
- [ ] Laptop link: the USB meter shows ≈ 0 A from the laptop; the laptop receives a `192.168.7.x` address
- [ ] The host launcher opens Studio within ~5 s of plugging in
- [ ] **Eject:** the Puck says "You are safe to remove now", powers off, and data is intact after the next boot
- [ ] Low-battery automatic shutdown triggers
- [ ] Wi-Fi access requires the token; USB access does not
- [ ] Benchmarks recorded in `benchmarks/results/` using
      [`benchmarks/REPORT_TEMPLATE.md`](../benchmarks/REPORT_TEMPLATE.md):
  - Puck: `python3 /opt/quanverge/benchmarks/run_bench.py`
  - Pro: `QPUCK_LIB=/opt/quanverge/engine/build-pro/libqpuck.so python3 /opt/quanverge/benchmarks/run_bench.py --qubits 8 16 20 22 24`

---

## Part 5 — Everyday operation

| Task | Command (on the Pi) |
|---|---|
| Service status | `systemctl status quanverge-api quanverge-power` |
| Live logs | `journalctl -u quanverge-api -f` |
| Restart the service | `sudo systemctl restart quanverge-api` |
| Show the API token | `sudo cat /etc/quanverge/puck.env` |
| Change limits (qubits, shots, timeouts) | edit `/etc/quanverge/puck.toml`, then restart the service |
| Update the software | Studio → Device → **Software updates** → Check / Install (over Wi-Fi; signed, automatic rollback). Manual: copy the new project folder and re-run the installer (data and token are kept) — see [UPDATES.md](UPDATES.md) |
| Back up experiments and notebooks | copy `/var/lib/quanverge/` |
| Full diagnostics | `sudo /opt/quanverge/device/scripts/diagnose.sh --sound` |

**Troubleshooting**

| Symptom | Check |
|---|---|
| No ready chime | `journalctl -u quanverge-api -b`; audio device in `aplay -l` |
| Laptop gets no 192.168.7.x address | `sudo /usr/local/sbin/quanverge-usb-gadget status`; try another cable/adapter |
| Battery shows "not fitted" | `i2cdetect -y 1` should list 0x42 (Puck) or 0x36 (Pro); I2C enabled in `raspi-config` |
| Throttling warnings | `vcgencmd get_throttled` should be `0x0`; check cooling and the power supply |
| Notebook kernel won't start (Pro) | `journalctl -u quanverge-api -e`; `/var/lib/quanverge/kernel-venv/bin/python -c "import ipykernel"` |

Only expose a Puck on networks you trust; never port-forward it to the internet.
