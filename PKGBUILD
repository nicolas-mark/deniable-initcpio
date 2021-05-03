# Maintainer: Nicolas Mark <nicolas.mark@ghostroad.studio>

pkgname="deniable-initcpio"
pkgver={PKG_VERSION}
pkgrel=1
pkgdesc="Full-disk encryption initcpio hook. Reads kernel flags to decrypt loopback device containing headers and keyfiles to unlock block devices."
url="https://github.com/nicolas-mark/deniable-initcpio.git"
license=("GPL3")
arch=('x86_64')
depends=('cryptsetup')
source=(
    "deniable-initcpio"
    'docs/'
)
sha256sums=(
    ''
)
prepare() {
    cd "$srcdir"/${pkgname}-${pkgver}
    make DESTDIR="$pkgdir" install

    install -Dm644 "$srcdir/deniable_hook" "$pkgdir/usr/lib/initcpio/hooks/deniable"
    install -Dm644 "$srcdir/deniable_install" "$pkgdir/usr/lib/initcpio/install/deniable"
}