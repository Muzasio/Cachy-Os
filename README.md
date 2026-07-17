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


### `Scripts`

Working automation built for this specific machine and use case — not generic tutorials. 

Each script gets a short header comment stating what it assumes about the system (hardware, kernel modules, installed packages) since these are not meant to run blind on an unrelated machine.

### `Configs`

Non-secret configuration files worth version-controlling: PipeWire `filter-chain` definitions, `pacman.conf` patterns (correct `IgnorePkg` placement, etc.), systemd unit overrides.

---

## After Update Fix

| # | Fix | Symptom | Root Cause |
|---|---|---|---|
| 001 | [Spectacle Print Screen](After-Update-Fix/Spectacle-PrintScreen-fix.md) | Print Screen fails silently after `pacman -Syu` | `libjxl` soname bump (0.11→0.12) ahead of dependent package rebuild |
| 002 | [Tor Browser](After-Update-Fix/Tor%20Browser.md) | Silent audio, broken install, bookmark loss after system update | PipeWire socket permissions / install-method mismatch |

|---|---|---|---|---|
| [001](issues/001-spectacle-libjxl-abi-mismatch.md) | Package/ABI | Print Screen / spectacle fails after `pacman -Syu` | `libjxl` soname bump (0.11→0.12) ahead of dependent package rebuild | Downgrade `libjxl` from Arch Archive, lock with `IgnorePkg` |

