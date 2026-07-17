# Tor Browser: Silent Audio, Broken Install, Bookmark Loss After System Update

Diagnosis and repair for Tor Browser after a full system update on an
Arch-based distribution: no audio output, a broken `torbrowser-launcher`
installation, and bookmarks missing after profile restoration.

## Environment

| Component | Version / Detail |
|---|---|
| Distribution | CachyOS (Arch-based), pacman package manager |
| Package manager config | `/etc/pacman.conf` with `multilib` repo enabled |
| Audio server | PipeWire 1.6.8, PulseAudio-compatible protocol v15.0.0, native protocol v35 |
| Audio sample spec | float32le, 2 channel, 48000 Hz |
| Browser package | `torbrowser-launcher` 0.3.9-1 (pacman, not Flatpak, not manual tarball) |
| Shell tested against | fish (interactive use); commands below are bash, see note in Pitfalls |
| Profile format | Firefox-based (Gecko), `places.sqlite` in WAL journal mode |

Confirm your own versions before applying fixes — behavior differs by distro,
audio server, and install method:

```bash
cat /etc/os-release
pactl info | grep -E "Server Name|Server Version|Protocol"
pacman -Q torbrowser-launcher
echo $SHELL
```

## 1. No Audio Output

Confirm the system audio stack is healthy before touching the browser:

```bash
pactl info
```

Check `Default Sink`, `Default Source`, and that `Server Name` reports
PipeWire. If this fails, the problem is system-wide, not browser-specific.

If `pactl info` succeeds but Tor Browser has no audio, install method
determines the fix:

**Flatpak install** — sandboxing blocks the PipeWire/Pulse socket by default:

```bash
flatpak info --show-permissions <app-id> | grep -i socket
flatpak override --user --socket=pulseaudio <app-id>
```

**Pacman or tarball install** — sandbox permissions don't apply. Audio loss
here after a system update is more often a corrupted binary or profile.
Go to Section 2 rather than chasing Flatpak-specific fixes.

## 2. Repairing a Broken Install Without Losing Profile Data

`torbrowser-launcher` has no scriptable "install complete" signal, so this
is interactive by design — it pauses once for you to confirm the browser
opened before restoring your profile.

```bash
#!/bin/bash
set -euo pipefail

TBB_ROOT="$HOME/.local/share/torbrowser/tbb/x86_64/tor-browser"
PROFILE_REL="Browser/TorBrowser/Data/Browser/profile.default"
BACKUP_DIR="$HOME/tor_profile_backup_$(date +%Y%m%d_%H%M%S)"

echo "[*] Terminating running Tor Browser processes..."
pkill -f torbrowser-launcher || true
pkill -f "tor-browser" || true
sleep 1

if [ -d "$TBB_ROOT/$PROFILE_REL" ]; then
    echo "[*] Backing up profile to $BACKUP_DIR"
    mkdir -p "$BACKUP_DIR"
    cp -a "$TBB_ROOT/$PROFILE_REL" "$BACKUP_DIR/profile.default"
else
    echo "[!] No existing profile found, skipping backup."
fi

echo "[*] Removing installed Tor Browser directory..."
rm -rf "$TBB_ROOT"

echo "[*] Launching torbrowser-launcher to redownload and reinstall..."
torbrowser-launcher &

echo
echo "Wait for the launcher to finish downloading, verifying, and open"
echo "Tor Browser once. Close it fully once it has opened, then press Enter."
read -r

echo "[*] Waiting for reinstalled profile directory..."
for _ in $(seq 1 30); do
    [ -d "$TBB_ROOT/$PROFILE_REL" ] && break
    sleep 1
done

if [ ! -d "$TBB_ROOT/Browser" ]; then
    echo "[!] Reinstall not detected at $TBB_ROOT. Aborting restore."
    exit 1
fi

if [ -d "$BACKUP_DIR/profile.default" ]; then
    echo "[*] Restoring profile (bookmarks, logins, preferences)..."
    rm -rf "$TBB_ROOT/$PROFILE_REL"
    cp -a "$BACKUP_DIR/profile.default" "$TBB_ROOT/$PROFILE_REL"
    echo "[+] Profile restored."
else
    echo "[*] No backup available; fresh profile in use."
fi

echo "[+] Done. Backup retained at: $BACKUP_DIR"
```

Verify after running:

```bash
ls -la ~/.local/share/torbrowser/tbb/x86_64/tor-browser/Browser/TorBrowser/Data/Browser/profile.default/places.sqlite
```

## 3. Bookmarks Missing After Restore

### Root cause

Firefox-based browsers use SQLite WAL mode for `places.sqlite`. If a backup
is taken while the WAL (`places.sqlite-wal`) holds writes not yet committed
to the main file, copying `places.sqlite` alone produces an inconsistent
database — Firefox silently drops the bookmark data instead of erroring.

Check for the mismatch by comparing modification times of all three files
in both the live profile and the backup:

```bash
ls -la <profile>/places.sqlite*
ls -la <backup>/places.sqlite*
```

If `-wal` in the backup is newer than `places.sqlite`, the WAL wasn't
checkpointed before backup.

### Fix: restore the database, WAL, and SHM as a matched set

```bash
#!/bin/bash
set -euo pipefail

PROFILE="$1"   # live profile.default path
BACKUP="$2"    # backed-up profile.default path

echo "[*] Terminating running Tor Browser processes..."
pkill -f torbrowser-launcher || true
pkill -f "tor-browser" || true
sleep 2

if [ ! -f "$BACKUP/places.sqlite" ]; then
    echo "[!] No places.sqlite found in backup at $BACKUP"
    exit 1
fi

echo "[*] Removing stale WAL/SHM from live profile..."
rm -f "$PROFILE/places.sqlite-wal" "$PROFILE/places.sqlite-shm"

echo "[*] Copying database, WAL, and SHM as a matched set..."
cp -a "$BACKUP/places.sqlite" "$PROFILE/places.sqlite"
[ -f "$BACKUP/places.sqlite-wal" ] && cp -a "$BACKUP/places.sqlite-wal" "$PROFILE/places.sqlite-wal"
[ -f "$BACKUP/places.sqlite-shm" ] && cp -a "$BACKUP/places.sqlite-shm" "$PROFILE/places.sqlite-shm"

echo "[+] Restore complete. Launch the browser and verify immediately."
ls -la "$PROFILE"/places.sqlite*
```

Run with paths as arguments: `bash script.sh <profile-path> <backup-path>`.
Launch the browser and check bookmarks immediately — Firefox checkpoints and
deletes the WAL on open, making this a one-shot recovery.

### Fallback: JSON bookmark snapshot

Independent of `places.sqlite` state, Firefox-based browsers periodically
export bookmarks to `bookmarkbackups/*.jsonlz4` inside the profile. More
reliable than raw SQLite recovery when the database itself is in doubt.

```bash
ls -la <profile>/bookmarkbackups/
```

Pick the most recent snapshot with the highest bookmark count (encoded in
the filename), then in-browser: `Ctrl+Shift+O` → **Import and Backup** →
**Restore** → **Choose File...** → select it → confirm overwrite.

## Pitfalls to Avoid

- **Don't assume Flatpak sandboxing on a pacman install.** Checking
  `flatpak override` against a non-Flatpak install wastes a diagnostic pass;
  confirm install method first.
- **Don't inline `VAR=value` bash syntax into a fish shell.** Fish requires
  `set VAR value` and errors on standard bash assignment. Run scripts via
  `bash script.sh` to sidestep shell differences entirely.
- **Don't copy `places.sqlite` without its `-wal` and `-shm` siblings.** A
  backup taken while the browser was recently running can have newer data
  in the WAL than the main file — losing bookmarks with no error.
- **Don't treat a `torbrowser-launcher` reinstall as complete once the
  process starts.** The download/verify step has no scriptable completion
  signal; wait for the browser window to actually open.
- **Don't overwrite a live profile without a timestamped backup first.** A
  failed restore mid-way leaves no fallback otherwise.

## Generalizes To

- Any SQLite database accessed by a running app (Chromium profiles, most
  desktop app stores): back up `.sqlite`, `.sqlite-wal`, `.sqlite-shm`
  together, never `.sqlite` alone, if the app may have run recently.
- Any CLI installer without a scriptable completion signal: pause for
  manual confirmation rather than guessing a timeout.
