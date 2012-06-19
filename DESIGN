
# LIST OF TOPICS #

* What are the design goals?
* Which HAI script does what?
* Why doesn't HAI resolve 'provider packages'?
* What packages don't work in HAI?
* How are packages stored?
* Why can't HAI install old package versions?
* How does HAI download packages?
* HAI compared to similar programs



# WHAT ARE THE DESIGN GOALS? #

The historical use case was to allow using Arch Linux programs on other
distributions without a root account.

HAI was completely rewritten multiple times during the development process.
Thus it must be compact (to make such rewrites easy) and elegant (to make them
unnecessary).

HAI has to work *now*, not *eventually*, so it has to use an existing package
format. Arch Linux packages were chosen for a variety of reasons: the database
is text-based, the build scripts are simple, the AUR is large, and the
maintainers have a healthy dislike for distro-specific patches.

To make HAI reasonably fast, it should use precompiled binary packages by
default, but also be able to compile packages from source if the user says so.



# WHICH HAI SCRIPT DOES WHAT? #

**`bootstrap`**: installs HAI's dependencies missing on the base system.

**`hai`**: calls `get` to fulfill all requests (and their dependencies), then
creates a fake root directory with unionfs/fakechroot to run the program in.

**`get`**: finds and downloads/builds a package that satisfies a request.
receives two arguments - the request and a list of packages that were already
selected to be in the union. writes the directory name to be added to unionfs
to stdout, or nothing if nothing needs to be selected.

**`build`**: reads a PKGBUILD, uses `hai` to get its dependencies and run
`makepkg` to build it. `get` calls this when it downloads things from AUR.
Sourcing `PKGBUILD` directly from `get` would pollute its global environment.

From a power user's point of view, the most interesting part is `get`. It
defines some functions and calls `DOWNLOAD_STRATEGIES` from the config, which
means the user can completely change its behavior. For example, you could
change `DOWNLOAD_STRATEGIES` to download packages from another distro and
generate `.PKGINFO` files on its own. The request is stored in `$requestname`
and `version-ok` can be used to query if a particular version is OK - see the
existing strategies for sample code.



# WHY DOESN'T HAI RESOLVE 'PROVIDER PACKAGES'? #

HAI doesn't resolve `%PROVIDES%` and `%REPLACES%` dependency rules
automatically - you have to specify what to install every time. (For example,
if a package requires `java-environment` and Java isn't in the local system,
you should specify a provider package, e.g. `openjdk6`, before it.) This is
because:

* Pacman can remember "installed packages" and thus if you install a package
  that provides something, it'll be used the next time as well. HAI doesn't
  have "installed packages", only packages in cache.
* HAI intentionally doesn't make any decisions depending on what's in cache
  (only whether to skip a download), so it can't use the cache for remembering
  the user's preferred packages either.
* Pacman chooses the first provider package it finds. HAI could do that, but
  it doesn't, because it would be difficult, slow and not what you want most
  of the time. (Why slow: the slowest part of extracting sync databases is
  actually creating a lot of files on disk. HAI currently avoids that and
  extracts just the package info it needs, but that's hard to do when you need
  to parse `%PROVIDES%`.)

You can use `MAP_REQUESTS` in the config file to work around this. For example
`MAP_REQUESTS=(java-environment:openjdk6)` means that instead of trying to get
`java-environment` (and failing, because there's no such package), HAI will
get `openjdk6` - which is just what you wanted.



# WHAT PACKAGES DON'T WORK IN HAI? #

* Some packages have scripts that run when they're installed or removed. HAI
  doesn't run these scripts.
* Some packages provide system services in `/etc/rc.d`. As they aren't in the
  real `/etc`, you won't be able to list them in `/etc/rc.conf`.
* Integration with the desktop environment is limited. HAI doesn't track
  what's installed and what isn't, only what's currently downloaded in the
  cache. Thus HAI can't provide mime type bindings or start menu items well.
  (However, you could manually create one that calls `hai ...`.)
* Some packages break out of HAI's sandbox, by e.g. setting `$LD_PRELOAD` to
  another value instead of appending to it.
* Packages might break for all sorts of other mysterious reasons.



# HOW ARE PACKAGES STORED? #

In the original design, packages were stored as .pkg.tar.* files and unpacked
on union creation. This was too slow. Now packages are stored in the cache
unpacked, in directories named like the archives, but without the extension.

The .pkg.tar.* files contain a .PKGINFO file with the package's information.
This file is unpacked into the cache as well.



# HOW DOES THE FAKE ROOT WORK? #

The program has to believe it's installed in `/usr` as usual. There are two
reasons: First, we want to save on compilation time and use pre-made binaries,
which assume they're installed in `/usr`. Second, if HAI compiled programs
with `--prefix=$HOME/.hai/something`, the user wouldn't be able to move
`~/.hai` somewhere else. (Right now, you can just move the directory and
change the config file and everything will still work.)

Even though most programs today can change where they're installed, the path
is still compiled into the binary. The program can't be completely moved to
another directory, it would still look for its files in the old location.

`hai` uses `unionfs-fuse` to put all the package directories it selects and
the standard root directory into an union filesystem in `~/.hai/unions`.
Then the requested command is ran chrooted in the directory. To simulate the
effects of chroot without root privileges, HAI uses fakechroot - a LD_PRELOAD
hack that wraps glibc I/O functions.

It isn't hard to get out of fakechroot, even by mistake (such as setting
LD_PRELOAD to a different value instead of appending to it). This is
unfortunate, but it's not a problem that often. In the future, HAI might
investigate other alternatives, for example fakeroot-ng.



# WHY CAN'T HAI INSTALL OLD PACKAGE VERSIONS? #

Because Arch Linux can't either.

In fact, nobody can - not reliably, at least.

If you want an old version of a package to keep working, you have to keep
compatible library versions around. The best way to accomplish this is to
rename library packages every time the API versions change, and strictly
enforce that versions of the same package are always 100% backwards
compatible. (0install got this right. It's a pity they don't have more
packages.)

Most distros try to do this - Debian, for example, appends the API version to
all library names. So you can have both `libfreetype1` and `libfreetype6`
installed if some packages depend on them. This breaks rather magnificently
when it comes to packages that don't make a habit of publishing compatibility
breaking API changes, such as most Python/Perl/... libraries.

The developers of Arch worked around the whole problem by removing version
information from packages and saying, "you either update everything, or
nothing at all". And actually, it turns out Debian is the same: you either
update everything (on testing/unstable) or nothing at all (on stable).

Can something be done about this?

Not all packages publish their API changes, so your best options probably are
time-based snapshots - i.e. when a particular system works and all of its
packages are up-to-date at the time, bundle them together. (`pacsnap` attempts
to do this for Arch Linux, though we haven't tested it at this time.)

HAI's situation is a bit more difficult still, because it has to coordinate
packages in its cache and packages on the local system (which it can't
control). However, a package bundle creating tool is a potential road for
HAI's future development.



# HOW DOES HAI DOWNLOAD PACKAGES? #

The HAI download system uses a worse-is-better approach that attempts to work
reasonably well while being as simple as possible, even at the cost of some
functionality.

Instead of parsing a mirror list, HAI just connects to `mirrors.kernel.org`,
which has worldwide mirrors and synchronizes with `www.archlinux.org` quite
quickly anyway. However, this is, of course, completely configurable.

Downloading and parsing a sync database is too much work for too little
reward. `<rant>` Seriously, does the `desc`/`depends` format have *any*
advantages over `.PKGINFO`? It's split into two files, it can't be grepped,
and you have to implement two parsers instead of one. `</rant>`

Instead of downloading, unpacking and parsing the sync databases, HAI just
downloads the directory indexes (`index.html`) from the mirrors and uses one
grep expression to check for available package versions and their file
extensions.

This has some disadvantages - most notably, you lose checksums and you lose
`%PROVIDES%`. Patches to switch to the sync database are welcome.



# HAI COMPARED TO SIMILAR PROGRAMS #

HAI faces the most competition from `klik` and Zero Install.

`klik`'s main motto is "1 App = 1 file". This is convenient for users, but
duplicates a lot of libraries. HAI's design goals differ a lot. `klik` also
feels ugly, like a quickly thrown-together hack, and not just the coding style
(see for example the `install` file), but the architecture as well. `klik`
doesn't really care about dependencies (it's assumed the user has a "base
system" installed without really specifying what the base system is).

`klik2` looks promising, but it's been stuck in development for ages.

Zero Install is much more similar to HAI. Programs are kept separate from
libraries and a small runtime is used to download all the dependencies. Zero
Install suffers from a lack of packages available in its custom format.
However, its design is amazingly elegant and provided a lot of inspiration
for HAI.

GoboLinux is what originally sparked the idea for this project. The cache
directory's design is similar to Gobo's `/Programs` directory.

Both klik and Zero Install have feature comparison matrices:

* http://www.0install.net/matrix.html
* http://klik.atekon.de/wiki/index.php/Comparison

This is what's missing from HAI (i.e. where HAI gets a "NO" answer):

* "Downloads shared between users", because HAI avoids system-wide effects.
* "Multiple versions coexist" - see above.
* "Digital signatures"
* "Decentralised" - you can add download sources/strategies, but it's not
  truly decentralised like Zero Install is.
* "One app = one file". It could be done with existing HAI, but the design
  isn't well suited for it.
* "Core system suitable" (as in, you can't install the kernel with HAI)

HAI fares quite well, but of course, there's still room for improvement, and
patches are always welcome :-)

