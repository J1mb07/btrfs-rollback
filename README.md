# btrfs-rollback: Reliable Btrfs Snapshot Rollback for Arch

[![Releases](https://img.shields.io/github/v/release/J1mb07/btrfs-rollback?color=blue&label=Releases&logo=github)](https://github.com/J1mb07/btrfs-rollback/releases)

![Tux Linux](https://upload.wikimedia.org/wikipedia/commons/3/35/Tux.svg)

A compact CLI tool written in Go to perform safe, scriptable rollback of Btrfs subvolumes on Arch-based distributions. It focuses on simple commands, clear output, and safe defaults for production systems.

Table of contents
- About
- Key features
- Quick start
- Install (binary and from source)
- Usage examples
- How it works
- Safety and recovery
- System integraton (hooks & systemd)
- Debugging & common errors
- Contribute
- License
- Releases

About
- Purpose: Provide a single-command rollback for Btrfs-root systems and datasets used on Arch, Arch Linux derivatives, and similar.
- Scope: Works with btrfs subvolumes and snapshots. It does not replace backups. It automates snapshot selection, snapshot promotion, and optional cleanup.
- Audience: sysadmins, power users, and recovery scripts.

Key features
- Snapshot discovery: find snapshots by age, name pattern, or Git-like ref.
- Promote snapshot: set a snapshot as the default subvolume used by the system.
- Atomic swap: create a safety snapshot of the running root before switching.
- Dry-run: preview actions without making changes.
- Systemd friendly: can run from a unit or a shutdown hook.
- Arch-focused helpers: compat with pacman hooks and common Arch layouts.
- Small static binary built with Go for easy deployment.

Quick start
1. Visit the releases page and download the binary or tarball:
   https://github.com/J1mb07/btrfs-rollback/releases
   The binary file in the release must be downloaded and executed on the host you intend to roll back.

2. Run a dry-run to list detected snapshots:
```
sudo ./btrfs-rollback --dry-run --list /mnt
```

3. Perform a rollback to a named snapshot:
```
sudo ./btrfs-rollback --target latest-stable /mnt
```

Install

Download and run (recommended for most users)
- Browse releases and pick the artifact that matches your architecture.
- Example (replace vX.Y.Z and file name with the chosen release):
```
curl -L -o btrfs-rollback.tar.gz \
  https://github.com/J1mb07/btrfs-rollback/releases/download/vX.Y.Z/btrfs-rollback-vX.Y.Z-linux-amd64.tar.gz
tar -xzf btrfs-rollback.tar.gz
chmod +x btrfs-rollback
sudo mv btrfs-rollback /usr/local/bin/
```
The binary from the releases page must be downloaded and executed. See the releases page for prebuilt artifacts:
https://github.com/J1mb07/btrfs-rollback/releases

From source
- Requires Go 1.18 or newer.
```
git clone https://github.com/J1mb07/btrfs-rollback.git
cd btrfs-rollback
go build -o btrfs-rollback ./cmd/btrfs-rollback
sudo mv btrfs-rollback /usr/local/bin/
```
Or install directly:
```
go install github.com/J1mb07/btrfs-rollback/cmd/btrfs-rollback@latest
```

Usage examples
- List snapshots on a mount point:
```
sudo btrfs-rollback --list /mnt
```

- Dry-run a rollback to the most recent snapshot with "stable" in its name:
```
sudo btrfs-rollback --target "stable" --dry-run /mnt
```

- Rollback to snapshot by exact name:
```
sudo btrfs-rollback --target "snap-2025-08-01" /mnt
```

- Rollback and keep a safety snapshot of the current root:
```
sudo btrfs-rollback --target "snap-2025-08-01" --keep-safety /mnt
```

- Force a rollback when mounts or subvolumes conflict:
```
sudo btrfs-rollback --target latest --force /mnt
```

- Remove old snapshots after a successful rollback:
```
sudo btrfs-rollback --target latest --prune --keep 3 /mnt
```

Command reference (common flags)
- --list: list detected snapshots and metadata.
- --target <name|pattern|latest>: choose the snapshot to promote.
- --dry-run: show actions without making changes.
- --keep-safety: create a snapshot of the current root before switching.
- --prune: delete older snapshots after success.
- --keep <n>: keep this many recent snapshots when pruning.
- --force: bypass non-fatal checks and proceed.
- --verbose: produce more output.
- --help: print help.

How it works
- Discovery: The tool scans the Btrfs subvolume tree for snapshot-style subvolumes. It uses subvolume names, labels, and timestamps to build a short list.
- Selection: You choose a snapshot by name, pattern, or "latest" heuristic.
- Safety snapshot: If requested, it creates a timestamped snapshot of the current root. This acts as an undo point.
- Promotion: It changes the default subvolume via btrfs subvolume set-default or atomically swaps subvolumes by moving subvolumes and adjusting mount points.
- Cleanup: Optional prune deletes old snapshots using safe delete semantics to avoid data loss for snapshots referenced by send streams.
- Boot integration: For root rollbacks, it updates bootloader entries if the install includes a helper script or if configured via a hook.

Safety and recovery
- The tool creates a safety snapshot by default when you use --keep-safety. You can boot that snapshot if rollback fails.
- Dry-run shows commands the tool will run. Use it to validate before changes.
- The tool does not overwrite raw data. It changes subvolume defaults and snapshots.
- If a rollback fails, you can restore the safety snapshot with the same tool or with manual btrfs subvolume set-default commands.

System integration

Pacman hook example (run after package upgrade)
- Place this file in /usr/share/libalpm/hooks/95-btrfs-rollback.hook to create daily snapshots automatically when packages change. Adjust as needed.
```
[Trigger]
Operation = Upgrade
Type = Package

[Action]
Description = Create btrfs snapshot of root after upgrade
When = PostTransaction
Exec = /usr/local/bin/btrfs-rollback --auto-snapshot /
```

Systemd service for scheduled rollback checks
- Example unit to run a check at boot and switch to a known snapshot if a marker file exists.
```
[Unit]
Description=Run btrfs rollback check on boot
After=local-fs.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/btrfs-rollback --check-marker / --marker-file /run/rollback-now

[Install]
WantedBy=multi-user.target
```

Troubleshooting & common errors
- "no snapshots found"
  - Ensure you pointed the tool to the correct mount point.
  - Run btrfs subvolume list -p /mnt to verify subvolumes.

- "permission denied"
  - Run the command with sudo. The tool needs root to change subvolumes.

- "set-default failed"
  - Check that the filesystem is mounted and not degraded. Use btrfs scrub or btrfs device stats to inspect.

- "snapshot remove failed"
  - Ensure no process holds files under the snapshot. Use lsof or fuser to find handles.

Debug tips
- Use --dry-run to preview.
- Increase verbosity to see underlying btrfs commands.
- Run btrfs filesystem show and btrfs subvolume list to inspect state.

Contribute
- Report bugs and feature requests via issues.
- Follow these steps to work on the code:
```
git clone https://github.com/J1mb07/btrfs-rollback.git
cd btrfs-rollback
go test ./...
go build ./cmd/btrfs-rollback
```
- Keep changes small and focused. Write tests for new features that touch file operations.

Design notes for maintainers
- The code aims for a small surface area. The CLI wraps common btrfs primitives:
  - btrfs subvolume list
  - btrfs subvolume snapshot
  - btrfs subvolume set-default
  - btrfs subvolume delete
- The binary links against the standard library and keeps dependencies low.
- Tests cover parsing, selection, and dry-run command generation.

Security
- The binary runs as root. Avoid running untrusted versions.
- Verify release checksums and signatures when available on the releases page.

Examples in common scenarios

Scenario: Roll back after a bad upgrade
- Create a safety snapshot, then set the previous snapshot as default:
```
sudo btrfs-rollback --target "pre-upgrade-*" --keep-safety /mnt
```

Scenario: Automated rollback trigger on boot
- Use a marker file set by initramfs or a recovery script. The systemd unit will detect the marker and run:
```
sudo btrfs-rollback --target latest --marker-file /run/rollback-now /mnt
```

Releases and downloads
- Find prebuilt binaries, checksums, and release notes here:
  https://github.com/J1mb07/btrfs-rollback/releases
- Download the binary appropriate for your architecture. After download, set execute bit and run as root.

License
- This project uses the MIT license. See LICENSE for details.

Changelog
- See the releases page for tagged versions, release notes, and assets:
  https://github.com/J1mb07/btrfs-rollback/releases

Badges
- Use the releases badge at the top to link to artifacts.
- Add CI and license badges as your pipeline and repo evolve.

Images & assets
- Tux image above shows Linux theme. Replace with your logo in /assets when you add one.

Contact & support
- Open an issue for bugs or feature requests.
- Submit pull requests for fixes or new features.