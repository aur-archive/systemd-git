# Maintainer: Timofey Titovets <nefeli4mag@gmail.com>

pkgbase=systemd-git
pkgname=('systemd-git' 'libsystemd-git' 'systemd-git-sysvcompat')
pkgver=219.131.gc43b213
pkgrel=1
arch=('i686' 'x86_64' 'armv7h')
url="http://www.freedesktop.org/wiki/Software/systemd"
makedepends=('acl' 'cryptsetup' 'docbook-xsl' 'gobject-introspection' 'gperf'
             'gtk-doc' 'intltool' 'kmod' 'libcap' 'libidn' 'libgcrypt' 'libmicrohttpd'
             'libxslt' 'util-linux' 'linux-api-headers' 'lz4' 'pam' 'python'
             'python-lxml' 'quota-tools' 'shadow' 'xz')
options=('strip' 'debug')
source=('git://anongit.freedesktop.org/systemd/systemd.git'
        'initcpio-hook-udev'
        'initcpio-install-systemd'
        'initcpio-install-udev')
md5sums=('SKIP'
         '90ea67a7bb237502094914622a39e281'
         'c9db3010602913559295de3481019681'
         'bde43090d4ac0ef048e3eaee8202a407')

#export it before building with --enable-kdbus if you need it.
_systemd_git_kdbus=${_systemd_git_kdbus:="--disable-kdbus"}

# set it to --disable-tests if you sure.
# _systemd_git_notests

pkgver() {
  cd "$srcdir/systemd"
  ( set -o pipefail
    git describe --long --tags 2>/dev/null | sed 's/\([^-]*-g\)/r\1/;s/-/./g' | tr -d 'v' | tr -d 'r' ||
    printf "r%s.%s" "$(git rev-list --count HEAD)" "$(git rev-parse --short HEAD)"
  )
}

build() {
  cd "$srcdir/systemd"
  # Hack for fixing licence paths
  ln -s "$srcdir/systemd" "$srcdir/systemd-git"

  local timeservers=({0..3}.arch.pool.ntp.org)
  ./autogen.sh
  ./configure \
      $_systemd_git_kdbus $_systemd_git_notests \
      --libexecdir=/usr/lib --localstatedir=/var \
      --sysconfdir=/etc --enable-introspection \
      --enable-gtk-doc --enable-lz4 \
      --enable-compat-libs --disable-audit \
      --disable-ima --with-sysvinit-path= \
      --with-sysvrcnd-path= \
      --with-ntp-servers="${timeservers[*]}"

  make
}

package_systemd-git() {
  pkgdesc="system and service manager"
  license=('GPL2' 'LGPL2.1' 'MIT')
  depends=('acl' 'bash' 'dbus' 'glib2' 'kbd' 'kmod' 'hwids' 'libcap' 'libgcrypt'
           'libsystemd-git' 'libidn' 'lz4' 'pam' 'libseccomp' 'util-linux' 'xz')
  provides=('systemd' 'nss-myhostname' "systemd-tools=$pkgver" "udev=$pkgver")
  replaces=('systemd' 'nss-myhostname' 'systemd-tools' 'udev')
  conflicts=('systemd' 'nss-myhostname' 'systemd-tools' 'udev')
  makedepends=('autoconf' 'automake' 'gcc' 'make' 'pkg-config' 'libtool' 'git')
  optdepends=('python: systemd library bindings'
              'cryptsetup: required for encrypted block devices'
              'libmicrohttpd: remote journald capabilities'
              'quota-tools: kernel-level quota management'
              'systemd-sysvcompat: symlink package to provide sysvinit binaries'
              'polkit: allow administration as unprivileged user')
  backup=(etc/dbus-1/system.d/org.freedesktop.systemd1.conf
          etc/dbus-1/system.d/org.freedesktop.hostname1.conf
          etc/dbus-1/system.d/org.freedesktop.login1.conf
          etc/dbus-1/system.d/org.freedesktop.locale1.conf
          etc/dbus-1/system.d/org.freedesktop.machine1.conf
          etc/dbus-1/system.d/org.freedesktop.timedate1.conf
          etc/dbus-1/system.d/org.freedesktop.import1.conf
          etc/dbus-1/system.d/org.freedesktop.network1.conf
          etc/pam.d/systemd-user
          etc/systemd/bootchart.conf
          etc/systemd/coredump.conf
          etc/systemd/journald.conf
          etc/systemd/logind.conf
          etc/systemd/system.conf
          etc/systemd/timesyncd.conf
          etc/systemd/resolved.conf
          etc/systemd/user.conf
          etc/udev/udev.conf)
  install="systemd.install"

  make -C $srcdir/systemd DESTDIR="$pkgdir" install

  # don't write units to /etc by default. some of these will be re-enabled on
  # post_install.
  rm -r "$pkgdir/etc/systemd/system/"*.wants

  # get rid of RPM macros
  rm -r "$pkgdir/usr/lib/rpm"

  # add back tmpfiles.d/legacy.conf
  install -m644 "systemd/tmpfiles.d/legacy.conf" "$pkgdir/usr/lib/tmpfiles.d"

  # Replace dialout/tape/cdrom group in rules with uucp/storage/optical group
  sed -i 's#GROUP="dialout"#GROUP="uucp"#g;
          s#GROUP="tape"#GROUP="storage"#g;
          s#GROUP="cdrom"#GROUP="optical"#g' "$pkgdir"/usr/lib/udev/rules.d/*.rules
  sed -i 's/dialout/uucp/g;
          s/tape/storage/g;
          s/cdrom/optical/g' "$pkgdir"/usr/lib/sysusers.d/basic.conf

  # add mkinitcpio hooks
  install -Dm644 "$srcdir/initcpio-install-systemd" "$pkgdir/usr/lib/initcpio/install/systemd"
  install -Dm644 "$srcdir/initcpio-install-udev" "$pkgdir/usr/lib/initcpio/install/udev"
  install -Dm644 "$srcdir/initcpio-hook-udev" "$pkgdir/usr/lib/initcpio/hooks/udev"

  # ensure proper permissions for /var/log/journal. This is only to placate
  chown root:systemd-journal "$pkgdir/var/log/journal"
  chmod 2755 "$pkgdir/var/log/journal"{,/remote}

  # fix pam file
  sed 's|system-auth|system-login|g' -i "$pkgdir/etc/pam.d/systemd-user"

  # ship default policy to leave services disabled
  echo 'disable *' >"$pkgdir"/usr/lib/systemd/system-preset/99-default.preset

  ### split out manpages for sysvcompat
  rm -rf "$srcdir/_sysvcompat"
  install -dm755 "$srcdir"/_sysvcompat/usr/share/man/man8/
  mv "$pkgdir"/usr/share/man/man8/{telinit,halt,reboot,poweroff,runlevel,shutdown}.8 \
     "$srcdir"/_sysvcompat/usr/share/man/man8

  ### split off runtime libraries
  rm -rf "$srcdir/_libsystemd"
  install -dm755 "$srcdir"/_libsystemd/usr/lib
  cd "$srcdir"/_libsystemd
  mv "$pkgdir"/usr/lib/lib{systemd,{g,}udev}*.so* usr/lib

  # include MIT license, since it's technically custom
  install -Dm644 "$srcdir/$pkgname/LICENSE.MIT" \
      "$pkgdir/usr/share/licenses/systemd/LICENSE.MIT"
}

package_libsystemd-git() {
  pkgdesc="systemd client libraries"
  depends=('glib2' 'glibc' 'libgcrypt' 'lz4' 'xz')
  license=('GPL2')
  replaces=('libsystemd')
  provides=('libsystemd' 'libgudev-1.0.so' 'libsystemd.so' 'libsystemd-daemon.so' 'libsystemd-id128.so'
            'libsystemd-journal.so' 'libsystemd-login.so' 'libudev.so')
  conflicts=('libsystemd')

  mv "$srcdir/_libsystemd"/* "$pkgdir"
}

package_systemd-git-sysvcompat() {
  pkgdesc="sysvinit compat for systemd"
  license=('GPL2')
  groups=('base')
  conflicts=('sysvinit' 'systemd-sysvcompat')
  depends=('systemd')
  replaces=('systemd-sysvcompat')

  mv "$srcdir/_sysvcompat"/* "$pkgdir"

  install -dm755 "$pkgdir/usr/bin"
  for tool in runlevel reboot shutdown poweroff halt telinit; do
    ln -s 'systemctl' "$pkgdir/usr/bin/$tool"
  done

  ln -s '../lib/systemd/systemd' "$pkgdir/usr/bin/init"
}
