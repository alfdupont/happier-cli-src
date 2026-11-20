# Maintainer: zhaole_lv
pkgname=happier-cli
pkgver=0.2.20
pkgrel=3
pkgdesc="Open-source companion for AI coding agents (Claude Code, Codex, OpenCode, etc.)"
arch=('x86_64' 'aarch64')
url="https://github.com/happier-dev/happier"
license=('MIT')
# The happier binary is a Bun single-file executable whose app payload is
# appended after the ELF sections; makepkg's strip would delete it and leave
# a bare "bun" CLI. Keep all shipped binaries byte-for-byte.
options=('!strip')
optdepends=(
    'ripgrep: file search (bundled fallback included)'
)

# Pin the immutable per-version tag: the rolling cli-stable tag only hosts the
# latest release's assets, so its URLs break as soon as upstream moves on.
_tag="cli-v${pkgver}"
_base="https://github.com/happier-dev/happier/releases/download/${_tag}"

# Local patches, applied to the extracted release tree in prepare().
source=(
    'copilot-per-model-reasoning-effort.patch'
    'session-mode-override-retry-backoff.patch'
    'acp-mode-metadata-write-storm.patch'
)
sha256sums=(
    '8ae18a890afea27cc95e347efd6ba7a5185db83fbc9b14db5149b598d87c547c'
    'e765f38d5a9609786482c2c111d522d004937a20e418d7b712b988bbb560d385'
    '220ba34e6b506597b760abee724b289125865a34aca80d467d478edc8c06bee1'
)
source_x86_64=(
    "${_base}/happier-v${pkgver}-linux-x64.tar.gz"
    "https://raw.githubusercontent.com/happier-dev/happier/cli-v${pkgver}/LICENCE"
)
sha256sums_x86_64=(
    'cac890b90e53309afd5178332017b195071cf1e3dd373ac5d8524c2bb2c74b60'
    'e251d0448ef3ce023c20ebac9b90a7d8642b1434825838247d6e457668eb3e00'
)
source_aarch64=(
    "${_base}/happier-v${pkgver}-linux-arm64.tar.gz"
    "https://raw.githubusercontent.com/happier-dev/happier/cli-v${pkgver}/LICENCE"
)
sha256sums_aarch64=(
    'b0f8afe3b6941f0e11a240b59fc15721da726ac16aad8f122c9a64515982f188'
    'e251d0448ef3ce023c20ebac9b90a7d8642b1434825838247d6e457668eb3e00'
)

prepare() {
    local _arch
    case "${CARCH}" in
        x86_64)  _arch="x64" ;;
        aarch64) _arch="arm64" ;;
    esac

    # Show Copilot's per-model reasoning effort in the model picker. The patch
    # edits this release's bundled package-dist chunks, whose file names are
    # content hashes, so it must be ported to every new pkgver.
    patch --directory="${srcdir}/happier-v${pkgver}-linux-${_arch}" --forward \
        --strip=1 --fuzz=0 --input="${srcdir}/copilot-per-model-reasoning-effort.patch"

    # Stop a failing session mode override from being retried on every metadata
    # sync, which saturates the link while the session is idle. Like the patch
    # above, it targets a content-hashed chunk and must be ported per pkgver.
    patch --directory="${srcdir}/happier-v${pkgver}-linux-${_arch}" --forward \
        --strip=1 --fuzz=0 --input="${srcdir}/session-mode-override-retry-backoff.patch"

    # Stop repeated ACP mode notifications (such as a resumed session replaying
    # its history) from queueing one full metadata write each. It builds on the
    # chunks the patches above leave, so it must stay last.
    patch --directory="${srcdir}/happier-v${pkgver}-linux-${_arch}" --forward \
        --strip=1 --fuzz=0 --input="${srcdir}/acp-mode-metadata-write-storm.patch"
}

package() {
    local _arch
    case "${CARCH}" in
        x86_64)  _arch="x64" ;;
        aarch64) _arch="arm64" ;;
    esac

    # The release archive is a self-contained app tree: a native ELF launcher
    # plus its bundled node runtime (node_modules/), application code
    # (package-dist/), helper scripts and pre-unpacked tools (rg, zellij).
    # Install the tree whole under /opt and expose it in PATH. Since 0.2.14,
    # happier downloads its optional runtimes (difftastic, local embeddings)
    # into ~/.happier on first use instead of bundling them.
    install -d "${pkgdir}/opt/${pkgname}"
    cp -a "${srcdir}/happier-v${pkgver}-linux-${_arch}/." "${pkgdir}/opt/${pkgname}/"

    install -d "${pkgdir}/usr/bin"
    ln -s "/opt/${pkgname}/happier" "${pkgdir}/usr/bin/happier"

    # Install the MIT license (the release tarball ships none).
    install -Dm644 "${srcdir}/LICENCE" \
        "${pkgdir}/usr/share/licenses/${pkgname}/LICENCE"
}
