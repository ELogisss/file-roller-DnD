# Maintainer: logis <logis@archlinux>

pkgname=file-roller-dnd-git
_gitname=file-roller
pkgver=44.7.r0.g70b26ce
pkgrel=1
pkgdesc="GNOME archive manager with drag-and-drop extraction for GTK4 (git)"
arch=('x86_64')
url="https://github.com/Elogisss/$_gitname"
license=('GPL-2.0-or-later')
depends=(
  'dconf' 'glib2' 'glibc' 'gtk4'
  'hicolor-icon-theme' 'json-glib' 'libadwaita'
  'libarchive' 'libgcc' 'pango'
)
makedepends=('meson' 'ninja' 'git')
optdepends=('7zip: 7z archive support')
provides=("$_gitname=$pkgver")
conflicts=("$_gitname")
source=("$_gitname::git+$url.git")
sha256sums=('SKIP')

pkgver() {
  cd "$srcdir/$_gitname"
  git describe --long --tags | sed 's/\([^-]*-g\)/r\1/;s/-/./g'
}

build() {
  arch-meson "$_gitname" build
  ninja -C build
}

package() {
  DESTDIR="$pkgdir" meson install -C build
}
