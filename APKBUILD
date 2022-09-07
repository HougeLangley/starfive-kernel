# Maintainer: Milan P. Stanić <mps@arvanta.net>

_flavor=edge
pkgname=linux-${_flavor}
# NOTE: this kernel is intended for testing
# please resist urge to upgrade it blindly
pkgver=5.19.0
case $pkgver in
	*.*.*)	_kernver=${pkgver%.*};;
	*.*) _kernver=$pkgver;;
esac
pkgrel=0
pkgdesc="Linux latest stable kernel"
provides="linux-edge4virt=$pkgver-$pkgrel"
replaces="linux-edge4virt"
url="https://www.kernel.org"
depends="initramfs-generator"
_depends_dev="perl gmp-dev elfutils-dev bash flex bison"
makedepends="$_depends_dev sed installkernel bc linux-headers linux-firmware-any
	openssl-dev diffutils findutils xz ncurses-dev"
options="!strip !check" # no tests
_config=${config:-config-edge.${CARCH}}
install=

subpackages="$pkgname-dev:_dev:$CBUILD_ARCH $pkgname-doc:_doc"
source="http://hougearch.litterhougelangley.club/linux/linux-$pkgver.tar.xz http://hougearch.litterhougelangley.club/linux/config-edge.riscv64"

builddir="$srcdir/linux-${pkgver}"
arch="riscv64"
license="GPL-2.0"

_flavors=
for _i in $source; do
	case $_i in
	config-*.$CARCH)
		_f=${_i%.$CARCH}
		_f=${_f#config-}
		_flavors="$_flavors ${_f}"
		if [ "linux-$_f" != "$pkgname" ]; then
			subpackages="$subpackages linux-${_f}::$CBUILD_ARCH linux-${_f}-dev:_dev:$CBUILD_ARCH"
		fi
		;;
	esac
done

_carch=${CARCH}
case "$_carch" in
riscv64) _carch="riscv" ;;
esac

#prepare() {
#	local _patch_failed=
#  cd $builddir
#	case $pkgver in
#		*.*.0);;
#		*)
#		msg "Applying patch-$pkgver.xz"
#		unxz -c < "$srcdir"/patch-$pkgver.xz | patch -p1 -N ;;
#	esac
#
	# first apply patches in specified order
#	for i in $source; do
#		case $i in
#		*.patch)
#			msg "Applying $i..."
#			if ! patch -s -p1 -N -i "$srcdir"/$i; then
#				echo $i >>failed
#				_patch_failed=1
#			fi
#			;;
#		esac
#	done
#
#	if ! [ -z "$_patch_failed" ]; then
#		error "The following patches failed:"
#		cat failed
#		return 1
#	fi
#
	# remove localversion from patch if any
#	rm -f localversion*
#	oldconfig
#}

#oldconfig() {
#	for i in $_flavors; do
#		local _config=config-$i.${CARCH}
#		mkdir -p "$builddir"
#		echo "-$pkgrel-$i" > "$builddir"/localversion-alpine \
#			|| return 1
#
#		cp "$srcdir"/$_config "$builddir"/.config
#		make -C $builddir \
#			O="$builddir" \
#			ARCH="$_carch" \
#			listnewconfig oldconfig
#	done
#}

build() {
	cp "$srcdir"/config-edge.riscv64 "$builddir"/.config
	unset LDFLAGS
	export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"
	for i in $_flavors; do
		cd "$builddir"
		make -j40 ARCH="$_carch" CC="${CC:-gcc}" \
			KBUILD_BUILD_VERSION="$((pkgrel + 1 ))-Alpine"
	done
}

_package() {
	local _buildflavor="$1" _outdir="$2"
	local _abi_release=${pkgver}-${pkgrel}-${_buildflavor}
	export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

	cd "$builddir"
	# modules_install seems to regenerate a defect Modules.symvers on s390x. Work
	# around it by backing it up and restore it after modules_install
	cp Module.symvers Module.symvers.backup

	mkdir -p "$_outdir"/boot "$_outdir"/lib/modules

	local _install
	case "$CARCH" in
		riscv64) _install="install dtbs_install";;
		*) _install=install;;
	esac

	make -j40 modules_install $_install \
		ARCH="$_carch" \
		INSTALL_MOD_PATH="$_outdir" \
		INSTALL_PATH="$_outdir"/boot \
		INSTALL_DTBS_PATH="$_outdir/boot/dtbs-$_buildflavor"

	cp Module.symvers.backup Module.symvers

	rm -f "$_outdir"/lib/modules/${_abi_release}/build \
		"$_outdir"/lib/modules/${_abi_release}/source
	rm -rf "$_outdir"/lib/firmware

	install -D -m644 include/config/kernel.release \
		"$_outdir"/usr/share/kernel/$_buildflavor/kernel.release
}

# main flavor installs in $pkgdir
package() {
	depends="$depends linux-firmware-any"

	_package edge "$pkgdir"
}

_dev() {
	local _flavor=$(echo $subpkgname | sed -E 's/(^linux-|-dev$)//g')
	local _abi_release=${pkgver}-${pkgrel}-$_flavor
	# copy the only the parts that we really need for build 3rd party
	# kernel modules and install those as /usr/src/linux-headers,
	# simlar to what ubuntu does
	#
	# this way you dont need to install the 300-400 kernel sources to
	# build a tiny kernel module
	#
	pkgdesc="Headers and script for third party modules for $_flavor kernel"
	depends="$_depends_dev"
	local dir="$subpkgdir"/usr/src/linux-headers-${_abi_release}
	export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

	# first we import config, run prepare to set up for building
	# external modules, and create the scripts
	mkdir -p "$dir"
	cp "$srcdir"/config-$_flavor.${CARCH} "$dir"/.config
	echo "-$pkgrel-$_flavor" > "$dir"/localversion-alpine
	cd $builddir

  echo "Installing headers..."
	case "$_carch" in
	x86_64)
		_carch="x86"
		install -Dt "${dir}/tools/objtool" $builddir/tools/objtool/objtool
		;;
	esac
  cp -t "$dir" -a $builddir/include

  install -Dt "${dir}" -m644 $builddir/Makefile
  install -Dt "${dir}" -m644 $builddir/Module.symvers
  install -Dt "${dir}" -m644 $builddir/System.map
	cp -t "$dir" -a $builddir/scripts


  install -Dt "${dir}/arch/${_carch}" -m644 $builddir/arch/${_carch}/Makefile
  install -Dt "${dir}/arch/${_carch}/kernel" -m644 $builddir/arch/${_carch}/kernel/asm-offsets.s
  cp -t "${dir}/arch/${_carch}" -a $builddir/arch/${_carch}/include

  install -Dt "$dir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$dir/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "$dir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "$dir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$dir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$dir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "$dir/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  echo "Removing unneeded architectures..."
  local arch
  for arch in "$dir"/arch/*/; do
		case $(basename "$arch") in $_carch) continue ;; esac
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -bi "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "$builddir/vmlinux"

  echo "Adding symlink..."
	mkdir -p "$subpkgdir"/lib/modules/${_abi_release}
	ln -sf /usr/src/linux-headers-${_abi_release} \
		"$subpkgdir"/lib/modules/${_abi_release}/build
}

_doc() {
	pkgdesc="documentation for $_flavor kernel"
	mkdir -p "$subpkgdir"/usr/share/doc/linux-edge-doc
	cp -r "$builddir"/Documentation \
		"$subpkgdir"/usr/share/doc/linux-edge-doc/

}

sha512sums="
6c5f629b2d033cdc7323608e66b6ea2c8b668e549f1e849cb6f1b3ddc93cd2c42b24a66ed0ca5fec48df53d26e51fea32ad58d2347b90dfd1041a9c15783e8e9  linux-5.19.0.tar.xz
8fd947baa9048c24b7d6591851c03104ec2ea9194c37fe803b797552bc504c67eddbc902b1fc4e19a8a314e4d1535c9c48a742d8ce0d32a66a9caad045584f16  config-edge.riscv64
"
