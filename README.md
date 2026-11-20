# happier-cli

AUR packaging repository for `happier-cli`.

Links:

- AUR: <https://aur.archlinux.org/packages/happier-cli>
- GitHub: <https://github.com/stoicneko/happier-cli>
- Upstream: <https://github.com/happier-dev/happier>

## What This Repo Contains

This repository is for Arch Linux packaging metadata, not the upstream application source code.

Main files:

- `PKGBUILD`
- `.SRCINFO`

## Local Patches

This private fork patches the release tree in `prepare()` before installing it:

- `copilot-per-model-reasoning-effort.patch`: lets the model picker offer a
  reasoning-effort choice for GitHub Copilot models. Its header explains the
  bug and the backported fix.
- `session-mode-override-retry-backoff.patch`: stops a session mode override
  the provider refuses (for example a Copilot `--agent-mode` outside its
  allowed modes) from being retried on every metadata sync, which saturated
  the link while the session was idle. See
  `BUG-session-mode-override-retry-loop.md` for the investigation.
- `acp-mode-metadata-write-storm.patch`: stops repeated ACP mode notifications
  from queueing one full session metadata write each. Without it, resuming a
  session that went through the retry loop above replays hundreds of thousands
  of notifications and floods the relay for hours. It must be applied last.

The patches edit bundled `package-dist` chunks whose file names are content hashes, so they currently apply to 0.2.20. When bumping `pkgver`, port them to the new chunks or drop them once upstream ships the fixes. `--fuzz=0` makes the build fail instead of patching the wrong code.

After installing, restart the daemon so it loads the patched code:

```bash
systemctl --user restart happier-daemon.default.service
```

The `aur` remote points at the public AUR package. Don't push this fork there.

## Update Workflow

When upstream releases a new version:

1. Update `pkgver`, release checksums, and packaging logic in `PKGBUILD`.
2. Port or remove release-specific local patches and update their checksums.
3. Regenerate `.SRCINFO` and build the package locally for inspection.
4. Keep this private fork local; do not publish it to the public AUR.

Commands:

```bash
makepkg --printsrcinfo > .SRCINFO
makepkg -f
```

## Local Build

To build and install locally:

```bash
makepkg -sf
makepkg -si
```

Or install the already built package file:

```bash
paru -U ./*.pkg.tar.zst
```

## Notes

- Do not commit built artifacts like `*.pkg.tar.zst`.
- Do not upload package archives to AUR. AUR only stores packaging metadata.
- If `PKGBUILD` changes, regenerate `.SRCINFO` and inspect the built package locally.
