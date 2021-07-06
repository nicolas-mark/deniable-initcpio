# Maintainer: Nicolas Mark <nicolas.mark@ghostroad.studio>

pkgname="deniable-initcpio"
pkgver=r5.481299e
pkgrel=1
arch=('x86_64')
pkgdesc="Full-disk encryption initcpio hook. Reads kernel flags to decrypt loopback device containing headers and keyfiles to unlock block devices."
url="https://github.com/nicolas-mark/deniable-initcpio.git"
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

    install -Dm644 "./deniable_hook" "/usr/lib/initcpio/hooks/deniable"
    install -Dm644 "./deniable_install" "/usr/lib/initcpio/install/deniable"
}