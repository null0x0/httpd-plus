# httpd-plus

Add-ons for the [OpenBSD](https://www.openbsd.org) [`httpd(8)`](http://man.openbsd.org/httpd) web server applicable to the latest `-current` branch.

Other branches available:
* [7.1-stable](https://github.com/mpfr/httpd-plus/tree/7.1-stable)
* [7.0-stable](https://github.com/mpfr/httpd-plus/tree/7.0-stable)

Notes:
* The former [`location-access-tests`](https://github.com/mpfr/httpd-plus/blob/cb4a396d7fa8426a353e6a1b91887fc544f525f8/02-location-access-tests.patch) add-on has been [imported](https://github.com/openbsd/src/commit/e96b74b9e3e44aa22060826006547b90ccc38faa#diff-ed9bfab4d87ea6df040a9696cb1860f82d56e70486351f950b3fca91eab7175d) into `-current`.
* Based on the former [`custom-error-documents`](https://github.com/mpfr/httpd-plus/blob/ed8d5e2eed2236cf3f7d3043992a924aeee81475/06-custom-error-documents.patch) add-on, the named functionality has been [added](https://github.com/openbsd/src/commit/cbced0bd4e14972a40a5dcfce5bb001f5654350f#diff-ed9bfab4d87ea6df040a9696cb1860f82d56e70486351f950b3fca91eab7175d) to `-current`.

## List of add-ons

### updates

Bug fixes:
* Failing `directory auto index` of `location` in case enclosing `server` specifies `directory no index` (see on [tech@](https://marc.info/?l=openbsd-tech&m=160293921708844&w=2))
* Failing location access test in case `server`/`location` `root` is empty (see on [tech@](https://marc.info/?l=openbsd-tech&m=160468404614852&w=2))
* Inconsistent handling of inaccessible `server`/`location` `root` (regular file access still returns status `404` instead of `500`)
* Inappropriate `log_warn` in case of missing custom error documents and cleanup of `read_errdoc` function (see on [misc@](https://marc.info/?l=openbsd-misc&m=163869316928263&w=2))

Amendments:
* The info regarding the usage of the former `location-access-tests` add-on with [WordPress](https://wordpress.org) was moved from this website to [`httpd.conf(5)`](https://mpfr.net/man/httpd-plus/current/httpd.conf.5.html#EXAMPLES).

### cache-control-headers

Optional HTTP `Cache-Control` headers via [`httpd.conf(5)`](https://mpfr.net/man/httpd-plus/current/httpd.conf.5.html#TYPES).

```
types {
	...
	image/jpeg  { cache "max-age=2592000, public" }             jpeg jpg
	text/css    { cache "max-age=86400, private" }              css
	text/html   { cache "no-store, no-cache, must-revalidate" } html
	...
}
```

### fastcgi-script-overrides

Definition of `script` overrides for `fastcgi` environments via [`httpd.conf(5)`](https://mpfr.net/man/httpd-plus/current/httpd.conf.5.html#script). This may be used either to run a dedicated `script` in its specific `param` environment for a certain `location`, or simply as a shortcut avoiding unnecessary evaluation rounds for the server (as required when using `request rewrite`).

```
server "www.example.com" {
	...
	location "/foobar/*" {
		fastcgi {
			socket "/run/php-fpm.sock"
			param "PARAM_1" "value_1"
			param "PARAM_2" "value_2"
			script "/index.php"
		}
	}
	location not found "/*" {
		# request rewrite "/index.php"
		fastcgi {
			socket "/run/php-fpm.sock"
			script "/index.php"
		}
	}
	...
}
```

### client-address-filters

Client address matching (`from` or `not from`) for `location` sections in [`httpd.conf(5)`](https://mpfr.net/man/httpd-plus/current/httpd.conf.5.html#location).

```
server "www.example.com" {
	...
	location "/intranet*" not from "10.0.0/24" { block }
	...
}
```

### notify-on-block

Send notification messages to UNIX-domain sockets for `server` and/or `location` sections with a `block` directive in [`httpd.conf(5)`](https://mpfr.net/man/httpd-plus/current/httpd.conf.5.html#message).

This cooperates perfectly with [pftbld(8)](https://github.com/mpfr/pftbld), offering an easy and straightforward means to effectively protect the web server from offensive clients and successively build customized firewall blocklists. In the example below, access to `/restricted*` URLs from outside the `10.0.0/24` network is not just blocked, but `httpd(8)` also reports client IP addresses to `pftbld(8)` (with its listening socket at `/var/www/run/pftbld-www.sock`) for further handling.

`httpd.conf`:

```
server "www.example.com" {
	...
	location "/restricted*" not from "10.0.0/24" {
		notify-on-block {
			socket "/run/pftbld-www.sock"
			message "$REMOTE_ADDR"
		}
		block
	}
	...
}
```

`pftbld.conf`:

```
target "www" {
	...
	socket "/var/www/run/pftbld-www.sock" {
		owner "www"
		group "www"
	}
	cascade {
		table "attackers"
		expire 1h
		...
	}
	...
}
```

### brace-expansion

Simple brace expansion for `alias <name>`, `include <path>` and `location <path>` option parameters in [`httpd.conf(5)`](https://mpfr.net/man/httpd-plus/current/httpd.conf.5.html#BRACE_EXPANSION).
Helps to minimize the configuration file size by avoiding duplicate content.

```
include "/etc/httpd-{0..5}-incl.conf"
...
server "www.example.com" {
	alias "www.{a,b,c}.example.com"
	...
	location "/*.{bmp,gif,ico,jpg,png}" { pass }
	...
}
```

## How to install

`httpd-plus` is a series of consecutive patch files which may be applied easily by following the steps below.

Make sure your user has sufficient `doas` permissions. To start, `cd` into the user's home directory, for example `/home/mpfr`.

```
$ cat /etc/doas.conf
permit nopass mpfr
$ cd
$ pwd
/home/mpfr
```

Download and extract patch files and installation script.

```
$ rm -rf httpd-plus-current/
$ ftp -Vo - https://codeload.github.com/mpfr/httpd-plus/tar.gz/current | tar xzvf -
httpd-plus-current
httpd-plus-current/00-updates.patch
httpd-plus-current/01-cache-control-headers.patch
httpd-plus-current/02-fastcgi-script-overrides.patch
httpd-plus-current/03-client-address-filters.patch
httpd-plus-current/04-notify-on-block.patch
httpd-plus-current/05-brace-expansion.patch
httpd-plus-current/LICENSE
httpd-plus-current/README.md
httpd-plus-current/install
```

Apply the patch files by running the installation script. This will build and install the `httpd-plus` binary. After the build process, the original source is restored.

```
$ doas ksh httpd-plus-current/install 2>&1 | tee httpd-plus-install.log
Backing up original sources ... Done.
Applying patch files ...
====================
===  00-updates  ===
====================
Hmm...  Looks like a unified diff to me...
The text leading up to this was:
--------------------------
.
.
.
done
==================================
===  01-cache-control-headers  ===
==================================
Hmm...  Looks like a unified diff to me...
.
.
.
done
Building and installing httpd-plus binary and manpage ...
.
.
.
Restoring original sources ... Done.

Installing httpd-plus binary and manpage completed successfully.
Please consult 'man httpd.conf' for further information on new features.
```

Adapt your `httpd.conf` for the newly added features. For further information, have a look at the updated `httpd.conf(5)` [manpage](https://mpfr.net/man/httpd-plus/current/httpd.conf.5.html) (also via `man httpd.conf`). Make sure your new configuration is valid.

```
$ doas vi /etc/httpd.conf
...
$ doas httpd -n
configuration OK
```

Restart the `httpd` daemon.

```
$ doas rcctl restart httpd
httpd(ok)
httpd(ok)
```

## How to uninstall

The original version of `httpd` can easily be restored by performing a fresh rebuild and reinstall.

```
$ cd /usr/src/usr.sbin/httpd
$ doas make obj
$ doas make clean
$ doas make
$ doas make install
```

Remove `httpd-plus` related features from your configuration file and make sure it is valid. Don't forget to restart the `httpd` daemon.

```
$ doas vi /etc/httpd.conf
...
$ doas httpd -n
configuration OK
$ doas rcctl restart httpd
httpd(ok)
httpd(ok)
```
