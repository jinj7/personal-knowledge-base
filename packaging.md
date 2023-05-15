
# 1. Preparations

```sh
$ sudo apt install debhelper dh-make dh-systemd -y
```

&nbsp;


# 2. Create a new directory

```sh
$ mkdir test-1.0.0

$ cd test-1.0.0/

$ dh_make -e "<jinaijun@bytedance.com>" --createorig --single
Maintainer Name     : unknown
Email-Address       : <jinaijun@bytedance.com>
Date                : Tue, 25 Oct 2022 19:10:31 +0800
Package Name        : test
Version             : 1.0.0
License             : blank
Package Type        : single
Are the details correct? [Y/n/q]
Currently there is not top level Makefile. This may require additional tuning
Done. Please edit the files in the debian/ subdirectory now.

$ tree
.
└── debian
    ├── changelog
    ├── compat
    ├── control
    ├── copyright
    ├── manpage.1.ex
    ├── manpage.sgml.ex
    ├── manpage.xml.ex
    ├── menu.ex
    ├── postinst.ex
    ├── postrm.ex
    ├── preinst.ex
    ├── prerm.ex
    ├── README.Debian
    ├── README.source
    ├── rules
    ├── source
    │   └── format
    ├── test.cron.d.ex
    ├── test.doc-base.EX
    ├── test-docs.docs
    └── watch.ex

2 directories, 20 files
```

&nbsp;


# 3. Create a new .service file

```sh
$ cat debian/test.service
[Unit]
Description=getting IPv4 addrs of a specified interface
After=network.target
ConditionPathExists=/etc/itfname

[Service]
Type=oneshot
ExecStart=/usr/local/bin/get_ip

[Install]
WantedBy=multi-user.target
```

&nbsp;


# 4. Build this package

```sh
$ dpkg-buildpackage -b -rfakeroot -uc -us
dpkg-buildpackage: info: source package test
dpkg-buildpackage: info: source version 1.0.0-1
dpkg-buildpackage: info: source distribution unstable
dpkg-buildpackage: info: source changed by unknown <<jinaijun@bytedance.com>>
dpkg-buildpackage: info: host architecture amd64
 dpkg-source --before-build .
 fakeroot debian/rules clean
dh clean
   dh_clean
 debian/rules build
dh build
   dh_update_autotools_config
   dh_autoreconf
   create-stamp debian/debhelper-build-stamp
 fakeroot debian/rules binary
dh binary
   dh_testroot
   dh_prep
   dh_installdocs
   dh_installchangelogs
   dh_installinit
   dh_installsystemd
   dh_perl
   dh_link
   dh_strip_nondeterminism
   dh_compress
   dh_fixperms
   dh_missing
   dh_strip
   dh_makeshlibs
   dh_shlibdeps
   dh_installdeb
   dh_gencontrol
dpkg-gencontrol: warning: Depends field of package test: substitution variable ${shlibs:Depends} used, but is not defined
   dh_md5sums
   dh_builddeb
dpkg-deb: building package 'test' in '../test_1.0.0-1_amd64.deb'.
 dpkg-genbuildinfo --build=binary
 dpkg-genchanges --build=binary >../test_1.0.0-1_amd64.changes
dpkg-genchanges: info: binary-only upload (no source code included)
 dpkg-source --after-build .
dpkg-buildpackage: info: binary-only upload (no source included)

$ tree
.
└── debian
    ├── changelog
    ├── compat
    ├── control
    ├── copyright
    ├── debhelper-build-stamp
    ├── files
    ├── manpage.1.ex
    ├── manpage.sgml.ex
    ├── manpage.xml.ex
    ├── menu.ex
    ├── postinst.ex
    ├── postrm.ex
    ├── preinst.ex
    ├── prerm.ex
    ├── README.Debian
    ├── README.source
    ├── rules
    ├── source
    │   └── format
    ├── test
    │   ├── DEBIAN
    │   │   ├── control
    │   │   ├── md5sums
    │   │   ├── postinst
    │   │   ├── postrm
    │   │   └── prerm
    │   ├── lib
    │   │   └── systemd
    │   │       └── system
    │   │           └── test.service
    │   └── usr
    │       └── share
    │           └── doc
    │               └── test
    │                   ├── changelog.Debian.gz
    │                   ├── copyright
    │                   └── README.Debian
    ├── test.cron.d.ex
    ├── test.doc-base.EX
    ├── test-docs.docs
    ├── test.postrm.debhelper
    ├── test.service
    ├── test.substvars
    └── watch.ex

11 directories, 34 files
```

&nbsp;


# 5. Remove useless files and add custom files

```sh
$ cp -r debian/test ../

$ cd ../test

$ tree
.
├── DEBIAN
│   ├── control
│   ├── md5sums
│   ├── postinst
│   ├── postrm
│   └── prerm
├── lib
│   └── systemd
│       └── system
│           └── test.service
└── usr
    └── share
        └── doc
            └── test
                ├── changelog.Debian.gz
                ├── copyright
                └── README.Debian

8 directories, 9 files

$ mkdir -p usr/local/bin

$ cp /path/to/file/get_ip usr/local/bin/

$ mkdir -p etc/

$ cp /path/to/file/itfname etc/

$ tree
.
├── DEBIAN
│   ├── control
│   ├── md5sums
│   ├── postinst
│   ├── postrm
│   └── prerm
├── etc
│   └── itfname
├── lib
│   └── systemd
│       └── system
│           └── test.service
└── usr
    ├── local
    │   └── bin
    │       └── get_ip
    └── share
        └── doc
            └── test
                ├── changelog.Debian.gz
                ├── copyright
                └── README.Debian

11 directories, 11 files
```

&nbsp;


# 6. Packaging

```sh
$ cd ..

$ dpkg -b ./test test-final_1.0.0-1_amd64.deb
dpkg-deb: building package 'test' in 'test-final_1.0.0-1_amd64.deb'.
```

&nbsp;


# 7. Verify

```sh
$ sudo dpkg -i test-final_1.0.0-1_amd64.deb
Selecting previously unselected package test.
(Reading database ... 90424 files and directories currently installed.)
Preparing to unpack test-final_1.0.0-1_amd64.deb ...
Unpacking test (1.0.0-1) ...
Setting up test (1.0.0-1) ...
Created symlink /etc/systemd/system/multi-user.target.wants/test.service → /lib/systemd/system/test.service.

$ ls -l /lib/systemd/system/test.service
-rw-r--r-- 1 jinaijun jinaijun 211 Oct 25 19:31 /lib/systemd/system/test.service

$ ls -l /usr/local/bin/get_ip
-rwxr-xr-x 1 jinaijun jinaijun 2109445 Oct 25 19:35 /usr/local/bin/get_ip

$ ls -l /etc/itfname
-rw-r--r-- 1 jinaijun jinaijun 4 Oct 25 19:35 /etc/itfname

$ dpkg -l | grep test
...
ii  test                                 1.0.0-1                         amd64        <insert up to 60 chars description>

$ dpkg -L test
/.
/etc
/etc/itfname
/lib
/lib/systemd
/lib/systemd/system
/lib/systemd/system/test.service
/usr
/usr/local
/usr/local/bin
/usr/local/bin/get_ip
/usr/share
/usr/share/doc
/usr/share/doc/test
/usr/share/doc/test/README.Debian
/usr/share/doc/test/changelog.Debian.gz
/usr/share/doc/test/copyright

$ sudo dpkg --purge test
(Reading database ... 90431 files and directories currently installed.)
Removing test (1.0.0-1) ...
Purging configuration files for test (1.0.0-1) ...

```

&nbsp;


# 8. References

1. [Introduction to Debian Packaging](https://wiki.debian.org/Packaging/Intro)
2. [DEB打包方法简介](https://www.small09.top/posts/211019-dpkg_package_guide/)
3. [Systemd 添加自定义服务(开机自启动)](https://www.cnblogs.com/jhxxb/p/10654554.html)

&nbsp;

