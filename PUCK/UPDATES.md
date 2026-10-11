# Software updates — how they work and how to publish one

Quantum Puck and Puck Pro devices update themselves from Quanverge Labs over **Wi-Fi**, with a
**notify + one-click install** model. Every update is cryptographically signed by Quanverge Labs; a device installs
nothing that is not signed with the Quanverge key, and returns to the previous version automatically if a new one
does not start.

## How a device updates

```
Quanverge website (primary) ──┐      manifest.json + manifest.json.sig + quanverge-puck-<v>.tar.gz
GitHub Releases (fallback) ───┤
                              ▼
 Puck API (every 6 h over Wi-Fi) ── verifies the Ed25519 signature with the public key on the device
        │                            └─ newer + compatible? → "Update <v> available" in Studio
        ▼  user clicks Install (blocked while jobs/notebooks run or battery < 30 % and not charging)
 download bundle → SHA-256 must match the signed manifest → write updates/request.json
        ▼  (systemd quanverge-updater.path)
 root updater ── re-verifies signature + checksum (trusts nothing from the API)
              ── extracts to /opt/quanverge-releases/<v>   (no path traversal, no links)
              ── builds the engine for this board, runs the engine self-test, installs Python packages
              ── atomically switches /opt/quanverge → <v>, restarts the services
              ── waits for /api/v1/health to report <v>   ── fails? → switch back + restart (rollback)
              ── keeps the current and previous release, writes updates/result.json
```

Experiments, notebooks, settings and the API token in `/var/lib/quanverge` and `/etc/quanverge` are never touched.
Without Wi-Fi (USB link only), the device does not check; updates install the next time it is on Wi-Fi.

## One-time setup (Quanverge side)

### 1. Signing key (done once — already generated on the development laptop)

```bash
./scripts/release/gen_keys.sh            # creates ~/.quanverge/release-keys/update-signing.key.pem (WSL home)
```

* **Private key** `~/.quanverge/release-keys/update-signing.key.pem` — signs releases. **Back it up now** to two safe
  places (password manager attachment + encrypted USB). Never commit it, email it or put it on a server.
  * Lost key → devices in the field can no longer be updated over the air (they need a manual re-install).
  * Leaked key → anyone could sign "updates". Generate a new pair and ship the new public key to all devices.
* **Public key** `device/keys/update-signing.pub.pem` — committed in the repo and installed on every device at
  `/etc/quanverge/update-signing.pub.pem`.

### 2. Hosting (both locations, as configured in `update_manifest_urls`)

| Location | URL the devices read |
|---|---|
| Website (primary) | `https://updates.quanvergelabs.com/puck/stable/manifest.json` (+ `.sig`, + bundle) |
| GitHub (fallback) | `https://github.com/quanverge-labs/quantum-puck-releases/releases/latest/download/manifest.json` |

* **Website:** create the subdomain `updates.quanvergelabs.com` with HTTPS, and a folder `/puck/stable/`. Any static
  hosting works (the same host as quanvergelabs.com, Cloudflare R2/Pages, S3…).
* **GitHub:** create a **public** repository `quanverge-labs/quantum-puck-releases` used only for release files (the
  source code repository can stay private). `releases/latest/download/<file>` always serves the newest release.

To use other URLs, change `update_manifest_urls` in `services/api/app/config.py` (default for new devices) or in
`/etc/quanverge/puck.toml` on a device.

## Publishing a release (every time)

On the development laptop (WSL terminal, in the project folder):

1. **Test** — `make -C engine test`, `pytest tests`, the Pro tests and both browser E2E tests must pass.
2. **Bump the version** in `VERSION` (semantic versioning: `0.1.0` → `0.1.1` for fixes, `0.2.0` for features).
3. **Build the UI** — `cd apps/studio && npm run build`.
4. **Commit and tag** — `git commit -am "Release 0.2.0" && git tag v0.2.0 && git push --tags`.
5. **Build and sign the release:**
   ```bash
   python3 scripts/release/make_release.py --notes @release-notes.md
   ```
   Output in `release/<version>/`: `quanverge-puck-<version>.tar.gz`, `manifest.json`, `manifest.json.sig`. The
   script verifies the signature with the device public key before finishing.
   Options: `--min-version 0.2.0` (devices older than this must be updated manually), `--base-url`, `--github-repo`.
6. **Upload to the website** — replace the three files in `/puck/stable/` (upload the `.tar.gz` first, the manifest
   files last, so devices never see a manifest pointing to a missing file).
7. **Upload to GitHub** — new release with tag `v<version>` in `quantum-puck-releases`, attach all three files,
   mark it "Latest release".
8. **Verify** — on a test Puck: Device → Software updates → *Check for updates* → *Install*. Confirm the version in
   the sidebar and Device page, and check `journalctl -u quanverge-updater -b`.

Release notes appear on the device exactly as written in `--notes`, so write them for customers.

## Testing updates

* `tests/integration/test_updates.py` covers signed/unsigned/tampered manifests, website→GitHub fallback, checksum
  mismatch, incompatible versions, low-battery blocking, simulated install, path-traversal bundles, rollback when the
  new version fails its health check, and pruning of old releases.
* On the laptop ("virtual Puck") an install runs the real updater in a sandbox (`var/.../update-sim`) without
  replacing the running code; Studio reports "laptop simulation".
* On a Pi: `sudo journalctl -u quanverge-updater -f` while installing; `ls -l /opt/quanverge` shows the active release.

## Manual recovery (support)

```bash
ls /opt/quanverge-releases                                   # installed versions
sudo ln -sfn /opt/quanverge-releases/<good-version> /opt/quanverge
sudo systemctl restart quanverge-api quanverge-power         # manual rollback
cat /var/lib/quanverge/updates/result.json                   # last update result
```
