# Home Arch Install #
HAI can install and run Arch Linux programs from your
normal user account. Even if you don't run Arch Linux at all.

HAI's design is heavily inspired by Zero Install, but it uses Arch Linux
packages instead of a new package format.

HAI follows the KISS and worse-is-better philosophies. HAI is implemented as a
set of specialized bash scripts, with under 500 lines of code in total, so
it's not that hard to read, understand and extend.

## Installation ##
HAI doesn't need system-wide installation. Just unpack it somewhere.

The only HAI script you need to run is `hai`. If you want to type `hai`
instead of `some/long/path/to/hai`, put appropriate alias in your shell config.

Some programs HAI uses aren't commonly installed:

* [fakechroot](https://github.com/fakechroot/fakechroot/wiki): pretends some directory is actually the filesystem root.
* [unionfs-fuse](http://podgorny.cz/moin/UnionFsFuse): merges directories (creates an union) without root privileges.
* [vercmp](https://www.archlinux.org/pacman/): compares versions. (unfortunately majority of distributions haven't pacman in repositories)

Because the point of `hai` is avoiding changes to the local system, you can
use `bootstrap` to install these programs into `~/.hai/bootstrap`. It won't do
anything if you already have them. (You need FUSE and its header files to
compile unionfs-fuse, though.)

## Usage ##

You tell HAI what packages to install and what program to run by running the
`hai` command. `hai` doesn't really install the packages - it uses fakechroot
and unionfs-fuse to fool the program into thinking it's installed in `/usr`.

Try using HAI to run [Inkscape](http://inkscape.org/) or [Sound Converter](http://soundconverter.org/):

    hai inkscape
    hai soundconverter

The general form is

    hai [--upgrade] <packages to install> -- <command to run>

(`hai foo` is the same as `hai foo -- foo`, so if the package is called the
same as the program, you don't have to write it twice.)

HAI only installs packages it considers up-to-date (unless you put an older
version in the `local` override directory, see the config file). However,
it doesn't refresh the package list unless you say `hai --upgrade`. This is
so that you can use `hai` even when offline.

In pacman terms, `--upgrade` is similar to `-Sy`. However,
`--upgrade` itself only removes the old package lists. New package lists and
packages themselves are lazily downloaded when they're needed.

You should use `--upgrade` as often as possible, to avoid the programs you
downloaded with HAI getting out of sync with the libraries on the base system.

### Remove unused packages ###
HAI touches (i.e. sets modification time of) packages it uses. This allows you
to, for example, find all the packages that weren't used in the last 30 days
and remove them from HAI's cache:

    find ~/.hai/cache/* -maxdepth 0 -mtime +30 -print0 | xargs -0 rm -r

## Configuration ##
The default configuration file is `config` in HAI's directory. You can put
your own configuration in `~/.hai/config`, or set `$HAICONFIG` to another
location. It's recommended to read the config and see what can be changed.

By default, HAI stores everything in `~/.hai`, but you can always change that
in the config file.

Note that out of all directories in `~/.hai`, only `local` contains your
personal data. All the others (`cache`, `sync` etc.) can be recreated if lost.
