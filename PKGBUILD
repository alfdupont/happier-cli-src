# Maintainer: zhaole_lv
pkgname=happier-cli
pkgver=0.2.14
pkgrel=1
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
source=('copilot-per-model-reasoning-effort.patch')
sha256sums=('0714612ec345eb546acd8721169d966e7bd7a0bdc46908911256c2dd4723f892')
source_x86_64=(
    "${_base}/happier-v${pkgver}-linux-x64.tar.gz"
    "https://raw.githubusercontent.com/happier-dev/happier/cli-v${pkgver}/LICENCE"
)
sha256sums_x86_64=(
    'b8b290abc8080362b3f518590adfb961246815e55850811c8b6a46a041acce45'
    'e251d0448ef3ce023c20ebac9b90a7d8642b1434825838247d6e457668eb3e00'
)
source_aarch64=(
    "${_base}/happier-v${pkgver}-linux-arm64.tar.gz"
    "https://raw.githubusercontent.com/happier-dev/happier/cli-v${pkgver}/LICENCE"
)
sha256sums_aarch64=(
    'b1c39cfe96474b06adfd28caf91f5831b802c097c6848ebad80d1575493abc8f'
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
