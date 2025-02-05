## lr: list files, recursively

`lr` is a new tool for generating file listings, which includes the
best features of `ls(1)`, `find(1)` and `du(1)`.

## Benefits

Over find:
* friendly and logical C-style filter syntax
* getopt is used, can mix filters and arguments in any order
* can sort
* compute directory sizes
* can strip leading `./`

Over ls:
* sorts over all files, not per directory
* copy & paste file names from the output since they are relative to pwd
* ISO dates
* powerful filters

## Rosetta stone

* `ls`: `lr -1 | column`
* `find .`: `lr` (or `lr -U` for speed.)
* `ls -l`: `lr -1l`
* `ls -ltrc`: `lr -l1Aoc`
* `find . -name '*.c'`: `lr -t 'name ~~ "*.c"'`
* `find . -regex '.*c': `lr -t 'path =~ "c$"'`
* `find -L /proc/*/fd -maxdepth 1 -type f -links 0 -printf '%b %p\n'`:
`lr -UL1 -t 'type == f && links == 0' -f '%b %p\n' /proc/*/fd`
* `find "${@:-.}" -name HEAD -execdir sh -c 'git rev-parse --resolve-git-dir . >/dev/null 2>/dev/null && pwd' ';'`: `lr -0t 'name == "HEAD"' "$@" | xapply -fz 'cd %Q[1/-$] && git rev-parse --resolve-git-dir . >/dev/null && pwd' - 2>/dev/null`

## Usage:

	lr [-0|-F|-l|-f FMT] [-D] [-H|-L] [-1AQdhsx] [-U|-o ORD] [-t TEST]* PATH...

* `-0`: output filenames seperated by NUL bytes (implies `-Q`).
* `-F`: output filenames and an indicator of their file type (`*/=>@|`).
* `-l`: long output ala `ls -l`.
* `-f FMT`: custom formatting, see below.
* `-D`: depth first traversal. `prune` does not work, but `entries`
  and `total` is computed.
* `-H`: only follow symlinks on command line.
* `-L`: follow all symlinks.
* `-1`: don't go below one level of directories.
* `-A`: don't list files starting with a dot.
* `-Q`: don't shell quote file names.
* `-d`: don't enter directories.
* `-h`: print human readable size for `-l` (also `%s`).
* `-s`: don't print leading `./`.
* `-x`: don't enter other filesystems.
* `-U`: don't sort results.
* `-o ORD`: sort according to the string `ORD`, see below.
* `-t TEST`: only show files matching all `TEST`s, see below.

## Output formatting:

* `\a`, `\b`, `\f`, `\n`, `\r`, `\v`, `\0` as in C.
* `%%`: plain `%`.
* `%s`: file size in bytes.
* `%S`: file size, with human readable unit.
* `%b`: file size in 512-byte blocks.
* `%k`: file size in 1024-byte blocks.
* `%d`: path depth.
* `%D`: device number (`stat.st_dev`).
* `%i`: inode number.
* `%I`: one space character for every depth level.
* `%p`: full path (without `./` if `-s`).
* `%l`: symlink target.
* `%n`: number of hardlinks.
* `%F`: file indicator type symbol (`*/=>@|`).
* `%f`: file basename (everything after last `/`).
* `%Ax`, `%Cx`, `%Tx`: result of `strftime` for `%x` on atime/ctime/mtime.
* `%m`: octal file permissions.
* `%M`: ls-style symbolic file permissions.
* `%y`: ls-style symbolic file type (`bcdfls`).
* `%g`: group name.
* `%G`: numeric gid.
* `%u`: user name.
* `%U`: numeric uid.
* `%e`: number of entries in directories (only with `-D`).
* `%t`: total size used by accepted files in directories (only with `-D`).
* `%Y`: type of the filesystem the file resides on.

## Sort order

Sort order is string consisting of the following letters.
Uppercase letters reverse sorting.
E.g. `Sn` sorts first by size, smallest last, and then by name (in
case sizes are equal).

Default: `n`.

* `a`: atime.
* `c`: ctime.
* `d`: path depth.
* `e`: file extension.
* `i`: inode number.
* `m`: mtime.
* `n`: file name.
* `s`: file size.
* `t`: file type.  This sorts all directories before other files.

## Filter expressions

`lr` filters are given by the following EBNF:

	<expr>     ::= <expr> || <expr>  -- disjunction
	             | <expr> && <expr>  -- conjunction
	             | ! <expr>          -- negation
	             | ( <expr )
	             | <numprop> <numop> <num>
	             | <strprop> <strop> <str>
	             | <typetest>
	             | <modetest>
	             | prune             -- do not traverse into subdirectories
	             | print             -- always true value
	
	<numprop>  ::= atime | ctime | depth | dev | entries | gid
	             | inode | links | mode | mtime | size | total | uid
	
	<numop>    ::= <= | < | >= | > | == | !=
	
	<num>      ::= [0-9]+ ( c        -- *1
	                      | b        -- *512
	                      | k        -- *1024
	                      | M        -- *1024*1024
	                      | G        -- *1024*1024*1024
	                      | T )?     -- *1024*1024*1024*1024
	
	<strprop>  ::= fstype | group | name | path | target | user
	
	<strop>    ::= ==                -- string equality
	             | ===               -- case insensitive string equality
	             | ~~                -- glob (fnmatch)
	             | ~~~               -- case insensitive glob (fnmatch)
	             | =~                -- POSIX Extended Regular Expressions
	             | =~~               -- case insensitive POSIX Extended Regular Expressions
	
	<str>      ::= " ([^"] | "")+ "  -- use "" for a single " inside "

	<typetest> ::= type == ( b | c | d | p | f | l )

	<modetest> ::= mode ( ==         -- exact permissions
	                    | &          -- check if all bits of <octal> set
	                    | |          -- check if any bit of <octal> set
	                    ) <octal>
	
	<octal> ::= [0-7]+

## EWONTFIX

The following features won't be implemented:

* `-exec`: use `-0` and `xargs`
  (or even better [xa](https://github.com/chneukirchen/xa)).
* columns: use `column`, `git-column`, Plan 9 `mc`.

## "Screenshots"

Default output, sorted by name:

```
% lr
./
.git/
.git/HEAD
.git/config
[...]
Makefile
README.md
lr.c
```

Long output format:

```
% lr -l
drwxrwxr-x 3 chris users   120 2015-10-27 13:56 ./
drwxrwxr-x 7 chris users   240 2015-10-27 13:56 .git/
-rw-rw-r-- 1 chris users    23 2015-10-27 13:56 .git/HEAD
-rw-rw-r-- 1 chris users   257 2015-10-27 13:56 .git/config
[...]
-rw-rw-r-- 1 chris users   297 2015-10-27 13:56 Makefile
-rw-rw-r-- 1 chris users  5828 2015-10-27 13:56 README.md
-rw-rw-r-- 1 chris users 27589 2015-10-27 13:56 lr.c
```

Simple test:

```
% lr -t 'type == d'
./
.git/
.git/hooks/
.git/info/
.git/logs/
.git/logs/refs/
.git/logs/refs/heads/
.git/logs/refs/remotes/
.git/logs/refs/remotes/origin/
.git/objects/
.git/objects/info/
.git/objects/pack/
.git/refs/
.git/refs/heads/
.git/refs/remotes/
.git/refs/remotes/origin/
.git/refs/tags/
```

List regular files by size, largest first:

```
% lr -f '%S %f\n' -1 -t 'type == f' -oS
  27K lr.c
 5.7K README.md
  297 Makefile
```

List directory total sizes, indented:

```
% lr -D -t 'type == d' -f '%I%I%t %p\n'
172 .
  132 .git
    40 .git/hooks
    4 .git/info
    12 .git/logs
      8 .git/logs/refs
        4 .git/logs/refs/heads
        4 .git/logs/refs/remotes
          4 .git/logs/refs/remotes/origin
    48 .git/objects
      0 .git/objects/info
      48 .git/objects/pack
    8 .git/refs
      4 .git/refs/heads
      4 .git/refs/remotes
        4 .git/refs/remotes/origin
      0 .git/refs/tags
```

## Installation

Use `make all` to build, `make install` to install relative to `PREFIX`
(`/usr/local` by default).  The `DESTDIR` convention is respected.
You can also just copy the binary into your `PATH`.

## Copyright

Copyright (C) 2015 Christian Neukirchen <purl.org/net/chneukirchen>

Licensed under the terms of the MIT license, see lr.c.
