# LFScripts
Scripts to build Linux From Scratch

version-check.sh
```bash
#!/bin/bash
# A script to list version numbers of critical development tools

# If you have tools installed in other directories, adjust PATH here AND
# in ~lfs/.bashrc (section 4.4) as well.

LC_ALL=C 
PATH=/usr/bin:/bin

bail() { echo "FATAL: $1"; exit 1; }
grep --version > /dev/null 2> /dev/null || bail "grep does not work"
sed '' /dev/null || bail "sed does not work"
sort   /dev/null || bail "sort does not work"

ver_check()
{
   if ! type -p $2 &>/dev/null
   then 
     echo "ERROR: Cannot find $2 ($1)"; return 1; 
   fi
   v=$($2 --version 2>&1 | grep -E -o '[0-9]+\.[0-9\.]+[a-z]*' | head -n1)
   if printf '%s\n' $3 $v | sort --version-sort --check &>/dev/null
   then 
     printf "OK:    %-9s %-6s >= $3\n" "$1" "$v"; return 0;
   else 
     printf "ERROR: %-9s is TOO OLD ($3 or later required)\n" "$1"; 
     return 1; 
   fi
}

ver_kernel()
{
   kver=$(uname -r | grep -E -o '^[0-9\.]+')
   if printf '%s\n' $1 $kver | sort --version-sort --check &>/dev/null
   then 
     printf "OK:    Linux Kernel $kver >= $1\n"; return 0;
   else 
     printf "ERROR: Linux Kernel ($kver) is TOO OLD ($1 or later required)\n" "$kver"; 
     return 1; 
   fi
}

# Coreutils first because --version-sort needs Coreutils >= 7.0
ver_check Coreutils      sort     8.1 || bail "Coreutils too old, stop"
ver_check Bash           bash     3.2
ver_check Binutils       ld       2.13.1
ver_check Bison          bison    2.7
ver_check Diffutils      diff     2.8.1
ver_check Findutils      find     4.2.31
ver_check Gawk           gawk     4.0.1
ver_check GCC            gcc      5.2
ver_check "GCC (C++)"    g++      5.2
ver_check Grep           grep     2.5.1a
ver_check Gzip           gzip     1.3.12
ver_check M4             m4       1.4.10
ver_check Make           make     4.0
ver_check Patch          patch    2.5.4
ver_check Perl           perl     5.8.8
ver_check Python         python3  3.4
ver_check Sed            sed      4.1.5
ver_check Tar            tar      1.22
ver_check Texinfo        texi2any 5.0
ver_check Xz             xz       5.0.0
ver_kernel 4.14

if mount | grep -q 'devpts on /dev/pts' && [ -e /dev/ptmx ]
then echo "OK:    Linux Kernel supports UNIX 98 PTY";
else echo "ERROR: Linux Kernel does NOT support UNIX 98 PTY"; fi

alias_check() {
   if $1 --version 2>&1 | grep -qi $2
   then printf "OK:    %-4s is $2\n" "$1";
   else printf "ERROR: %-4s is NOT $2\n" "$1"; fi
}
echo "Aliases:"
alias_check awk GNU
alias_check yacc Bison
alias_check sh Bash

echo "Compiler check:"
if printf "int main(){}" | g++ -x c++ -
then echo "OK:    g++ works";
else echo "ERROR: g++ does NOT work"; fi
rm -f a.out

if [ "$(nproc)" = "" ]; then
   echo "ERROR: nproc is not available or it produces empty output"
else
   echo "OK: nproc reports $(nproc) logical cores are available"
fi
```

pre-chroot.sh
```bash
mkdir -pv $LFS/{dev,proc,sys,run}
mount -v --bind /dev $LFS/dev
mount -v --bind /dev/pts $LFS/dev/pts
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run
if [ -h $LFS/dev/shm ]; then
    mkdir -pv $LFS/$(readlink $LFS/dev/shm)
else
    mount -t tmpfs -o nosuid,nodev tmpfs $LFS/dev/shm
fi
chroot "$LFS" /usr/bin/env -i HOME=/root TERM="$TERM" PS1='(lfs chroot) \u:\w\$ ' PATH=/usr/bin:/usr/sbin /bin/bash --login
```

backup.sh
```bash
mountpoint -q $LFS/dev/shm && umount $LFS/dev/shm
umount $LFS/dev/pts
umount $LFS/{sys,proc,run,dev}
cd $LFS
tar -cJpf $HOME/lfs-12.0_$(date +%F_%H.%M.%S).tar.xz .
```

chap8-install.sh
```bash
#8.3
cd sources/
tar -xpvf man-pages-6.05.01.tar.xz
cd man-pages-6.05.01
rm -v man3/crypt*
make prefix=/usr install
cd ..
rm -rf man-pages-6.05.01
#8.4
tar -xpvf iana-etc-20230810.tar.gz 
cd iana-etc-20230810
cp -v services protocols /etc
cd ..
rm -rf iana-etc-20230810
#8.5
tar -xpvf glibc-2.38.tar.xz 
cd glibc-2.38
patch -Np1 -i ../glibc-2.38-fhs-1.patch
patch -Np1 -i ../glibc-2.38-memalign_fix-1.patch
mkdir -v build
cd build
echo "rootsbindir=/usr/sbin" > configparms
time { ../configure --prefix=/usr --disable-werror --enable-kernel=4.14 --enable-stack-protector=strong --with-headers=/usr/include libc_cv_slibdir=/usr/lib && make -j10 && make check; }
touch /etc/ld.so.conf
sed '/test-installation/s@$(PERL)@echo not running@' -i ../Makefile
make install
sed '/RTLDLIST=/s@/usr@@g' -i /usr/bin/ldd
cp -v ../nscd/nscd.conf /etc/nscd.conf
mkdir -pv /var/cache/nscd
mkdir -pv /usr/lib/locale
localedef -i POSIX -f UTF-8 C.UTF-8 2> /dev/null || true
localedef -i cs_CZ -f UTF-8 cs_CZ.UTF-8
localedef -i de_DE -f ISO-8859-1 de_DE
localedef -i de_DE@euro -f ISO-8859-15 de_DE@euro
localedef -i de_DE -f UTF-8 de_DE.UTF-8
localedef -i el_GR -f ISO-8859-7 el_GR
localedef -i en_GB -f ISO-8859-1 en_GB
localedef -i en_GB -f UTF-8 en_GB.UTF-8
localedef -i en_HK -f ISO-8859-1 en_HK
localedef -i en_PH -f ISO-8859-1 en_PH
localedef -i en_US -f ISO-8859-1 en_US
localedef -i en_US -f UTF-8 en_US.UTF-8
localedef -i es_ES -f ISO-8859-15 es_ES@euro
localedef -i es_MX -f ISO-8859-1 es_MX
localedef -i fa_IR -f UTF-8 fa_IR
localedef -i fr_FR -f ISO-8859-1 fr_FR
localedef -i fr_FR@euro -f ISO-8859-15 fr_FR@euro
localedef -i fr_FR -f UTF-8 fr_FR.UTF-8
localedef -i is_IS -f ISO-8859-1 is_IS
localedef -i is_IS -f UTF-8 is_IS.UTF-8
localedef -i it_IT -f ISO-8859-1 it_IT
localedef -i it_IT -f ISO-8859-15 it_IT@euro
localedef -i it_IT -f UTF-8 it_IT.UTF-8
localedef -i ja_JP -f EUC-JP ja_JP
localedef -i ja_JP -f SHIFT_JIS ja_JP.SJIS 2> /dev/null || true
localedef -i ja_JP -f UTF-8 ja_JP.UTF-8
localedef -i nl_NL@euro -f ISO-8859-15 nl_NL@euro
localedef -i ru_RU -f KOI8-R ru_RU.KOI8-R
localedef -i ru_RU -f UTF-8 ru_RU.UTF-8
localedef -i se_NO -f UTF-8 se_NO.UTF-8
localedef -i ta_IN -f UTF-8 ta_IN.UTF-8
localedef -i tr_TR -f UTF-8 tr_TR.UTF-8
localedef -i zh_CN -f GB18030 zh_CN.GB18030
localedef -i zh_HK -f BIG5-HKSCS zh_HK.BIG5-HKSCS
localedef -i zh_TW -f UTF-8 zh_TW.UTF-8
cat > /etc/nsswitch.conf << "EOF"
# Begin /etc/nsswitch.conf
passwd: files
group: files
shadow: files
hosts: files dns
networks: files
protocols: files
services: files
ethers: files
rpc: files
# End /etc/nsswitch.conf
EOF
tar -xf ../../tzdata2023c.tar.gz
ZONEINFO=/usr/share/zoneinfo
mkdir -pv $ZONEINFO/{posix,right}
for tz in etcetera southamerica northamerica europe africa antarctica asia australasia backward; do
zic -L /dev/null -d $ZONEINFO ${tz}
zic -L /dev/null -d $ZONEINFO/posix ${tz}
zic -L leapseconds -d $ZONEINFO/right ${tz}
done
cp -v zone.tab zone1970.tab iso3166.tab $ZONEINFO
zic -d $ZONEINFO -p America/New_York
unset ZONEINFO
ln -sfv /usr/share/zoneinfo/$(tzselect) /etc/localtime # or ln -sfv /usr/share/zoneinfo/Asia/Singapore /etc/localtime
cat > /etc/ld.so.conf << "EOF"
# Begin /etc/ld.so.conf
/usr/local/lib
/opt/lib
EOF
cat >> /etc/ld.so.conf << "EOF"
# Add an include directory
include /etc/ld.so.conf.d/*.conf
EOF
mkdir -pv /etc/ld.so.conf.d
cd ../..
rm -rf glibc-2.38
#8.6
tar -xpvf zlib-1.2.13.tar.xz
cd zlib-1.2.13
./configure --prefix=/usr
make -j10
make check
make install
rm -fv /usr/lib/libz.a
cd ..
rm -rf zlib-1.2.13
#8.7
tar -xpvf bzip2-1.0.8.tar.gz
cd bzip2-1.0.8
patch -Np1 -i ../bzip2-1.0.8-install_docs-1.patch
sed -i 's@\(ln -s -f \)$(PREFIX)/bin/@\1@' Makefile
sed -i "s@(PREFIX)/man@(PREFIX)/share/man@g" Makefile
make -f Makefile-libbz2_so
make clean
make -j10
make PREFIX=/usr install
cp -av libbz2.so.* /usr/lib
ln -sv libbz2.so.1.0.8 /usr/lib/libbz2.so
cp -v bzip2-shared /usr/bin/bzip2
for i in /usr/bin/{bzcat,bunzip2}; do
ln -sfv bzip2 $i
done
rm -fv /usr/lib/libbz2.a
cd ..
rm -rf bzip2-1.0.8
#8.8
tar -xpvf xz-5.4.4.tar.xz
cd xz-5.4.4
./configure --prefix=/usr --disable-static --docdir=/usr/share/doc/xz-5.4.4
make -j10
make check
make install
cd ..
rm -rf xz-5.4.4
#8.9
tar -xpvf zstd-1.5.5.tar.gz
cd zstd-1.5.5
make prefix=/usr
make check
make prefix=/usr install
rm -v /usr/lib/libzstd.a
cd ..
rm -rf zstd-1.5.5
#8.10
tar -xpvf file-5.45.tar.gz
cd file-5.45
./configure --prefix=/usr
make -j10
make check
make install
cd ..
rm -rf file-5.45
#8.11
tar -xpvf readline-8.2.tar.gz
cd readline-8.2
sed -i '/MV.*old/d' Makefile.in
sed -i '/{OLDSUFF}/c:' support/shlib-install
patch -Np1 -i ../readline-8.2-upstream_fix-1.patch
./configure --prefix=/usr --disable-static --with-curses --docdir=/usr/share/doc/readline-8.2
make SHLIB_LIBS="-lncursesw" -j10
make SHLIB_LIBS="-lncursesw" install
install -v -m644 doc/*.{ps,pdf,html,dvi} /usr/share/doc/readline-8.2
cd ..
rm -rf readline-8.2
#8.12
tar -xpvf m4-1.4.19.tar.xz
cd m4-1.4.19
./configure --prefix=/usr
make -j10
make check
make install
cd ..
rm -rf m4-1.4.19
#8.13
tar -xpvf bc-6.6.0.tar.xz
cd bc-6.6.0
CC=gcc ./configure --prefix=/usr -G -O3 -r
make -j10
make test
make install
cd ..
rm -rf bc-6.6.0
#8.14
tar -xpvf flex-2.6.4.tar.gz
cd flex-2.6.4
./configure --prefix=/usr --docdir=/usr/share/doc/flex-2.6.4 --disable-static
make -j10
make check
make install
ln -sv flex /usr/bin/lex
ln -sv flex.1 /usr/share/man/man1/lex.1
cd ..
rm -rf flex-2.6.4
#8.15
tar -xpvf tcl8.6.13-src.tar.gz
cd tcl8.6.13
SRCDIR=$(pwd)
cd unix
./configure --prefix=/usr --mandir=/usr/share/man
make -j10
sed -e "s|$SRCDIR/unix|/usr/lib|" -e "s|$SRCDIR|/usr/include|" -i tclConfig.sh
sed -e "s|$SRCDIR/unix/pkgs/tdbc1.1.5|/usr/lib/tdbc1.1.5|" -e "s|$SRCDIR/pkgs/tdbc1.1.5/generic|/usr/include|" -e "s|$SRCDIR/pkgs/tdbc1.1.5/library|/usr/lib/tcl8.6|" -e "s|$SRCDIR/pkgs/tdbc1.1.5|/usr/include|" -i pkgs/tdbc1.1.5/tdbcConfig.sh
sed -e "s|$SRCDIR/unix/pkgs/itcl4.2.3|/usr/lib/itcl4.2.3|" -e "s|$SRCDIR/pkgs/itcl4.2.3/generic|/usr/include|" -e "s|$SRCDIR/pkgs/itcl4.2.3|/usr/include|" -i pkgs/itcl4.2.3/itclConfig.sh
unset SRCDIR
make test
make install
chmod -v u+w /usr/lib/libtcl8.6.so
make install-private-headers
ln -sfv tclsh8.6 /usr/bin/tclsh
mv /usr/share/man/man3/{Thread,Tcl_Thread}.3
cd ..
tar -xf ../tcl8.6.13-html.tar.gz --strip-components=1
mkdir -v -p /usr/share/doc/tcl-8.6.13
cp -v -r ./html/* /usr/share/doc/tcl-8.6.13
cd ..
rm -rf tcl8.6.13
#8.16
tar -xpvf expect5.45.4.tar.gz
tar -xpvf gcc-13.2.0.tar.xz
mv -v expect5.45.4/tclconfig/config.guess expect5.45.4/tclconfig/config.guess.bak
mv -v gcc-13.2.0/config.guess expect5.45.4/tclconfig/config.guess
rm -rf gcc-13.2.0
cd expect5.45.4
./configure --prefix=/usr --with-tcl=/usr/lib --enable-shared --mandir=/usr/share/man --with-tclinclude=/usr/include
make -j10
make test
make install
ln -svf expect5.45.4/libexpect5.45.4.so /usr/lib
cd ..
rm -rf expect5.45.4
#8.17
tar -xvf dejagnu-1.6.3.tar.gz
cd dejagnu-1.6.3
mkdir -v build
cd build
../configure --prefix=/usr
makeinfo --html --no-split -o doc/dejagnu.html ../doc/dejagnu.texi
makeinfo --plaintext -o doc/dejagnu.txt ../doc/dejagnu.texi
make install
install -v -dm755 /usr/share/doc/dejagnu-1.6.3
install -v -m644 doc/dejagnu.{html,txt} /usr/share/doc/dejagnu-1.6.3
make check
cd ../..
rm -rf dejagnu-1.6.3
#8.18
tar -xpvf binutils-2.41.tar.xz
cd binutils-2.41
mkdir -v build
cd build
../configure --prefix=/usr --sysconfdir=/etc --enable-gold --enable-ld=default --enable-plugins --enable-shared --disable-werror --enable-64-bit-bfd --with-system-zlib
make -j10 tooldir=/usr
make -k check
make tooldir=/usr install
rm -fv /usr/lib/lib{bfd,ctf,ctf-nobfd,gprofng,opcodes,sframe}.a
cd ../..
rm -rf binutils-2.41
#8.19
tar -xpvf gmp-6.3.0.tar.xz
cd gmp-6.3.0
./configure --prefix=/usr --enable-cxx --disable-static --docdir=/usr/share/doc/gmp-6.3.0
make -j10
make html
make check 2>&1 | tee gmp-check-log
awk '/# PASS:/{total+=$3} ; END{print total}' gmp-check-log
awk '/# PASS:/{total+=$3} ; END{print total}' gmp-check-log
make install
make install-html
cd ..
rm -rf gmp-6.3.0
#8.20
tar -xpvf mpfr-4.2.0.tar.xz
cd mpfr-4.2.0
sed -e 's/+01,234,567/+1,234,567 /' -e 's/13.10Pd/13Pd/' -i tests/tsprintf.c
./configure --prefix=/usr --disable-static --enable-thread-safe --docdir=/usr/share/doc/mpfr-4.2.0
make -j10
make html
make check
make install
make install-html
cd ..
rm -rf mpfr-4.2.0
#8.21
tar -xpvf mpc-1.3.1.tar.gz
cd mpc-1.3.1
./configure --prefix=/usr --disable-static --docdir=/usr/share/doc/mpc-1.3.1
make -j10
make html
make check
make install
make install-html
cd ..
rm -rf mpc-1.3.1
#8.22
tar -xpvf attr-2.5.1.tar.gz
cd attr-2.5.1
./configure --prefix=/usr --disable-static --sysconfdir=/etc --docdir=/usr/share/doc/attr-2.5.1
make -j10
make check
make install
cd ..
rm -rf attr-2.5.1
#8.23
tar -xpvf acl-2.3.1.tar.xz
cd acl-2.3.1
./configure --prefix=/usr --disable-static --docdir=/usr/share/doc/acl-2.3.1
make -j10
make install
cd ..
rm -rf acl-2.3.1
#8.24
tar -xpvf libcap-2.69.tar.xz
cd libcap-2.69
sed -i '/install -m.*STA/d' libcap/Makefile
make prefix=/usr lib=lib
make test
make prefix=/usr lib=lib install
cd ..
rm -rf libcap-2.69
#8.25
tar -xpvf libxcrypt-4.4.36.tar.xz
cd libxcrypt-4.4.36
./configure --prefix=/usr --enable-hashes=strong,glibc --enable-obsolete-api=no --disable-static --disable-failure-tokens
make -j10
make check
make install
cd ..
rm -rf libxcrypt-4.4.36
#8.26
tar -xpvf shadow-4.13.tar.xz
cd shadow-4.13
sed -i 's/groups$(EXEEXT) //' src/Makefile.in
find man -name Makefile.in -exec sed -i 's/groups\.1 / /' {} \;
find man -name Makefile.in -exec sed -i 's/getspnam\.3 / /' {} \;
find man -name Makefile.in -exec sed -i 's/passwd\.5 / /' {} \;
sed -e 's:#ENCRYPT_METHOD DES:ENCRYPT_METHOD YESCRYPT:' -e 's:/var/spool/mail:/var/mail:' -e '/PATH=/{s@/sbin:@@;s@/bin:@@}' -i etc/login.defs
touch /usr/bin/passwd
./configure --sysconfdir=/etc --disable-static --with-{b,yes}crypt --with-group-name-max-length=32
make -j10
make exec_prefix=/usr install
make -C man install-man
pwconv
grpconv
mkdir -p /etc/default
useradd -D --gid 999
passwd root # or echo "123" | passwd --stdin root
cd ..
rm -rf shadow-4.13
#8.27
tar -xpvf gcc-13.2.0.tar.xz
cd gcc-13.2.0
case $(uname -m) in x86_64) sed -e '/m64=/s/lib64/lib/' -i.orig gcc/config/i386/t-linux64; ;; esac
mkdir -v build
cd build
../configure --prefix=/usr LD=ld --enable-languages=c,c++ --enable-default-pie --enable-default-ssp --disable-multilib --disable-bootstrap --disable-fixincludes --with-system-zlib
make -j10
ulimit -s 32768
chown -Rv tester .
su tester -c "PATH=$PATH make -j10 -k check"
../contrib/test_summary | grep -A7 Summ
make install
chown -v -R root:root /usr/lib/gcc/$(gcc -dumpmachine)/13.2.0/include{,-fixed}
ln -svr /usr/bin/cpp /usr/lib
ln -sv gcc.1 /usr/share/man/man1/cc.1
ln -sfv ../../libexec/gcc/$(gcc -dumpmachine)/13.2.0/liblto_plugin.so /usr/lib/bfd-plugins/
echo 'int main(){}' > dummy.c
cc dummy.c -v -Wl,--verbose &> dummy.log
readelf -l a.out | grep ': /lib'
grep -E -o '/usr/lib.*/S?crt[1in].*succeeded' dummy.log
grep -B4 '^ /usr/include' dummy.log
grep 'SEARCH.*/usr/lib' dummy.log |sed 's|; |\n|g'
grep "/lib.*/libc.so.6 " dummy.log
grep found dummy.log
rm -v dummy.c a.out dummy.log
mkdir -pv /usr/share/gdb/auto-load/usr/lib
mv -v /usr/lib/*gdb.py /usr/share/gdb/auto-load/usr/lib # mv: cannot stat '/usr/lib/*gdb.py': No such file or directory
cd ../..
rm -rf gcc-13.2.0
#8.28
tar -xpvf pkgconf-2.0.1.tar.xz
cd pkgconf-2.0.1
./configure --prefix=/usr --disable-static --docdir=/usr/share/doc/pkgconf-2.0.1
make -j10
make install
ln -sv pkgconf /usr/bin/pkg-config
ln -sv pkgconf.1 /usr/share/man/man1/pkg-config.1
cd ..
rm -rf pkgconf-2.0.1
#8.29
tar -xpvf ncurses-6.4.tar.gz
cd ncurses-6.4
./configure --prefix=/usr --mandir=/usr/share/man --with-shared --without-debug --without-normal --with-cxx-shared --enable-pc-files --enable-widec --with-pkg-config-libdir=/usr/lib/pkgconfig
make -j10
make DESTDIR=$PWD/dest install
install -vm755 dest/usr/lib/libncursesw.so.6.4 /usr/lib
rm -v dest/usr/lib/libncursesw.so.6.4
cp -av dest/* /
for lib in ncurses form panel menu ; do rm -vf /usr/lib/lib${lib}.so; echo "INPUT(-l${lib}w)" > /usr/lib/lib${lib}.so; ln -sfv ${lib}w.pc /usr/lib/pkgconfig/${lib}.pc; done
rm -vf /usr/lib/libcursesw.so
echo "INPUT(-lncursesw)" > /usr/lib/libcursesw.so
ln -sfv libncurses.so /usr/lib/libcurses.so
cp -v -R doc -T /usr/share/doc/ncurses-6.4
cd ..
rm -rf ncurses-6.4
#8.30
tar -xpvf sed-4.9.tar.xz
cd sed-4.9
./configure --prefix=/usr
make -j10
make html
chown -Rv tester .
su tester -c "PATH=$PATH make check"
make install
install -d -m755 /usr/share/doc/sed-4.9
install -m644 doc/sed.html /usr/share/doc/sed-4.9
cd ..
rm -rf sed-4.9
#8.31
tar -xpvf psmisc-23.6.tar.xz
cd psmisc-23.6
./configure --prefix=/usr
make -j10
make check
make install
cd ..
rm -rf psmisc-23.6
#8.32
tar -xpvf gettext-0.22.tar.xz
cd gettext-0.22
./configure --prefix=/usr --disable-static --docdir=/usr/share/doc/gettext-0.22
make -j10
make check
make install
chmod -v 0755 /usr/lib/preloadable_libintl.so
cd ..
rm -rf gettext-0.22
#8.33
tar -xpvf bison-3.8.2.tar.xz
cd bison-3.8.2
./configure --prefix=/usr --docdir=/usr/share/doc/bison-3.8.2
make -j10
make check
make install
cd ..
rm -rf bison-3.8.2
#8.34
tar -xpvf grep-3.11.tar.xz
cd grep-3.11
sed -i "s/echo/#echo/" src/egrep.sh
./configure --prefix=/usr
make -j10
make check
make install
cd ..
rm -rf grep-3.11
#8.35
tar -xpvf bash-5.2.15.tar.gz
cd bash-5.2.15
./configure --prefix=/usr --without-bash-malloc --with-installed-readline --docdir=/usr/share/doc/bash-5.2.15
make -j10
#su -s /usr/bin/expect tester << EOF
#set timeout -1
#spawn make tests
#expect eof
#lassign [wait] _ _ _ value
#exit $value
#EOF
make install
exec /usr/bin/bash --login
cd ..
rm -rf bash-5.2.15
#8.36
tar -xpvf libtool-2.4.7.tar.xz
cd libtool-2.4.7
./configure --prefix=/usr
make -j10
TESTSUITEFLAGS=-j10 make -k check
make install
rm -fv /usr/lib/libltdl.a
cd ..
rm -rf libtool-2.4.7
#8.37
tar -xpvf gdbm-1.23.tar.gz
cd gdbm-1.23
./configure --prefix=/usr --disable-static --enable-libgdbm-compat
make -j10
make check
make install
cd ..
rm -rf gdbm-1.23
#8.38
tar -xpvf gperf-3.1.tar.gz
cd gperf-3.1
./configure --prefix=/usr --docdir=/usr/share/doc/gperf-3.1
make -j10
make -j1 check
make install
cd ..
rm -rf gperf-3.1
#8.39
tar -xpvf expat-2.5.0.tar.xz
cd expat-2.5.0
./configure --prefix=/usr --disable-static --docdir=/usr/share/doc/expat-2.5.0
make -j10
make check
make install
install -v -m644 doc/*.{html,css} /usr/share/doc/expat-2.5.0
cd ..
rm -rf expat-2.5.0
#8.40
tar -xpvf inetutils-2.4.tar.xz
cd inetutils-2.4
./configure --prefix=/usr --bindir=/usr/bin --localstatedir=/var --disable-logger --disable-whois --disable-rcp --disable-rexec --disable-rlogin --disable-rsh --disable-servers
make -j10
make check
make install
mv -v /usr/{,s}bin/ifconfig
cd ..
rm -rf inetutils-2.4
#8.41
tar -xpvf less-643.tar.gz
cd less-643
./configure --prefix=/usr --sysconfdir=/etc
make -j10
make check
make install
cd ..
rm -rf less-643
#8.42
tar -xpvf perl-5.38.0.tar.xz
cd perl-5.38.0
export BUILD_ZLIB=False
export BUILD_BZIP2=0
sh Configure -des -Dprefix=/usr -Dvendorprefix=/usr -Dprivlib=/usr/lib/perl5/5.38/core_perl -Darchlib=/usr/lib/perl5/5.38/core_perl -Dsitelib=/usr/lib/perl5/5.38/site_perl -Dsitearch=/usr/lib/perl5/5.38/site_perl -Dvendorlib=/usr/lib/perl5/5.38/vendor_perl -Dvendorarch=/usr/lib/perl5/5.38/vendor_perl -Dman1dir=/usr/share/man/man1 -Dman3dir=/usr/share/man/man3 -Dpager="/usr/bin/less -isR" -Duseshrplib -Dusethreads
make -j10
make test
make install
unset BUILD_ZLIB BUILD_BZIP2
cd ..
rm -rf perl-5.38.0
#8.43
tar -xpvf XML-Parser-2.46.tar.gz
cd XML-Parser-2.46
perl Makefile.PL
make -j10
make test
make install
cd ..
rm -rf XML-Parser-2.46
#8.44
tar -xpvf intltool-0.51.0.tar.gz
cd intltool-0.51.0
sed -i 's:\\\${:\\\$\\{:' intltool-update.in
./configure --prefix=/usr
make -j10
make check
make install
install -v -Dm644 doc/I18N-HOWTO /usr/share/doc/intltool-0.51.0/I18N-HOWTO
cd ..
rm -rf intltool-0.51.0
#8.45
tar -xpvf autoconf-2.71.tar.xz
cd autoconf-2.71
sed -e 's/SECONDS|/&SHLVL|/' -e '/BASH_ARGV=/a\ /^SHLVL=/ d' -i.orig tests/local.at
./configure --prefix=/usr
make -j10
TESTSUITEFLAGS=-j10 make check
make install
cd ..
rm -rf autoconf-2.71
#8.46
tar -xpvf automake-1.16.5.tar.xz
cd automake-1.16.5
./configure --prefix=/usr --docdir=/usr/share/doc/automake-1.16.5
make -j10
make -j10 check
make install
cd ..
rm -rf automake-1.16.5
#8.47
tar -xpvf openssl-3.1.2.tar.gz
cd openssl-3.1.2
./config --prefix=/usr --openssldir=/etc/ssl --libdir=lib shared zlib-dynamic
make -j10
make test
sed -i '/INSTALL_LIBS/s/libcrypto.a libssl.a//' Makefile
make MANSUFFIX=ssl install
mv -v /usr/share/doc/openssl /usr/share/doc/openssl-3.1.2
cp -vfr doc/* /usr/share/doc/openssl-3.1.2
cd ..
rm -rf openssl-3.1.2
#8.48
tar -xpvf kmod-30.tar.xz
cd kmod-30
./configure --prefix=/usr --sysconfdir=/etc --with-openssl --with-xz --with-zstd --with-zlib
make -j10
make install
for target in depmod insmod modinfo modprobe rmmod; do ln -sfv ../bin/kmod /usr/sbin/$target; done
ln -sfv kmod /usr/bin/lsmod
cd ..
rm -rf kmod-30
#8.49
tar -xpvf elfutils-0.189.tar.bz2
cd elfutils-0.189
./configure --prefix=/usr --disable-debuginfod --enable-libdebuginfod=dummy
make -j10
make check
make -C libelf install
install -vm644 config/libelf.pc /usr/lib/pkgconfig
rm /usr/lib/libelf.a
cd ..
rm -rf elfutils-0.189
#8.50
tar -xpvf libffi-3.4.4.tar.gz
cd libffi-3.4.4
./configure --prefix=/usr --disable-static --libdir=/usr/lib --disable-multi-os-directory --with-gcc-arch=native
make -j10
make check
make install
cd ..
rm -rf libffi-3.4.4
#8.51
tar -xpvf Python-3.11.4.tar.xz
cd Python-3.11.4
./configure --prefix=/usr --enable-shared --with-system-expat --with-system-ffi --enable-optimizations
make -j10
make install
install -v -dm755 /usr/share/doc/python-3.11.4/html
tar --strip-components=1 --no-same-owner --no-same-permissions -C /usr/share/doc/python-3.11.4/html -xvf ../python-3.11.4-docs-html.tar.bz2
cd ..
rm -rf Python-3.11.4
#8.52
tar -xpvf flit_core-3.9.0.tar.gz
cd flit_core-3.9.0
pip3 wheel -w dist --no-build-isolation --no-deps $PWD
pip3 install --no-index --no-user --find-links dist flit_core
cd ..
rm -rf flit_core-3.9.0
#8.53
tar -xpvf wheel-0.41.1.tar.gz
cd wheel-0.41.1
pip3 wheel -w dist --no-build-isolation --no-deps $PWD
pip3 install --no-index --find-links=dist wheel
cd ..
rm -rf wheel-0.41.1
#8.54
tar -xpvf ninja-1.11.1.tar.gz
cd ninja-1.11.1
python3 configure.py --bootstrap
./ninja ninja_test
./ninja_test --gtest_filter=-SubprocessTest.SetWithLots
install -vm755 ninja /usr/bin/
install -vDm644 misc/bash-completion /usr/share/bash-completion/completions/ninja
install -vDm644 misc/zsh-completion /usr/share/zsh/site-functions/_ninja
cd ..
rm -rf ninja-1.11.1
#8.55
tar -xpvf meson-1.2.1.tar.gz
cd meson-1.2.1
pip3 wheel -w dist --no-build-isolation --no-deps $PWD
pip3 install --no-index --find-links dist meson
install -vDm644 data/shell-completions/bash/meson /usr/share/bash-completion/completions/meson
install -vDm644 data/shell-completions/zsh/_meson /usr/share/zsh/site-functions/_meson
cd ..
rm -rf meson-1.2.1
#8.56
tar -xpvf coreutils-9.3.tar.xz
cd coreutils-9.3
patch -Np1 -i ../coreutils-9.3-i18n-1.patch
autoreconf -fiv
FORCE_UNSAFE_CONFIGURE=1 ./configure --prefix=/usr --enable-no-install-program=kill,uptime
make -j10
make NON_ROOT_USERNAME=tester check-root
groupadd -g 102 dummy -U tester
chown -Rv tester .
su tester -c "PATH=$PATH make RUN_EXPENSIVE_TESTS=yes check"
groupdel dummy
make install
mv -v /usr/bin/chroot /usr/sbin
mv -v /usr/share/man/man1/chroot.1 /usr/share/man/man8/chroot.8
sed -i 's/"1"/"8"/' /usr/share/man/man8/chroot.8
cd ..
rm -rf coreutils-9.3
#8.57
tar -xpvf check-0.15.2.tar.gz
cd check-0.15.2
./configure --prefix=/usr --disable-static
make -j10
make check
make docdir=/usr/share/doc/check-0.15.2 install
cd ..
rm -rf check-0.15.2
#8.58
tar -xpvf diffutils-3.10.tar.xz
cd diffutils-3.10
./configure --prefix=/usr
make -j10
make check
make install
cd ..
rm -rf diffutils-3.10
#8.59
tar -xpvf gawk-5.2.2.tar.xz
cd gawk-5.2.2
sed -i 's/extras//' Makefile.in
./configure --prefix=/usr
make -j10
chown -Rv tester .
su tester -c "PATH=$PATH make check"
make LN='ln -f' install
ln -sv gawk.1 /usr/share/man/man1/awk.1
mkdir -pv /usr/share/doc/gawk-5.2.2
cp -v doc/{awkforai.txt,*.{eps,pdf,jpg}} /usr/share/doc/gawk-5.2.2
cd ..
rm -rf gawk-5.2.2
#8.60
tar -xpvf findutils-4.9.0.tar.xz
cd findutils-4.9.0
./configure --prefix=/usr --localstatedir=/var/lib/locate
make -j10
chown -Rv tester .
su tester -c "PATH=$PATH make check"
make install
cd ..
rm -rf findutils-4.9.0
#8.61
tar -xpvf groff-1.23.0.tar.gz
cd groff-1.23.0
PAGE=A4 ./configure --prefix=/usr
make -j10
make check
make install
cd ..
rm -rf groff-1.23.0
#8.62
tar -xpvf grub-2.06.tar.xz
cd grub-2.06
unset {C,CPP,CXX,LD}FLAGS
patch -Np1 -i ../grub-2.06-upstream_fixes-1.patch
./configure --prefix=/usr --sysconfdir=/etc --disable-efiemu --disable-werror
make -j10
make install
mv -v /etc/bash_completion.d/grub /usr/share/bash-completion/completions
cd ..
rm -rf grub-2.06
#8.63
tar -xpvf gzip-1.12.tar.xz
cd gzip-1.12
./configure --prefix=/usr
make -j10
make check
make install
cd ..
rm -rf gzip-1.12
#8.64
tar -xpvf iproute2-6.4.0.tar.xz
cd iproute2-6.4.0
sed -i /ARPD/d Makefile
rm -fv man/man8/arpd.8
make NETNS_RUN_DIR=/run/netns
make SBINDIR=/usr/sbin install
mkdir -pv /usr/share/doc/iproute2-6.4.0
cp -v COPYING README* /usr/share/doc/iproute2-6.4.0
cd ..
rm -rf iproute2-6.4.0
#8.65
tar -xpvf kbd-2.6.1.tar.xz
cd kbd-2.6.1
patch -Np1 -i ../kbd-2.6.1-backspace-1.patch
sed -i '/RESIZECONS_PROGS=/s/yes/no/' configure
sed -i 's/resizecons.8 //' docs/man/man8/Makefile.in
./configure --prefix=/usr --disable-vlock
make -j10
make check
make install
cp -R -v docs/doc -T /usr/share/doc/kbd-2.6.1
cd ..
rm -rf kbd-2.6.1
#8.66
tar -xpvf libpipeline-1.5.7.tar.gz 
cd libpipeline-1.5.7
./configure --prefix=/usr
make -j10
make check
make install
cd ..
rm -rf libpipeline-1.5.7
#8.67
tar -xpvf make-4.4.1.tar.gz
cd make-4.4.1
./configure --prefix=/usr
make -j10
chown -Rv tester .
su tester -c "PATH=$PATH make check"
make install
cd ..
rm -rf make-4.4.1
#8.68
tar -xpvf patch-2.7.6.tar.xz
cd patch-2.7.6
./configure --prefix=/usr
make -j10
make check
make install
cd ..
rm -rf patch-2.7.6
#8.69
tar -xvpf tar-1.35.tar.xz
cd tar-1.35
FORCE_UNSAFE_CONFIGURE=1 ./configure --prefix=/usr
make -j10
TESTSUITEFLAGS=-j10 make check
echo 'LFS: capabilities: binary store/restore, is known to fail.'
make install
make -C doc install-html docdir=/usr/share/doc/tar-1.35
cd ..
rm -rf tar-1.35
#8.70
tar -xpvf texinfo-7.0.3.tar.xz
cd texinfo-7.0.3
./configure --prefix=/usr
make -j10
make check
make install
make TEXMF=/usr/share/texmf install-tex
pushd /usr/share/info
  rm -v dir
  for f in *
    do install-info $f dir 2>/dev/null
  done
popd
cd ..
rm -rf texinfo-7.0.3
#8.71
tar -xpvf vim-9.0.1677.tar.gz
cd vim-9.0.1677
echo '#define SYS_VIMRC_FILE "/etc/vimrc"' >> src/feature.h
./configure --prefix=/usr
make -j10
chown -Rv tester .
su tester -c "LANG=en_US.UTF-8 make -j1 test" &> vim-test.log
make install
ln -sv vim /usr/bin/vi
for L in /usr/share/man/{,*/}man1/vim.1; do
  ln -sv vim.1 $(dirname $L)/vi.1
done
ln -sv ../vim/vim90/doc /usr/share/doc/vim-9.0.1677
cat > /etc/vimrc << "EOF"
" Begin /etc/vimrc
" Ensure defaults are set before customizing settings, not after
source $VIMRUNTIME/defaults.vim
let skip_defaults_vim=1
set nocompatible
set backspace=2
set mouse=
syntax on
if (&term == "xterm") || (&term == "putty")
  set background=dark
endif
" End /etc/vimrc
EOF
cd ..
rm -rf vim-9.0.1677
#8.72
tar -xpvf MarkupSafe-2.1.3.tar.gz
cd MarkupSafe-2.1.3
pip3 wheel -w dist --no-build-isolation --no-deps $PWD
pip3 install --no-index --no-user --find-links dist Markupsafe
cd ..
rm -rf MarkupSafe-2.1.3
#8.73
tar -xpvf Jinja2-3.1.2.tar.gz
cd Jinja2-3.1.2
pip3 wheel -w dist --no-build-isolation --no-deps $PWD
pip3 install --no-index --no-user --find-links dist Jinja2
cd ..
rm -rf Jinja2-3.1.2
#8.74
tar -xpvf systemd-254.tar.gz
cd systemd-254
sed -i -e 's/GROUP="render"/GROUP="video"/' -e 's/GROUP="sgx", //' rules.d/50-udev-default.rules.in
sed '/systemd-sysctl/s/^/#/' -i rules.d/99-systemd.rules.in
mkdir -p build
cd build
meson setup --prefix=/usr --buildtype=release -Dmode=release -Ddev-kvm-mode=0660 -Dlink-udev-shared=false ..
ninja udevadm systemd-hwdb $(grep -o -E "^build (src/libudev|src/udev|rules.d|hwdb.d)[^:]*" build.ninja | awk '{ print $2 }') $(realpath libudev.so --relative-to .)
rm rules.d/90-vconsole.rules
install -vm755 -d {/usr/lib,/etc}/udev/{hwdb,rules}.d
install -vm755 -d /usr/{lib,share}/pkgconfig
install -vm755 udevadm /usr/bin/
install -vm755 systemd-hwdb /usr/bin/udev-hwdb
ln -svfn ../bin/udevadm /usr/sbin/udevd
cp -av libudev.so{,*[0-9]} /usr/lib/
install -vm644 ../src/libudev/libudev.h /usr/include/
install -vm644 src/libudev/*.pc /usr/lib/pkgconfig/
install -vm644 src/udev/*.pc /usr/share/pkgconfig/
install -vm644 ../src/udev/udev.conf /etc/udev/
install -vm644 rules.d/* ../rules.d/{*.rules,README} /usr/lib/udev/rules.d/
install -vm644 hwdb.d/* ../hwdb.d/{*.hwdb,README} /usr/lib/udev/hwdb.d/
install -vm755 $(find src/udev -type f | grep -F -v ".") /usr/lib/udev
tar -xvf ../../udev-lfs-20230818.tar.xz
make -f udev-lfs-20230818/Makefile.lfs install
tar -xvf ../../systemd-man-pages-254.tar.xz --no-same-owner --strip-components=1 -C /usr/share/man --wildcards '*/udev*' '*/libudev*' '*/systemd-'{hwdb,udevd.service}.8
sed 's/systemd\(\\\?-\)/udev\1/' /usr/share/man/man8/systemd-hwdb.8 > /usr/share/man/man8/udev-hwdb.8
sed 's|lib.*udevd|sbin/udevd|' /usr/share/man/man8/systemd-udevd.service.8 > /usr/share/man/man8/udevd.8
rm /usr/share/man/man8/systemd-*.8
udev-hwdb update
cd ../..
rm -rf systemd-254
#8.75
tar -xpvf man-db-2.11.2.tar.xz
cd man-db-2.11.2
./configure --prefix=/usr --docdir=/usr/share/doc/man-db-2.11.2 --sysconfdir=/etc --disable-setuid --enable-cache-owner=bin --with-browser=/usr/bin/lynx --with-vgrind=/usr/bin/vgrind --with-grap=/usr/bin/grap --with-systemdtmpfilesdir= --with-systemdsystemunitdir=
make -j10
make -k check
echo 'LFS: man1/lexgrog.1 is known to fail.'
make install
cd ..
rm -rf man-db-2.11.2
#8.76
tar -xpvf procps-ng-4.0.3.tar.xz
cd procps-ng-4.0.3
./configure --prefix=/usr --docdir=/usr/share/doc/procps-ng-4.0.3 --disable-static --disable-kill
make -j10
make check
make install
cd ..
rm -rf procps-ng-4.0.3
#8.77
tar -xpvf util-linux-2.39.1.tar.xz
cd util-linux-2.39.1
sed -i '/test_mkfds/s/^/#/' tests/helpers/Makemodule.am
./configure ADJTIME_PATH=/var/lib/hwclock/adjtime --bindir=/usr/bin --libdir=/usr/lib --runstatedir=/run --sbindir=/usr/sbin --disable-chfn-chsh --disable-login --disable-nologin --disable-su --disable-setpriv --disable-runuser --disable-pylibmount --disable-static --without-python --without-systemd --without-systemdsystemunitdir --docdir=/usr/share/doc/util-linux-2.39.1
make -j10
chown -Rv tester .
su tester -c "make -k check"
make install
cd ..
rm -rf util-linux-2.39.1
#8.78
tar -xpvf e2fsprogs-1.47.0.tar.gz
cd e2fsprogs-1.47.0
mkdir -v build
cd build
../configure --prefix=/usr --sysconfdir=/etc --enable-elf-shlibs --disable-libblkid --disable-libuuid --disable-uuidd --disable-fsck
make -j10
make check
echo 'LFS: m_assume_storage_prezeroed is known to fail.'
make install
gunzip -v /usr/share/info/libext2fs.info.gz
install-info --dir-file=/usr/share/info/dir /usr/share/info/libext2fs.info
makeinfo -o doc/com_err.info ../lib/et/com_err.texinfo
install -v -m644 doc/com_err.info /usr/share/info
install-info --dir-file=/usr/share/info/dir /usr/share/info/com_err.info
cd ../..
rm -rf e2fsprogs-1.47.0
#8.79
tar -xpvf sysklogd-1.5.1.tar.gz
cd sysklogd-1.5.1
sed -i '/Error loading kernel symbols/{n;n;d}' ksym_mod.c
sed -i 's/union wait/int/' syslogd.c
make -j10
make BINDIR=/sbin install
cat > /etc/syslog.conf << "EOF"
# Begin /etc/syslog.conf
auth,authpriv.* -/var/log/auth.log
*.*;auth,authpriv.none -/var/log/sys.log
daemon.* -/var/log/daemon.log
kern.* -/var/log/kern.log
mail.* -/var/log/mail.log
user.* -/var/log/user.log
*.emerg *
# End /etc/syslog.conf
EOF
cd ..
rm -rf sysklogd-1.5.1
#8.80
tar -xpvf sysvinit-3.07.tar.xz
cd sysvinit-3.07
patch -Np1 -i ../sysvinit-3.07-consolidated-1.patch
make -j10
make install
cd ..
rm -rf sysvinit-3.07
```

stripping_and_cleanup.sh
```bash
#8.82
save_usrlib="$(cd /usr/lib; ls ld-linux*[^g])
    libc.so.6
    libthread_db.so.1
    libquadmath.so.0.0.0
    libstdc++.so.6.0.32
    libitm.so.1.0.0
    libatomic.so.1.2.0"

cd /usr/lib

for LIB in $save_usrlib; do
    objcopy --only-keep-debug $LIB $LIB.dbg
    cp $LIB /tmp/$LIB
    strip --strip-unneeded /tmp/$LIB
    objcopy --add-gnu-debuglink=$LIB.dbg /tmp/$LIB
    install -vm755 /tmp/$LIB /usr/lib
    rm /tmp/$LIB
done

online_usrbin="bash find strip"
online_usrlib="libbfd-2.41.so
    libsframe.so.1.0.0
    libhistory.so.8.2
    libncursesw.so.6.4
    libm.so.6
    libreadline.so.8.2
    libz.so.1.2.13
    $(cd /usr/lib; find libnss*.so* -type f)"

for BIN in $online_usrbin; do
    cp /usr/bin/$BIN /tmp/$BIN
    strip --strip-unneeded /tmp/$BIN
    install -vm755 /tmp/$BIN /usr/bin
    rm /tmp/$BIN
done

for LIB in $online_usrlib; do
    cp /usr/lib/$LIB /tmp/$LIB
    strip --strip-unneeded /tmp/$LIB
    install -vm755 /tmp/$LIB /usr/lib
    rm /tmp/$LIB
done

for i in $(find /usr/lib -type f -name \*.so* ! -name \*dbg) $(find /usr/lib -type f -name \*.a) $(find /usr/{bin,sbin,libexec} -type f); do
    case "$online_usrbin $online_usrlib $save_usrlib" in
        *$(basename $i)* )
            ;;
        * ) strip --strip-unneeded $i
            ;;
    esac
done

unset BIN LIB save_usrlib online_usrbin online_usrlib

#8.83
rm -rf /tmp/*
find /usr/lib /usr/libexec -name \*.la -delete
find /usr -depth -name $(uname -m)-lfs-linux-gnu\* | xargs rm -rf
userdel -r tester
```

system_configuration.sh
```bash
#9.2
tar -xpvf lfs-bootscripts-20230728.tar.xz
cd lfs-bootscripts-20230728
make install
cd ..
rm -rf lfs-bootscripts-20230728
#9.5.2
cat > /etc/resolv.conf << "EOF"
# Begin /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
# End /etc/resolv.conf
EOF
#9.5.3
echo "LFS" > /etc/hostname
#9.5.4
cat > /etc/hosts << "EOF"
# Begin /etc/hosts
127.0.0.1 localhost LFS
::1 localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
# End /etc/hosts
EOF
#9.6.2
cat > /etc/inittab << "EOF"
# Begin /etc/inittab
id:3:initdefault:
si::sysinit:/etc/rc.d/init.d/rc S
l0:0:wait:/etc/rc.d/init.d/rc 0
l1:S1:wait:/etc/rc.d/init.d/rc 1
l2:2:wait:/etc/rc.d/init.d/rc 2
l3:3:wait:/etc/rc.d/init.d/rc 3
l4:4:wait:/etc/rc.d/init.d/rc 4
l5:5:wait:/etc/rc.d/init.d/rc 5
l6:6:wait:/etc/rc.d/init.d/rc 6
ca:12345:ctrlaltdel:/sbin/shutdown -t1 -a -r now
su:S06:once:/sbin/sulogin
s1:1:respawn:/sbin/sulogin
1:2345:respawn:/sbin/agetty --noclear tty1 9600
2:2345:respawn:/sbin/agetty tty2 9600
3:2345:respawn:/sbin/agetty tty3 9600
4:2345:respawn:/sbin/agetty tty4 9600
5:2345:respawn:/sbin/agetty tty5 9600
6:2345:respawn:/sbin/agetty tty6 9600
# End /etc/inittab
EOF
#9.6.4
cat > /etc/sysconfig/clock << "EOF"
# Begin /etc/sysconfig/clock
UTC=1
# Set this to any options you might need to give to hwclock,
# such as machine hardware clock type for Alphas.
CLOCKPARAMS=
# End /etc/sysconfig/clock
EOF
#9.6.8.1
sed -i.bak 's/#VERBOSE_FSCK=no/VERBOSE_FSCK=yes/' /etc/sysconfig/rc.site
#9.7
cat > /etc/profile << "EOF"
# Begin /etc/profile
export LANG=en_US.UTF-8
# End /etc/profile
EOF
#9.8
cat > /etc/inputrc << "EOF"
# Begin /etc/inputrc
# Modified by Chris Lynn <roryo@roryo.dynup.net>
# Allow the command prompt to wrap to the next line
set horizontal-scroll-mode Off
# Enable 8-bit input
set meta-flag On
set input-meta On
# Turns off 8th bit stripping
set convert-meta Off
# Keep the 8th bit for display
set output-meta On
# none, visible or audible
set bell-style none
# All of the following map the escape sequence of the value
# contained in the 1st argument to the readline specific functions
"\eOd": backward-word
"\eOc": forward-word
# for linux console
"\e[1~": beginning-of-line
"\e[4~": end-of-line
"\e[5~": beginning-of-history
"\e[6~": end-of-history
"\e[3~": delete-char
"\e[2~": quoted-insert
# for xterm
"\eOH": beginning-of-line
"\eOF": end-of-line
# for Konsole
"\e[H": beginning-of-line
"\e[F": end-of-line
# End /etc/inputrc
EOF
#9.9
cat > /etc/shells << "EOF"
# Begin /etc/shells
/bin/sh
/bin/bash
# End /etc/shells
EOF
```
