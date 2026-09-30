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

The patch edits bundled `package-dist` chunks whose file names are content
hashes, so it only applies to 0.2.12. When bumping `pkgver`, port it to the
new chunks, or drop it once upstream ships the fix. `--fuzz=0` makes the build
fail instead of patching the wrong code.

After installing, restart the daemon so it loads the patched code:

```bash
systemctl --user restart happier-daemon.default.service
```

The `aur` remote points at the public AUR package. Don't push this fork there.

## Update Workflow

When upstream releases a new version:

1. Update `pkgver`, checksums, or packaging logic in `PKGBUILD`.
2. Regenerate `.SRCINFO`.
3. Commit the changes.
4. Push to GitHub and AUR.

Commands:

```bash
makepkg --printsrcinfo > .SRCINFO
git add PKGBUILD .SRCINFO
git commit -m "Update to <version>"
git push origin master
git push aur master
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
- If `PKGBUILD` changes, regenerate `.SRCINFO` before pushing.
