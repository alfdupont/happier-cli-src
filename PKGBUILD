# Maintainer: zhaole_lv
pkgname=happier-cli
pkgver=0.2.12
pkgrel=2
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
    'difftastic: structural diff (bundled fallback included)'
)

# Pin the immutable per-version tag: the rolling cli-stable tag only hosts the
# latest release's assets, so its URLs break as soon as upstream moves on.
_tag="cli-v${pkgver}"
_base="https://github.com/happier-dev/happier/releases/download/${_tag}"

# Local patches, applied to the extracted release tree in prepare().
source=('copilot-per-model-reasoning-effort.patch')
sha256sums=('2ff7db4acb615b139a25496696b5b05f73374a8e9d028125ed478d48cac8d66b')
source_x86_64=(
    "${_base}/happier-v${pkgver}-linux-x64.tar.gz"
    "https://raw.githubusercontent.com/happier-dev/happier/cli-v${pkgver}/LICENCE"
)
sha256sums_x86_64=(
    '8978265ce7677bd3f0a0f2f734304bb551452c962b7d46f8f34420892a01fa7e'
    'e251d0448ef3ce023c20ebac9b90a7d8642b1434825838247d6e457668eb3e00'
)
source_aarch64=(
    "${_base}/happier-v${pkgver}-linux-arm64.tar.gz"
    "https://raw.githubusercontent.com/happier-dev/happier/cli-v${pkgver}/LICENCE"
)
sha256sums_aarch64=(
    'bfa15e2aa79a6dbd0a714c7e0a6d51be011a43ddde2f4f453d6458b15b430e82'
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
}

package() {
    local _arch
    case "${CARCH}" in
        x86_64)  _arch="x64" ;;
        aarch64) _arch="arm64" ;;
    esac

    # The release archive is a self-contained app tree: a native ELF launcher
    # plus its bundled node runtime (node_modules/), application code
    # (package-dist/), helper scripts and pre-unpacked tools (rg, difft,
    # zellij). Install the tree whole under /opt and expose it in PATH.
    install -d "${pkgdir}/opt/${pkgname}"
    cp -a "${srcdir}/happier-v${pkgver}-linux-${_arch}/." "${pkgdir}/opt/${pkgname}/"

    install -d "${pkgdir}/usr/bin"
    ln -s "/opt/${pkgname}/happier" "${pkgdir}/usr/bin/happier"

    # Install the MIT license (the release tarball ships none).
    install -Dm644 "${srcdir}/LICENCE" \
        "${pkgdir}/usr/share/licenses/${pkgname}/LICENCE"
}
