# PipeWire Audio Config — Reference

System: CachyOS, fish shell, PipeWire 1.6.8. Goal: RT-stable, high-quality, low-latency audio, with VDO.Ninja capture routing that excludes Discord.

## Active config files

| File | Purpose |
|---|---|
| `/etc/pipewire/pipewire.conf.d/10-resample-quality.conf` | Max resampler quality |
| `/etc/pipewire/pipewire.conf.d/15-clock-quantum.conf` | Clock rate + buffer size |
| `/etc/security/limits.d/95-realtime.conf` | RT scheduling limits for `realtime` group |
| `/home/muzasio/.local/bin/discord-audio-fix.sh` | Routes Discord → HyperX direct, else → vdo-combine |
| `~/.config/pipewire/pipewire.conf.d/20-vdo-routing.conf.disabled` | Old WirePlumber-rule approach — DISABLED, do not re-enable (conflicts with the script) |

## Target values

```
resample.quality = 15
resample.disable = false
default.clock.rate = 48000
default.clock.allowed-rates = [ 48000 ]
default.clock.min-quantum = 256
default.clock.max-quantum = 2048
default.clock.quantum = 512
```

User must be in `realtime` group; `rtkit-daemon.service` must be enabled.

## Known hardware limit (not a bug)

HyperX 7.1 USB DAC (`alsa.resolution_bits = 16`) is hardware-capped at 16-bit output. PipeWire's internal graph runs float32 regardless — sink reporting `s16le` on output is correct and expected, not a misconfiguration.

## Routing logic (discord-audio-fix.sh)

Polls every 5s via `pw-dump`, matches `application.process.binary == 'Discord'`. Match → HyperX direct (excluded from capture). No match → `vdo-combine` (captured by VDO.Ninja).

Confirmed live values: Discord reports `application.process.binary = 'Discord'`, `application.name = 'WEBRTC VoiceEngine'`. Non-Discord apps report their own binary/app name (e.g. `msedge`, `cs2`).

## Health check — what to verify

1. `pw-metadata -n settings` → clock values match target above
2. Resample config file → `resample.quality = 15`
3. `pactl info` → `Default Sample Specification: float32le 48000Hz`
4. `journalctl --user -u pipewire` → no `nice-level ... Permission denied` line
5. `groups` → includes `realtime`
6. `ps aux | grep discord-audio-fix.sh` → process alive
7. `pw-top` while audio plays → `ERR` column stays ~0

## If routing breaks

Discord's `application.process.binary` may change after a Flatpak update. Re-check it via `pw-dump` while Discord plays audio (Voice & Video → test sound), filter for `media.class == Stream/Output/Audio`, read `application.process.binary`. If it no longer equals `'Discord'`, update the match string in `discord-audio-fix.sh`.

## Known open item

`discord-audio-fix.sh` runs as a plain background process, not a systemd unit — not guaranteed to survive crash/relaunch. Consider converting to `systemd --user` service.
