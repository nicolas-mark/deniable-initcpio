# Maintainer: Nicolas Mark <nicolas.mark@ghostroad.studio>

pkgname="deniable-initcpio"
pkgver=r2.e187d5b
pkgrel=1
arch=('x86_64')
pkgdesc="Full-disk encryption initcpio hook. Reads kernel flags to decrypt loopback device containing headers and keyfiles to unlock block devices."
url="https://github.com/nicolas-mark/deniable-initcpio"
license=("GPL3")
depends=('cryptsetup')
makedepends=('git')
source=(
    "git+$url"
)
md5sums=('SKIP')

pkgver() {
    cd "$pkgname"
    printf "r%s.%s" "$(git rev-list --count HEAD)" "$(git rev-parse --short HEAD)"
}

package() {
    cd "${pkgname}"

    make PREFIX=/usr DESTDIR="${pkgdir}" install
    install -Dm644 "deniable_hook" "${pkgdir}/usr/lib/initcpio/hooks/deniable"
    install -Dm644 "deniable_install" "${pkgdir}/usr/lib/initcpio/install/deniable"
    install -Dm644 "LICENSE" "${pkgdir}/usr/share/licenses/${pkgname}/LICENSE"
    install -Dm644 "README.md" "${pkgdir}/usr/share/doc/${pkgname}/README.md"
}
