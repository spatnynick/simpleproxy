# simpleproxy (APC fork)

> **Why this fork?** To make simpleproxy work with **SAP ABAP Push Channels
> (APC)**. APC TCP sockets need every message to end with a terminator (or
> have a fixed length). Upstream simpleproxy can't add or remove one. This
> fork adds the `-8` option, which translates CRLF-terminated messages on the
> client side to unterminated data on the remote side, and back.

This is a fork of [vzaliva/simpleproxy](https://github.com/vzaliva/simpleproxy),
a small TCP proxy (see [README.txt](README.txt) and `man simpleproxy` for the
general documentation). It is based on upstream **v3.6** and pulls in upstream
fixes. The APC changes are maintained only in this fork and are not submitted
upstream.

## Why this fork exists

SAP ABAP Push Channels over plain TCP need *message framing*: each message
has to end with a terminator, or have a fixed length. Upstream simpleproxy
forwards the byte stream as is, so it can't sit between an APC endpoint and a
TCP service that doesn't use such a terminator.

The new `-8` option adds that framing, so simpleproxy can be used with ABAP
Push Channels. Please read the [limitations](#limitations) before relying on
it: the terminator is handled per TCP read, not per message.

## What `-8` does

The terminator is `CRLF` (`\r\n`, `0x0D 0x0A`). It is set at compile time
(`APC_TERMINATOR` in `simpleproxy.c`).

| Direction | Action |
|---|---|
| client → remote (`-L` side to `-R` side) | a trailing `CRLF` is **removed** before forwarding |
| remote → client (`-R` side to `-L` side) | a `CRLF` is **appended** before forwarding |

So the peer that connects to simpleproxy (for example an ABAP program using an
APC TCP client with a CRLF terminator) sees terminator-framed messages. The
remote server sees the messages without terminators.

Example:

```sh
simpleproxy -8 -L 5000 -R device.example.com:6000
```

Config-file equivalent (see `sample.cfg`):

```
APCTerminator   yes
```

### Limitations

These are also described in `simpleproxy -h`, the man page, and the comment
above `APC_TERMINATOR` in `simpleproxy.c`.

TCP is a byte stream, so simpleproxy can't see message boundaries. `-8` works
on each `read()` chunk:

* **remote → client**: a `CRLF` is appended after every chunk read from the
  remote host. A remote reply that arrives in several TCP segments, or is
  larger than the buffer (80 KB), gets a `CRLF` after each piece. Replies that
  arrive in one piece (the normal case for short request/response traffic)
  get exactly one terminator.
* **client → remote**: only a `CRLF` at the very end of a chunk is removed.
  If several messages arrive in one chunk (`A\r\nB\r\n`), only the last
  terminator is removed. If a `CRLF` is split across two chunks, it is
  forwarded unchanged. A chunk that is exactly `CRLF` (an empty message) is
  forwarded unchanged.
* `-8` is ignored together with `-u` (HTML probe) or `-A` (HTTP auth), which
  use a different code path. simpleproxy prints a warning in that case.
* The trace file (`-t`) records data as received, before the terminator is
  added or removed.

## Differences from upstream

| Area | Upstream v3.6 | This fork |
|---|---|---|
| `-8` option / `APCTerminator` cfg key | — | APC CRLF terminator handling (see above) |
| `MBUFSIZ` (I/O buffer) | 8192 | 81920 (larger APC replies in one read) |
| `<syslog.h>`, `<fcntl.h>` | behind `HAVE_*_H` checks | always included (the code needs them anyway; keeps IDE indexers such as Eclipse working) |
| `getnameinfo()` for the client address | passes an uninitialized length (bug) | passes the `accept()` address length |
| Version string (`-V`) | `v3.6` | `v3.6-apc` |

All other behaviour, including every other option, is the same as upstream.
Without `-8`, the fork behaves like upstream, apart from the larger buffer and
the `getnameinfo()` fix.

## Syncing with upstream

```sh
git remote add upstream https://github.com/vzaliva/simpleproxy.git
git fetch upstream
git merge upstream/master
```

## Building

```sh
./configure
make
make install
```

License: GPLv2, see [LICENSE.txt](LICENSE.txt).
