# Spectacle: Print Screen Fails Silently After System Update

Print Screen (KDE Spectacle) stops working after `pacman -Syu` due to a shared library ABI break between `libjxl` and the `spectacle`/`qt6-imageformats` builds shipped in the same sync window.

## Environment

| Component | Version / Detail |
|---|---|
| Distribution | CachyOS (Arch-based), pacman package manager |
| Desktop | KDE Plasma, Wayland session |
| Package manager config | `/etc/pacman.conf`, `IgnorePkg` must sit under `[options]` |
| Screenshot utility | `spectacle` 1:6.7.3-1 (pacman, `extra` repo) |
| Affected library | `libjxl` 0.12.0-1 |
| Portal service | `plasma-xdg-desktop-portal-kde.service` (systemd user unit) |
| Shell tested against | bash for commands below; confirm equivalents if using fish/zsh |

Confirm your own versions before applying fixes:

```
cat /etc/os-release
pacman -Qi spectacle libjxl | grep -E "Name|Version"
systemctl --user status plasma-xdg-desktop-portal-kde.service
echo $SHELL
```

## 1. Rule out a partial upgrade

Confirm all portal/compositor packages are actually in sync before assuming a repo version mismatch:

```
pacman -Qu
grep -iE "spectacle|kwin|xdg-desktop-portal" /var/log/pacman.log | tail -30
```

If `pacman -Qu` returns nothing, package versions are consistent — the failure is not a partial upgrade, and downgrading `kwin` or `xdg-desktop-portal-kde` will not fix it.

## 2. Identify the real error

Notifications and `journalctl` only show that spectacle failed to start, not why. Run the binary directly:

```
spectacle -f
```

Expected output when this bug is present:

```
spectacle: error while loading shared libraries: libjxl.so.0.11: cannot open shared object file: No such file or directory
```

## 3. Confirm root cause

```
ldd $(which spectacle) | grep -i jxl
pacman -Qi libjxl | grep Version
```

Root cause: `libjxl` bumped its soname from `0.11` to `0.12` (an upstream ABI break). The `libjxl` *package* upgraded cleanly to the new version, but `spectacle` and `qt6-imageformats` in the repo were still linked against the old `.so.0.11` at the time of sync. All package versions report as current and matched — `pacman -Qu` and `pacman -Qi` show nothing wrong — because the mismatch is in the binary's compiled-in library reference, not in package version metadata.

## 4. Downgrade libjxl

Check local pacman cache first:

```
ls /var/cache/pacman/pkg/ | grep libjxl
```

If no pre-0.12 build is cached, pull one from the Arch Linux Archive:

```
curl -s https://archive.archlinux.org/packages/l/libjxl/ | grep -oP 'libjxl-0\.11[^"]*x86_64\.pkg\.tar\.zst"' | sort -u
cd /tmp
curl -O https://archive.archlinux.org/packages/l/libjxl/libjxl-<matched-version>-x86_64.pkg.tar.zst
sudo pacman -U ./libjxl-<matched-version>-x86_64.pkg.tar.zst
```

## 5. Lock the downgrade

Prevent the next `pacman -Syu` from re-breaking it until upstream republishes a matched build:

```
sudo sed -i '/^\[options\]/a IgnorePkg = libjxl' /etc/pacman.conf
grep -n "IgnorePkg\|^\[options\]" /etc/pacman.conf
```

`IgnorePkg` must be under `[options]`. Placing it under `[multilib]` or any repo section is silently ignored by pacman.

## 6. Verify

```
ldd $(which spectacle) | grep jxl
spectacle -f
```

`ldd` should resolve `libjxl.so.0.11` to a real path. Press Print Screen to confirm the shortcut fires correctly.

## Pitfalls to Avoid

- Do not assume a partial upgrade without checking `pacman -Qu` first — matched package versions with an ABI break inside won't show up there.
- Do not chase unrelated systemd service crashes (e.g. a portal service `core-dump`/`start-limit-hit` loop) unless confirmed as the actual cause — check with `systemctl --user status`; a self-recovered crash loop is a red herring.
- Do not rely on system notifications for the real error — run the failing binary directly in a terminal.
- Reinstalling a package at its *current* version (`pacman -S --overwrite '*' <pkg>`) does nothing if the binary itself needs to be relinked against a different library version. Confirm with `ldd` before reinstalling.
- Downgrading the dependent package (`spectacle`) instead of the package that actually changed ABI (`libjxl`) will reproduce the identical error — verify via `ldd` which library is actually missing before choosing what to downgrade.
- Never run a downgrade command with a literal placeholder left in (e.g. `<old-version>`) — always resolve the exact version string first with `ls /var/cache/pacman/pkg/` or the archive listing.

## Generalizes To

Any post-update failure where `pacman -Qu`/`-Qi` report clean versions but a binary throws a missing `.so` error is a library ABI/soname break, not a package version mismatch — the fix is downgrading the library the binary was actually linked against, confirmed via `ldd`, not the application package itself.
