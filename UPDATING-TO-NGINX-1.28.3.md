# Updating `lib-nginx` to nginx 1.28.3

This note documents the process used to analyze the existing `lib-nginx` port,
compare it with upstream nginx `1.28.3`, and update the port metadata and build
description accordingly.

## Goal

Update the Unikraft nginx application-library port from nginx `1.15.6` to nginx
`1.28.3` while preserving the existing Unikraft integration model:

* nginx is treated as an application library
* upstream source is fetched from `nginx.org`
* local compatibility patches are applied
* Unikraft builds nginx through `Config.uk`, `Makefile.uk`, generated config
  headers, and `ngx_modules.c`

## Files That Define the Port

The existing port was analyzed in the following order:

1. `Config.uk`
   This defines the KConfig surface, optional nginx modules, and the Unikraft
   dependencies selected when nginx is enabled.
2. `Makefile.uk`
   This registers the component, fetches upstream nginx, applies patches,
   declares include paths and flags, and enumerates source files.
3. `main.c`
   This provides the optional wrapper entry point used when the port exports a
   `main()` function for a Unikraft app.
4. `include/ngx_auto_config.h` and `include/ngx_auto_headers.h`
   These replace parts of nginx's normal configure-time output with
   port-maintained headers.
5. `ngx_modules.c`
   This is the static module table used by the port instead of nginx's native
   generated module list.
6. `patches/`
   These are the local adaptations that make upstream nginx fit the Unikraft
   runtime model.

## Upstream Analysis

The extracted upstream source tree `nginx-1.28.3/` was used as the reference.

The first comparison step was mechanical:

* identify the current ported version in `Library.uk` and `Makefile.uk`
* compare every source path listed in `Makefile.uk` against files that actually
  exist in `nginx-1.28.3/src`
* compare every module symbol referenced in `ngx_modules.c` against real module
  definitions found in nginx `1.28.3`
* check whether the existing local patches still apply to nginx `1.28.3`

This quickly separated the work into two categories:

* changes caused by upstream file/module layout drift
* pre-existing issues in the port that happened to become visible during the
  version bump

## What Was Checked

### 1. Versioned metadata

The old port still targeted nginx `1.15.6` in:

* `Library.uk`
* `Makefile.uk`

These needed to be updated to `1.28.3`.

### 2. Patch applicability

The three existing patches were tested against nginx `1.28.3` and all of them
still applied cleanly:

* `0001-Rename-main-and-force-setproctitle-to-be-a-nop.patch`
* `0002-pthread-embedded-does-not-allow-semaphores-shared-be.patch`
* `0003-Initialize-nginx-cached-time-structures.patch`

This was a strong signal that the port did not need to be redesigned from
scratch.

### 3. Source list drift

A scripted check was used to verify whether each source file referenced by
`Makefile.uk` still exists in `nginx-1.28.3/src`.

The main differences found were:

* `http/v2/ngx_http_v2_huff_decode.c` no longer exists
* `http/v2/ngx_http_v2_huff_encode.c` no longer exists
* `http/modules/ngx_http_status_module.c` no longer exists
* `http/modules/ngx_http_perl_module.c` moved under `http/modules/perl/`
* `stream/ngx_stream_upstream_module.c` no longer exists

Then nginx's own `auto/modules` and `auto/sources` logic was consulted to see
how upstream now composes those features.

### 4. Module table drift

`ngx_modules.c` was compared against real module declarations in nginx `1.28.3`.

That exposed:

* `ngx_http_status_module` should now be `ngx_http_stub_status_module`
* OpenSSL support in modern nginx also defines `ngx_openssl_cache_module`
* some config guards in `ngx_modules.c` no longer matched the names used in
  `Config.uk`

### 5. Generated header expectations

nginx `1.28.3` was configured natively once, using a close approximation of the
options encoded in the port. The goal was not to adopt nginx's native build
system, but to inspect the generated `objs/ngx_auto_config.h`,
`objs/ngx_auto_headers.h`, and `objs/ngx_modules.c` as a reference for modern
upstream expectations.

This confirmed, among other things:

* HTTP/2 huffman sources are now shared from `src/http/`
* stream support now includes additional sources such as
  `ngx_stream_variables.c`, `ngx_stream_script.c`, `ngx_stream_handler.c`,
  `ngx_stream_upstream.c`, and `ngx_stream_upstream_round_robin.c`
* OpenSSL support now includes `ngx_event_openssl_cache.c`

The temporary `Makefile` and `objs/` generated for this inspection were removed
afterward.

## Changes Made To the Port

### Metadata updates

`Library.uk`

* updated version from `1.15.6` to `1.28.3`
* added the SHA-256 checksum for the official
  `https://nginx.org/download/nginx-1.28.3.tar.gz` archive

`Makefile.uk`

* updated `LIBNGINX_VERSION` from `1.15.6` to `1.28.3`

### Source-list updates

The port's explicit source list in `Makefile.uk` was updated to match nginx
`1.28.3`:

* added `src/event/ngx_event_openssl_cache.c`
* replaced HTTP/2 huffman sources with:
  * `src/http/ngx_http_huff_decode.c`
  * `src/http/ngx_http_huff_encode.c`
* corrected the Perl module path to:
  * `src/http/modules/perl/ngx_http_perl_module.c`
* updated stream support to include:
  * `src/stream/ngx_stream_variables.c`
  * `src/stream/ngx_stream_script.c`
  * `src/stream/ngx_stream_handler.c`
  * `src/stream/ngx_stream_upstream.c`
  * `src/stream/ngx_stream_upstream_round_robin.c`

### Corrections to stale or incorrect entries

The upgrade process also uncovered a few old inconsistencies in the port:

* `LIBNGINX_HTTP_REALIP` incorrectly pointed at
  `ngx_http_limit_req_module.c`; it now points at
  `ngx_http_realip_module.c`
* `LIBNGINX_HTTP_STUB_STATUS` is now used consistently with
  `ngx_http_stub_status_module.c`
* `CONFIG_LIBNGINX_HTTP_SPLIT_CLENTS` was a typo and was corrected to
  `CONFIG_LIBNGINX_HTTP_SPLIT_CLIENTS`
* `CONFIG_LIBNGINX_UPSTREAM` in `ngx_modules.c` was corrected to
  `CONFIG_LIBNGINX_HTTP_UPSTREAM`
* `CONFIG_LIBNSSL` in `ngx_modules.c` was corrected to `CONFIG_LIBSSL`

### Module-table updates

`ngx_modules.c` was updated so that the port's static module registration matches
nginx `1.28.3`:

* `ngx_http_status_module` was replaced with `ngx_http_stub_status_module`
* `ngx_openssl_cache_module` was added alongside `ngx_openssl_module`
* stale config guards were aligned with the symbols defined in `Config.uk`
