# Cachy-Os
Bug fixes, diagnostic scripts, and CachyOS-specific configuration notes — root-caused.

---

# cachyos-fixes

A maintained log of CachyOS-specific bugs, package regressions, and system configuration work — root-caused and documented, not just patched and forgotten. Built while running CachyOS (Arch-based) with KDE Plasma/Wayland as a daily driver for security research, content creation, and gaming.

Every entry follows one rule: document the root cause and the fix that actually worked, including the approaches that looked correct but weren't. Most Arch/CachyOS troubleshooting guides online show only the final fix; this repo keeps the failed attempts too, because knowing what *doesn't* work saves as much time as knowing what does.

## Structure

```
cachyos-fixes/
├── README.md
├── TEMPLATE.md
├── issues/            # documented bugs: symptom → root cause → fix → prevention
├── scripts/            # standalone scripts for recurring CachyOS maintenance
└── configs/            # config snippets/dotfiles specific to this CachyOS setup
```

### `Issues`

Each file covers one bug: what broke, what was tried and ruled out, the exact root cause (confirmed by command output, not assumption), the fix, and how to prevent recurrence. Format defined in `TEMPLATE.md`.

Current categories these fall under:

- **Package/ABI mismatches** — soname bumps landing before dependent packages rebuild (e.g. `libjxl` vs `spectacle`)
- **Display/compositor** — KWin Wayland crashes, direct scanout issues, portal service failures
- **Audio** — PipeWire routing conflicts (`filter-chain` configs, virtual sink collisions with EasyEffects/JamesDSP under an existing VDO.Ninja setup)
- **Power management** — `systemd` service failures after updates, TDP/ryzenadj profile persistence
- **Gaming/performance** — DXVK/VKD3D config drift in Bottles, controller/mouse polling and DPI resets after driver updates

### `Scripts`

Working automation built for this specific machine and use case — not generic tutorials. Includes things like:

- `cachy-power.sh` — menu-driven CPU power management with ryzenadj TDP profiles and a live terminal dashboard, persisted via systemd
- `wifienv.sh` — WiFi pentesting environment setup/teardown for a Tenda RTL8192EU adapter (DKMS driver + aircrack-ng suite)
- `tekken_profile.sh` — latency profiling helper for Tekken 7 under Bottles

Each script gets a short header comment stating what it assumes about the system (hardware, kernel modules, installed packages) since these are not meant to run blind on an unrelated machine.

### `Configs`

Non-secret configuration files worth version-controlling: PipeWire `filter-chain` definitions, `pacman.conf` patterns (correct `IgnorePkg` placement, etc.), systemd unit overrides.

---

## After Update Fix

| # | Category | Symptom | Root cause | Fix |
|---|---|---|---|---|
| [001](issues/001-spectacle-libjxl-abi-mismatch.md) | Package/ABI | Print Screen / spectacle fails after `pacman -Syu` | `libjxl` soname bump (0.11→0.12) ahead of dependent package rebuild | Downgrade `libjxl` from Arch Archive, lock with `IgnorePkg` |

