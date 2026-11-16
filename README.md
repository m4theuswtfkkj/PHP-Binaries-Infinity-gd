# PHP 8.5 (+FFI) build scripts for iAkunaMC — fork "Infinity"

This fork builds **only PHP 8.5** with **ext-ffi** for the **iAkunaMC** server software (Minecraft Bedrock, based on PocketMine-MP) and 3 targets:

| Target | Command | Type | CI runner |
|--------|---------|------|-----------|
| Linux x86_64 | `./compile.sh -t linux64 -j4 -P5 -D` | dynamic (.so + symlinks) | `ubuntu-22.04` |
| Linux arm64 **Cobalt** (Azure Cobalt 100 / Neoverse N2) | `./compile.sh -t linux-arm64-cobalt -j4 -P5 -D` | dynamic, **native** (build on an aarch64 host) | `ubuntu-22.04-arm` |
| Android arm64 | `./compile.sh -t android-aarch64 -x -j4 -P5 -D` | **static** (musl, cross-compile) | `ubuntu-22.04` |

Windows and macOS were removed. Other PHP versions (8.1/8.2/8.3/8.4) were removed too. Only PHP 8.5 is built for iAkunaMC.

## FFI and generated `php.ini`
- `libffi 3.4.8` is built together with the other libraries and PHP is configured with `--with-ffi`.
- `php.ini` is generated with `ffi.enable=true` (PHP's default is `preload`, which **does not** work from plugins).
- FFI is compiled into the binary: no `extension=ffi` is required.
- OPcache is enabled by default in the CLI (`opcache.enable_cli=1`); JIT is left off by default (`opcache.jit=off`) because it can cause crashes/unstable behavior, but can be enabled in `php.ini`.
- `xdebug` is included but disabled by default (`xdebug.mode=off`).
- Check it: `bin/php7/bin/php -m | grep -i ffi`

## Libraries (libs) used in the binary

The PHP/iAkunaMC binary ships with the following libraries and extensions already compiled and integrated.
You do not need to install them separately: they travel inside the tarball/zip and `ld` resolves them automatically.

### System libraries (C/C++) used by the binary

| Library | Version | What it is used for in iAkunaMC/PHP |
|---------|---------|--------------------------------------|
| zlib | 1.3.2 | Base deflate/gzip compression. Used by PHP and by network/world compression. |
| GMP | 6.3.0 | Arbitrary-precision arithmetic. Required for Bedrock packet cryptography (RSA/DH and big integers). |
| OpenSSL | 3.6.5 | TLS/SSL and cryptography. Secure HTTPS connections (curl) and Bedrock packet encryption. |
| curl | 8.13.0 | HTTP/HTTPS client. Web requests from plugins/server, downloads and API consumption. |
| libyaml | 0.2.5 | YAML parser/emitter. Reading configuration (server.yml, plugins). |
| LevelDB | commit `1c756446` | Embedded key-value database. Storage of Bedrock worlds (chunks/regions). |
| libpng | 1.6.53 | PNG read/write. Image support (gd and plugins). |
| libjpeg | 9f | JPEG read/write. Image support. |
| libxml2 | 2.15.1 | XML parser. UPnP and XML manipulation in PHP. |
| libzip | 1.12.1 | ZIP files. Resource packs and phar files. |
| SQLite | 3.51.1 | Embedded database. Local data and plugins. |
| libdeflate | 1.25 (commit `c8c56a20`) | Optimized deflate/gzip/zlib compression. Faster network compression than zlib. |
| libffi | 3.4.8 | Foreign function interface. Enables `ext-ffi` to call C code from PHP. |

### PHP extensions compiled into the binary

| Extension | Version | What it is used for |
|-----------|---------|---------------------|
| pmmpthread (qthreads fork) | main | Threading for PHP ZTS. Source is Bedrock-Phanatics/ext-qthreads, Quark's fork of pmmp/ext-pmmpthread (itself a fork of krakjoe/pthreads). Drop-in replacement: still loaded as `pmmpthread`, API remains in `pmmp\\thread`. Requires PHP 8.4+; adds `ThreadSafeQueue`, faster `Pool`/`Worker` submission and collection, faster `ThreadSafeArray` merges, object creation and connections, plus fixes for crashes/races/deadlocks/leaks; includes sanitizer CI and a benchmark suite. |
| yaml | 2.3.0 | PHP extension for YAML. Configuration. |
| leveldb | commit `88071eb1` | PHP extension on top of LevelDB. World access. |
| chunkutils2 | 0.3.5 | Low-level utilities for chunks and block palettes. |
| xdebug | 3.5.0 | Debugging/profiling (not recommended for production). |
| igbinary | 3.2.17RC1 (PHP 8.5) | Fast binary serialization. Passing data between threads. |
| crypto | commit `999b3c7e` | Bedrock packet encryption (AES and others). |
| recursionguard | 0.1.0 | Protection against infinite recursion while debugging. |
| libdeflate | master | PHP bindings for libdeflate, a zlib replacement with significantly better performance. This fork adds whole-buffer decompression APIs for raw DEFLATE/ZLIB/GZIP with single-pass GZIP decompression, dynamic buffer scaling and zip-bomb protection (default 256 MB limit). |
| morton | 0.1.2 | Packs X/Z (and X/Y/Z) coordinates into integers for PHP array keys. |
| xxhash | 0.2.0 | Very fast hash (optional, better performance than `hash()`). |
| arraydebug | 0.2.1 | Debugging hash collisions in arrays. |
| encoding | master | High-performance raw data encoding/decoding utilities for PHP, superseding pocketmine/binaryutils and pack()/unpack(). Includes ByteBufferReader/Writer, LE/BE/Byte/VarInt and batch array operations (read*/unpack*/write*/pack*); measured 50x-60x faster writes/packs and 7.5x reads/unpacks. Fork increases ByteBufferWriter default buffer from 16 to 256 bytes and adds native batch array functions. |
| FFI | included in PHP 8.5 | Calling C from PHP (enabled with `ffi.enable=true`). |
| OPcache (+JIT) | included in PHP 8.5 | Bytecode cache and JIT compilation for better performance. JIT is disabled by default in php.ini. |

> Standard PHP extensions used by iAkunaMC are also compiled: `gd`, `mbstring`, `sockets`, `bcmath`, `calendar`, `ctype`, `dom`, `fileinfo`, `ftp`, `pcntl`, `phar`, `posix`/`shmop`, `simplexml`, `sqlite3` (PDO), `xml`, `zlib`, etc.

## Packaging: no empty .so libs

Use `./package.sh <output.tar.gz> [dir]` instead of `tar` by hand. It does:

1. Aborts if there are broken symlinks (a library that would be lost).
2. **Converts symlink chains into hardlinks** (`libffi.so -> libffi.so.8 -> libffi.so.8.x.y`:
   only the last name is a real file; the others point to it). With hardlinks, all names are the
   same file with full content. So even if your extractor does not support symlinks
   (FileZilla, Windows, cPanel...), no `.so` ends up empty or missing: it is extracted as a
   normal file and `ld` resolves it just the same.
3. Also generates a **twin `.zip`** with the same content (recommended download for
   FileZilla/Windows/cPanel).
4. Verifies the result by extracting tar and zip into temp folders and comparing the **sha256 of
   every `.so`** against the original: if any were empty or truncated, the build fails in CI and
   is not published.

### Binaries in the repo (`binarios` branch)
Each push to `stable` triggers the `publish-binaries-branch` job, which uploads the compiled
binaries to the [`binarios`](../binarios) branch: it deletes the previous binaries, uploads the
new ones and keeps README and workflows. Nothing is downloaded from external releases: the libs
travel inside the tarballs and none is lost.

**When extracting**, use `tar -xzf` or the `.zip`. Either one keeps all libs intact; it no longer
depends on your tool supporting symlinks.

---

# Build script reference

## compile.sh

Bash script used to compile PHP on MacOS and Linux platforms. Make sure you have ``make autoconf automake libtool m4 wget getconf gzip bzip2 bison g++ git cmake pkg-config re2c ca-certificates``.

### Recommendations
- If you're going to use the compiled binary only on the machine you're build it on, remove the `-t` option for best performance - this will allow the script to optimize for the current machine rather than a generic one.
- [`ext-gd`](https://www.php.net/manual/en/book.image.php) (GD) **is included by default** in this fork, together with `libpng` and `libjpeg`, so image functions are available out of the box. The `-g` flag keeps/enables it (it is already on by default).
- The `-c` and `-l` options can be used to specify cache folders to speed up recompiling if you're recompiling many times (e.g. to improve the script).

### Common pitfalls
- Avoid using the script in directory trees containing spaces. Some libraries don't like trying to be built in directory trees containing spaces, e.g. `/home/user/my folder/pocketmine-mp/` might experience problems.
- Avoid directory trees containing special (non-English) symbols. For example, `Développement` might cause issues.

### Additional notes
#### Mac OSX (native compile)
- Most dependencies can be installed using Homebrew
- You will additionally need `glibtool` (GNU libtool, xcode libtool won't work)

#### Android 64-bit (cross-compile)
- Only aarch64 targets are supported for Android cross-compile.
- The `aarch64-linux-musl` toolchain is required. You can compile and install it using https://github.com/pmmp/musl-cross-make (PMMP fork includes musl-libc patches for DNS resolver config path and increasing stack size limit for LevelDB)

| Script flags | Description                                                                                                 |
|--------------|-------------------------------------------------------------------------------------------------------------|
| -c           | Uses the folder specified for caching downloaded tarballs, zipballs etc.                                    |
| -d           | Compiles with debugging symbols and disables optimizations (slow, but useful for debugging segfaults)       |
| -D           | Compiles with separated debugging symbols, but leaves optimizations enabled (used for distributed binaries) |
| -g           | Will compile GD2                                                                                            |
| -j           | Set make threads to #                                                                                       |
| -l           | Uses the folder specified for caching compilation artifacts (useful for rapid rebuild and testing)          |
| -n           | Don't remove sources after completing compilation                                                           |
| -s           | Will compile everything statically                                                                          |
| -t           | Set target                                                                                                  |
| -v           | Enable Valgrind support in PHP                                                                              |
| -x           | Specifies we are doing cross-compile                                                                        |
| -P           | Set target to the specified major iAkunaMC (PocketMine-MP 5) version specified (currently only `5` is supported)         |
| -z           | PHP version to build, e.g `8.2` (optional, if not specified, will be auto-selected by PM version)           |

### Example:

| Target          | Arguments                         |
|-----------------|-----------------------------------|
| linux64         | ``-t linux64 -j4 -P5``            |
| android-aarch64 | ``-t android-aarch64 -x -j4 -P5`` |

## For developers: Version info sources
### Libraries

| Link to package | Needed for | Notes |
|:----------------|:-----------|:------|
| [zlib](https://github.com/madler/zlib/tags) | Compression | |
| [gmp](https://gmplib.org/) | Big integer math for Bedrock packet encryption | Hosted at [DependencyMirror](https://github.com/pmmp/DependencyMirror/releases) to avoid service outages |
| [curl](https://github.com/curl/curl/releases) | Web requests | |
| [libyaml](https://github.com/yaml/libyaml/releases) | Parsing YAML config files | |
| [leveldb](https://github.com/pmmp/leveldb/commits/mojang-compatible/) | Bedrock world support | Custom version based on google/leveldb with minimum required changes to support MCPE worlds |
| [libxml](https://gitlab.gnome.org/GNOME/libxml2/-/releases) | XML parsing support for UPnP | Hosted at [DependencyMirror](https://github.com/pmmp/DependencyMirror/releases) to avoid service outages |
| [libpng](https://sourceforge.net/projects/libpng/files/libpng16/) | php-gd, plugin use only | Hosted at [DependencyMirror](https://github.com/pmmp/DependencyMirror/releases) to avoid service outages |
| [libjpeg](https://ijg.org/) | php-gd, plugin use only | Hosted at [DependencyMirror](https://github.com/pmmp/DependencyMirror/releases) to avoid service outages |
| [openssl](https://github.com/openssl/openssl/releases) | Bedrock packet encryption, secure web requests | |
| [libzip](https://github.com/nih-at/libzip/releases) | Resource packs | |
| [sqlite3](https://sqlite.org/download.html) | Plugin use only | Hosted at [DependencyMirror](https://github.com/pmmp/DependencyMirror/releases) to avoid service outages |
| [libdeflate](https://github.com/ebiggers/libdeflate/blob/master/NEWS.md) | Faster alternative to zlib for network use | |
| [pthreads4w](https://sourceforge.net/projects/pthreads4w/files/) | Needed by ext-qthreads on Windows | Hosted at [DependencyMirror](https://github.com/pmmp/DependencyMirror/releases) to avoid service outages |

### PHP & extensions

| Link to package | Needed for | Notes |
|:----------------|:-----------|:------|
| [PHP](https://www.php.net/releases/?json&version=8.5) | Everything | Replace 8.5 in the URL with your chosen version |
| [qthreads](https://github.com/Bedrock-Phanatics/ext-qthreads) | PHP threading | Quark's fork of pmmp/ext-pmmpthread (itself a fork of krakjoe/pthreads); drop-in replacement loaded as `pmmpthread` with `pmmp\\thread` API. Requires PHP 8.4+; adds `ThreadSafeQueue`, faster `Pool`/`Worker` submission/collection, faster `ThreadSafeArray` merges/creation/connections, and fixes for crashes/races/deadlocks/leaks. |
| [yaml](https://github.com/php/pecl-file_formats-yaml/tags) | YAML config parsing | Yes, the mix of - and _ is intentional. Don't ask me. |
| [leveldb](https://github.com/pmmp/php-leveldb/commits/pmmp-mojang-compatible/) | Bedrock world support | Custom version to provide `LEVELDB_ZLIB_RAW_COMPRESSION` support |
| [chunkutils2](https://github.com/pmmp/ext-chunkutils2/releases) | `PalettedBlockArray` and other low-level stuff | |
| [xdebug](https://github.com/xdebug/xdebug/releases) | Debugging | Not needed for production |
| [igbinary](https://github.com/igbinary/igbinary/releases) | Faster serialization, mostly for moving stuff between threads | Non-essential, could be ditched if necessary |
| [crypto](https://github.com/bukka/php-crypto/tags) | Bedrock packet encryption | |
| [recursionguard](https://github.com/pmmp/ext-recursionguard/releases) | Debugging | Not needed for production |
| [libdeflate](https://github.com/m4theuswtfkkj/ext-libdeflate/releases) | Faster network compression | Fork of pmmp/ext-libdeflate; PHP bindings for libdeflate with significantly better performance than zlib, plus whole-buffer decompression APIs (raw DEFLATE/ZLIB/GZIP), single-pass GZIP decompression, dynamic buffer scaling and zip-bomb protection |
| [morton](https://github.com/pmmp/ext-morton) | Packing X/Z and X/Y/Z coordinates into ints in a format suitable for PHP array keys | Needed for performance |
| [xxhash](https://github.com/pmmp/ext-xxhash/releases) | Not currently used | Could be replaced by `hash()` in recent versions of PHP but this extension has much better performance |
| [arraydebug](https://github.com/pmmp/ext-arraydebug/tags) | Debugging array hash collisions | |
| [encoding](https://github.com/m4theuswtfkkj/ext-encoding/releases) | Disk & network data encoding & decoding | Fork of pmmp/ext-encoding; high-performance encoding/decoding utilities superseding pocketmine/binaryutils and pack()/unpack(), with ByteBufferReader/Writer, LE/BE/Byte/VarInt and batch array operations (50x-60x faster writes/packs, 7.5x reads/unpacks) |

### Misc

| Link to package | Needed for | Notes |
|:----------------|:-----------|:------|
| [php-sdk-binary-tools](https://github.com/php/php-sdk-binary-tools/releases) | Building PHP on Windows | |