# Spectacle: Print Screen Fails Silently After System Update

Print Screen (KDE Spectacle) stops working after `pacman -Syu` due to shared library ABI breaks in a chain: `libjxl`, and separately `x265` -> `libheif`. Both are independent soname breaks that can hit the same `spectacle` binary. Fix both, don't stop after the first one appears resolved.

## Environment

| Component | Version / Detail |
|---|---|
| Distribution | CachyOS (Arch-based), pacman package manager |
| Desktop | KDE Plasma, Wayland session |
| Package manager config | `/etc/pacman.conf`, `IgnorePkg` must sit under `[options]` |
| Screenshot utility | `spectacle` 1:6.7.3-1 (pacman, `extra` repo) |
| Affected libraries | `libjxl` 0.11->0.12, `x265` .so.215->.so.216, `libheif` (depends on x265 soname) |
| Shell tested against | fish; use fish syntax (`for ... end`), not bash heredoc |

Confirm your own versions before applying fixes:

```fish
cat /etc/os-release
pacman -Qi spectacle libjxl x265 libheif | grep -E "Name|Version"
echo $SHELL
```

## 1. Rule out a partial upgrade

```fish
pacman -Qu
```

If this returns nothing, versions are consistent — the failure is an ABI/soname break, not a partial upgrade or version mismatch. Do not chase unrelated systemd/portal crash loops unless `systemctl --user status` actually confirms them as cause.

## 2. Find every missing library, not just the first one

```fish
ldd $(which spectacle) | grep "not found"
```

This is the critical step. It will likely show **multiple** missing libraries at once, e.g.:

```
libjxl.so.0.11 => not found
libx265.so.215 => not found
```

Fix all of them in this pass. Fixing one and re-testing, then coming back for the next, wastes time — the `spectacle -f` error message only shows the *first* missing library it hits, not all of them, so `ldd` is the only way to see the full list.

## 3. For each missing library: find and install the matching old build

General pattern per library `<lib>`:

```fish
curl -s https://archive.archlinux.org/packages/{first-letter}/<lib>/ | grep -oP '<lib>-[^"]*x86_64\.pkg\.tar\.zst"' | sort -u
```

Pick candidates near the current version and download+inspect before installing blindly:

```fish
cd /tmp
curl -s -o check.pkg https://archive.archlinux.org/packages/{first-letter}/<lib>/<lib>-<version>-x86_64.pkg.tar.zst
bsdtar -xOf check.pkg .PKGINFO | grep '^depend'
```

Confirm the `.PKGINFO` shows the exact soname `ldd` reported missing before downloading the full package for real. Never leave a literal `<version>` placeholder in a real curl/pacman command — always resolve the exact filename first.

### libjxl example (soname 0.11 vs 0.12)

```fish
cd /tmp
curl -O https://archive.archlinux.org/packages/l/libjxl/libjxl-0.11.2-2-x86_64.pkg.tar.zst
sudo pacman -U ./libjxl-0.11.2-2-x86_64.pkg.tar.zst
```

### x265 + libheif example (soname 215 vs 216) — dependency chain

x265 cannot be downgraded alone if a newer `libheif` depends on the newer x265 soname — pacman will refuse with `breaks dependency 'libx265.so=216-64' required by libheif`. You must downgrade both together, matched to the same soname:

```fish
cd /tmp
for v in 1.23.1-1 1.23.0-2 1.23.0-1 1.22.2-1 1.22.1-1 1.22.0-2
    curl -s -o check-$v.pkg https://archive.archlinux.org/packages/l/libheif/libheif-$v-x86_64.pkg.tar.zst
    echo "== $v =="
    bsdtar -xOf check-$v.pkg .PKGINFO 2>/dev/null | grep x265
end
```

Pick the newest libheif version whose output shows `depend = libx265.so=215-64` (matching the x265 soname `ldd` reported). Then install x265 and that libheif version **in the same transaction**:

```fish
curl -O https://archive.archlinux.org/packages/l/libheif/libheif-1.23.0-1-x86_64.pkg.tar.zst
curl -O https://archive.archlinux.org/packages/x/x265/x265-4.1-1-x86_64.pkg.tar.zst
sudo pacman -U ./x265-4.1-1-x86_64.pkg.tar.zst ./libheif-1.23.0-1-x86_64.pkg.tar.zst
```

Installing them separately, x265 first, will fail with the dependency error above — pacman needs to see both new versions at once to resolve the graph.

## 4. Lock all downgraded packages in one line

```fish
grep IgnorePkg /etc/pacman.conf
```

If `IgnorePkg` already has entries (check first — don't blindly append and create duplicates), add all downgraded packages space-separated in one line under `[options]`:

```fish
sudo sed -i '/^\[options\]/a IgnorePkg = libjxl x265 libheif' /etc/pacman.conf
grep IgnorePkg /etc/pacman.conf
```

Multiple `IgnorePkg` lines are valid and additive in pacman — no need to merge with pre-existing unrelated entries (e.g. `IgnorePkg = discord`).

## 5. Verify

```fish
ldd $(which spectacle) | grep "not found"
spectacle -f
```

Empty output from the `grep` means all sonames resolved. Press Print Screen to confirm the shortcut itself fires (not just the binary launching cleanly).

## Pitfalls to Avoid

- **Don't stop at the first missing library.** `ldd | grep "not found"` can show two or more unrelated ABI breaks at once (libjxl and x265 are independent bugs that happened to land in the same sync window). Fix all of them before declaring it solved.
- **Don't run `pacman -Syyu` "to force a resync" once you already have `IgnorePkg` entries in place**, expecting it to pull a fixed build. If the specific repo mirror you're on hasn't published the corrected `spectacle`/`qt6-imageformats` build yet, a full resync does nothing except potentially re-break packages you already fixed, if you also strip the `IgnorePkg` lines first. Only remove `IgnorePkg` and resync after confirming (e.g. via the distro's bug tracker/forum) that a fix has actually been published.
- **Don't downgrade a dependency in isolation if something else depends on its new soname.** x265 downgrade will be blocked by libheif until libheif is downgraded to a matching version in the same transaction.
- Do not rely on system notifications or `spectacle -f`'s single error line for the full picture — always cross-check with `ldd`.
- Reinstalling a package at its *current* version (`pacman -S --overwrite '*' <pkg>`) does nothing for an ABI break — confirm with `ldd`, not by reinstalling blindly.
- Never leave a literal `<version>`/`<matched-version>` placeholder in a command you actually run.

## Generalizes To

Any post-update failure where `pacman -Qu` reports clean versions but a binary throws one or more missing `.so` errors is a library ABI/soname break, not a package version mismatch. Get the *complete* list of missing sonames via `ldd` first, resolve each one's dependency chain (some libraries may need to be downgraded together, not individually), then lock all of them with a single combined `IgnorePkg` line until upstream ships matched rebuilds.
