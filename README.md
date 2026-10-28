# Native Janus build for Moonraker-Obico on the Creality K1

## Status

This is a record of commands tested on a rooted Creality K1. The complete dependency stack and pinned Moonraker-Obico Janus fork have been built, installed and started successfully with DataChannels, the Streaming plugin and WebSockets transport.

The end-to-end procedure was tested on Creality firmware **1.3.5.22**.

The chosen architecture is an entirely Entware-hosted Janus process under `/opt`. It is intentionally isolated from the K1 firmware libraries.

## Purpose

Moonraker-Obico's `mjpeg_webrtc` mode does not transcode MJPEG into H.264. It captures JPEG frames and transports them through a WebRTC DataChannel via Janus. The required Janus features are therefore:

- WebRTC DataChannels (`usrsctp`)
- WebSockets transport
- Streaming plugin
- ICE via `libnice`
- DTLS/SRTP

The standard K1 camera is available directly at `localhost:8080`. Fluidd's proxied camera endpoint is available on port `4408` if required.

## Tested baseline and prerequisites

This Janus procedure was tested on a K1 that had already been prepared in the following order:

1. Root access and SSH were enabled through the K1 interface.
2. [Guilouz's Creality Helper Script](https://github.com/Guilouz/Creality-Helper-Script) was installed.
3. Moonraker and Fluidd were installed through the Helper Script.
4. Moonraker-Obico was installed, linked to the intended Obico server, and running from `/usr/data/moonraker-obico`.
5. The firmware 1.3.5.22 camera regression was corrected using the `mjpg_streamer` startup workaround from [Creality Helper Script Wiki discussion #901](https://github.com/Guilouz/Creality-Helper-Script-Wiki/discussions/901).
6. The restored camera producer was verified directly on `127.0.0.1:8080` before beginning the Janus work.

The guide does not require Fluidd to carry camera frames to Obico. The direct port `8080` endpoints are preferred; Fluidd on port `4408` is retained as a fallback and as a convenient independent camera test.

### Firmware 1.3.5.22 camera prerequisite

On the tested firmware, the camera could remain available to Creality Cloud while the local `mjpg_streamer` producer was no longer started automatically. The discussion #901 workaround adds `/etc/init.d/S60mjpg_streamer`, waits for `cam_app`, and starts:

```sh
/usr/bin/mjpg_streamer -b \
  -i "/usr/lib/mjpg-streamer/input_memfd.so -t 0" \
  -o "/usr/lib/mjpg-streamer/output_http.so -p 8080"
```

Follow discussion #901 for the complete init script and any subsequent revisions rather than treating the abbreviated command above as a complete service definition. Ensure the installed service is executable:

```sh
sed -i 's/\r$//' /etc/init.d/S60mjpg_streamer
chmod 755 /etc/init.d/S60mjpg_streamer
```

Firmware updates may replace `/etc/init.d`. Keep a recoverable copy on the persistent `/usr/data` partition:

```sh
cp -p /etc/init.d/S60mjpg_streamer /usr/data/S60mjpg_streamer
```

After a future firmware update, inspect compatibility before copying the service back. Do not assume the workaround remains necessary or unchanged on later firmware.

Confirm the producer and both HTTP endpoints before compiling Janus:

```sh
pidof cam_app
pidof mjpg_streamer

/opt/bin/curl -fsS \
  'http://127.0.0.1:8080/?action=snapshot' \
  -o /tmp/k1-camera.jpg

head -c 2 /tmp/k1-camera.jpg | od -An -tx1

/opt/bin/curl -sS --max-time 3 \
  'http://127.0.0.1:8080/?action=stream' \
  -o /tmp/k1-camera-stream.sample

test -s /tmp/k1-camera-stream.sample && echo "MJPEG stream returned data"
```

The JPEG signature must be `ff d8`; the continuous stream test normally exits with curl code 28 after its deliberate three-second timeout but must have written data.

Moonraker-Obico may use explicit direct URLs in its own configuration even when Moonraker has no registered webcam. Fluidd requires a Moonraker webcam entry; discussion #901 includes the relevant `/server/webcams/list` check and registration request when needed.

## Confirmed K1 ABI

The stock firmware reports:

```text
GNU C Library (Ingenic Linux-Release5.1.0.-Default(xburst2(fp64)+glibc2.29)) 2.29
Compiled by GNU GCC 7.2.0
libc ABIs: MIPS_PLT UNIQUE MIPS_O32_FP64 ABSOLUTE
```

The available Entware repository is `mipselsf-k3.4`. Entware produces MIPS32r2 O32 soft-float executables and supplies its own runtime beneath `/opt`. A test executable built with Entware GCC runs correctly on the K1.

Do not mix Entware objects with `/lib` or `/usr/lib`. Janus and all of its linked libraries must come from `/opt`. Communication with Moonraker-Obico occurs across process boundaries using WebSockets and UDP, so the differing C ABIs are acceptable.

## Storage and build precautions

Perform compilation while the printer is idle. The K1 has limited RAM, so use `make -j1` throughout.

Check available storage:

```sh
df -h /opt /usr/data
```

Allow at least 500 MB free; 1 GB is preferable while source and object trees are present.

## Install the Entware build environment

```sh
opkg update

opkg install \
  gcc binutils make cmake \
  autoconf automake libtool-bin m4 \
  glib2 jansson libconfig libffi \
  libpcre2 zlib libopenssl \
  libsrtp2 libwebsockets-openssl \
  curl ca-bundle ca-certificates \
  xz tar
```

Package names can vary slightly as the Entware feed changes. Confirm the installed WebSockets package using:

```sh
opkg list-installed | grep -E 'websocket|websockets'
```

## Install Entware development headers

Entware does not normally provide `-dev` packages. It publishes a combined header archive for each architecture.

```sh
mkdir -p /opt/include

/opt/bin/curl -fL \
  https://bin.entware.net/mipselsf-k3.4/include/include.tar.gz \
  -o /opt/entware-include.tar.gz

/opt/bin/tar -xzf /opt/entware-include.tar.gz -C /opt/include
rm /opt/entware-include.tar.gz
```

The archive contains the contents of the include directory itself, so it must be extracted into `/opt/include`, not `/opt`.

## Standard build environment

Use this environment for all subsequent builds:

```sh
export PATH=/opt/bin:/opt/sbin:$PATH
export LD_LIBRARY_PATH=/opt/lib
export PKG_CONFIG=/opt/bin/pkg-config
export PKG_CONFIG_PATH=/opt/lib/pkgconfig
export PKG_CONFIG_LIBDIR=/opt/lib/pkgconfig

export CPPFLAGS="-I/opt/include"
export CFLAGS="-Os -fPIC"
export LDFLAGS="-L/opt/lib -Wl,-rpath,/opt/lib"
```

## Bootstrap pkgconf

The Entware feed does not contain `pkg-config` or `pkgconf`, so build `pkgconf` natively:

```sh
mkdir -p /opt/src
cd /opt/src

/opt/bin/curl -fL \
  https://github.com/pkgconf/pkgconf/releases/download/pkgconf-3.0.6/pkgconf-3.0.6.tar.xz \
  -o pkgconf-3.0.6.tar.xz

/opt/bin/tar -xJf pkgconf-3.0.6.tar.xz
cd pkgconf-3.0.6

./configure \
  --prefix=/opt \
  --disable-static \
  --enable-shared \
  --with-system-libdir=/opt/lib \
  --with-system-includedir=/opt/include

make -j1
make install
```

Create the conventional command name if it was not installed automatically:

```sh
test -e /opt/bin/pkg-config || ln -s /opt/bin/pkgconf /opt/bin/pkg-config
pkg-config --version
```

## Restore missing linker symlinks

Entware installs several runtime SONAMEs without the unversioned names needed by the linker:

```sh
cd /opt/lib

ln -sf libconfig.so.15 libconfig.so
ln -sf libcrypto.so.3 libcrypto.so
ln -sf libssl.so.3 libssl.so
ln -sf libsrtp2.so.1 libsrtp2.so
ln -sf libffi.so.8 libffi.so
```

## Restore pkg-config metadata

Create the metadata directory:

```sh
mkdir -p /opt/lib/pkgconfig
```

Create `/opt/lib/pkgconfig/libffi.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: libffi
Description: Foreign Function Interface library
Version: 3.4.7
Libs: -L${libdir} -lffi
Cflags: -I${includedir}
```

Create `/opt/lib/pkgconfig/libpcre2-8.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: libpcre2-8
Description: PCRE2 8-bit library
Version: 10.47
Libs: -L${libdir} -lpcre2-8
Cflags: -I${includedir}
```

Create `/opt/lib/pkgconfig/zlib.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: zlib
Description: Compression library
Version: 1.3.1
Libs: -L${libdir} -lz
Cflags: -I${includedir}
```

Create `/opt/lib/pkgconfig/glib-2.0.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: GLib
Description: Core application support library
Version: 2.82.0
Requires.private: libpcre2-8
Libs: -L${libdir} -lglib-2.0
Libs.private: -pthread -lm
Cflags: -I${includedir}/glib-2.0
```

Create `/opt/lib/pkgconfig/gobject-2.0.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: GObject
Description: GLib object system
Version: 2.82.0
Requires: glib-2.0
Requires.private: libffi
Libs: -L${libdir} -lgobject-2.0
Cflags: -I${includedir}/glib-2.0
```

Create `/opt/lib/pkgconfig/gmodule-2.0.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: GModule
Description: GLib dynamic module support
Version: 2.82.0
Requires: glib-2.0
Libs: -L${libdir} -lgmodule-2.0
Libs.private: -ldl
Cflags: -I${includedir}/glib-2.0
```

Create `/opt/lib/pkgconfig/gthread-2.0.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: GThread
Description: GLib thread support
Version: 2.82.0
Requires: glib-2.0
Libs: -L${libdir} -lgthread-2.0 -pthread
Cflags: -I${includedir}/glib-2.0
```

Create `/opt/lib/pkgconfig/gio-2.0.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: GIO
Description: GLib input, output and streaming library
Version: 2.82.0
Requires: glib-2.0 gobject-2.0
Requires.private: zlib
Libs: -L${libdir} -lgio-2.0
Cflags: -I${includedir}/glib-2.0
```

Create `/opt/lib/pkgconfig/jansson.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: Jansson
Description: JSON encoding and decoding library
Version: 2.15.0
Libs: -L${libdir} -ljansson
Cflags: -I${includedir}
```

Create `/opt/lib/pkgconfig/libconfig.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: libconfig
Description: Structured configuration file library
Version: 1.8.1
Libs: -L${libdir} -lconfig
Cflags: -I${includedir}
```

Create `/opt/lib/pkgconfig/libcrypto.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: OpenSSL-libcrypto
Description: OpenSSL cryptography library
Version: 3.5.5
Libs: -L${libdir} -lcrypto
Libs.private: -ldl -pthread
Cflags: -I${includedir}
```

Create `/opt/lib/pkgconfig/libssl.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: OpenSSL-libssl
Description: OpenSSL TLS library
Version: 3.5.5
Requires.private: libcrypto
Libs: -L${libdir} -lssl
Cflags: -I${includedir}
```

Create `/opt/lib/pkgconfig/openssl.pc`:

```ini
prefix=/opt

Name: OpenSSL
Description: Secure Sockets Layer and cryptography libraries
Version: 3.5.5
Requires: libssl libcrypto
```

Create `/opt/lib/pkgconfig/libsrtp2.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: libsrtp2
Description: Secure Real-time Transport Protocol library
Version: 2.6.0
Requires.private: openssl
Libs: -L${libdir} -lsrtp2
Cflags: -I${includedir}
```

Create `/opt/lib/pkgconfig/libwebsockets.pc`, adjusting the version if the installed Entware package differs:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: libwebsockets
Description: WebSockets library
Version: 4.4.1
Requires.private: openssl zlib
Libs: -L${libdir} -lwebsockets
Cflags: -I${includedir}
```

Validate the reconstructed metadata:

```sh
for module in \
  glib-2.0 gthread-2.0 gobject-2.0 gio-2.0 \
  jansson libconfig openssl libsrtp2 libwebsockets
do
    if pkg-config --exists "$module"; then
        echo "OK: $module $(pkg-config --modversion "$module")"
    else
        echo "MISSING: $module"
        pkg-config --print-errors --exists "$module"
    fi
done
```

This validation has succeeded on the target K1.

## Build usrsctp

```sh
cd /opt/src

/opt/bin/curl -fL \
  https://github.com/sctplab/usrsctp/archive/refs/tags/0.9.5.0.tar.gz \
  -o usrsctp-0.9.5.0.tar.gz

/opt/bin/tar -xzf usrsctp-0.9.5.0.tar.gz
cd usrsctp-0.9.5.0

./bootstrap

./configure \
  --prefix=/opt \
  --disable-static \
  --enable-shared \
  --disable-programs \
  --disable-inet \
  --disable-inet6

make -j1
make install
```

Validate the installation:

```sh
ls -l /opt/lib/libusrsctp.so*
find /opt/include -name usrsctp.h
find /opt/lib/pkgconfig -name '*usrsctp*.pc'

pkg-config --modversion usrsctp
pkg-config --cflags --libs usrsctp
```

The build did not install pkg-config metadata on the tested K1. Create `/opt/lib/pkgconfig/usrsctp.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: usrsctp
Description: Userspace SCTP implementation
Version: 0.9.5.0
Libs: -L${libdir} -lusrsctp
Libs.private: -pthread
Cflags: -I${includedir}
```

Compile a functional test:

```sh
cat >/tmp/usrsctp-test.c <<'EOF'
#include <stdio.h>
#include <usrsctp.h>

int main(void)
{
    usrsctp_init(0, NULL, NULL);
    puts("usrsctp initialised");
    usrsctp_finish();
    return 0;
}
EOF

gcc /tmp/usrsctp-test.c \
  $(pkg-config --cflags --libs usrsctp) \
  -Wl,-rpath,/opt/lib \
  -o /tmp/usrsctp-test

/tmp/usrsctp-test
```

Expected output:

```text
usrsctp initialised
```

The library build, manually created metadata and functional compilation test have all succeeded on the target K1.

## Build libnice

Use libnice 0.1.17 because it retains the Autotools build system and requires only GLib 2.54 or newer. Later versions use Meson, which is not available in this Entware feed.

Install the additional build utilities discovered during the tested build:

```sh
opkg install sed perl perlbase-safe
```

`perlbase-safe` supplies `Safe.pm` and should pull in the required Opcode module.

Download the official release archive:

```sh
cd /opt/src

/opt/bin/curl -fL \
  https://libnice.freedesktop.org/releases/libnice-0.1.17.tar.gz \
  -o libnice-0.1.17.tar.gz

/opt/bin/tar -xzf libnice-0.1.17.tar.gz
cd libnice-0.1.17
```

The official release archive contains `configure`; there is no need to run `autogen.sh` or `autoreconf`.

### Repair MB_LEN_MAX for Entware native compilation

Entware's development-header archive omits the standard C `limits.h`. GCC consequently falls back to a generic `MB_LEN_MAX` value of 1, while Entware glibc correctly expects 16. Supply the correct value through the preprocessor:

```sh
export CPPFLAGS="-I/opt/include -DMB_LEN_MAX=16"
export CFLAGS="-Os -fPIC"
export LDFLAGS="-L/opt/lib -Wl,-rpath,/opt/lib"
```

Validate the workaround:

```sh
printf '#include <stdlib.h>\nint main(void){return 0;}\n' | \
  gcc -I/opt/include -DMB_LEN_MAX=16 -x c - -o /tmp/stdlib-test

/tmp/stdlib-test
```

### Install glib-mkenums

Entware's GLib package omits the build-time `glib-mkenums` utility. Use the classic Perl implementation from GLib 2.52.3, which is compatible with libnice's enum templates:

```sh
/opt/bin/curl -fL \
  https://raw.githubusercontent.com/GNOME/glib/2.52.3/gobject/glib-mkenums.in \
  -o /tmp/glib-mkenums.in

ls -lh /tmp/glib-mkenums.in
wc -c -l /tmp/glib-mkenums.in

sed \
  -e '1s|@PERL_PATH@|/opt/bin/perl|' \
  -e 's|@GLIB_VERSION@|2.52.3|g' \
  /tmp/glib-mkenums.in \
  > /opt/bin/glib-mkenums

chmod 755 /opt/bin/glib-mkenums
rm /tmp/glib-mkenums.in
```

Do not assume `/opt/bin/sed` exists. Use `sed` after installing the Entware package. Shell redirection creates the destination before attempting to execute `sed`, so using a nonexistent `/opt/bin/sed` silently leaves an empty `/opt/bin/glib-mkenums`. Always verify the installed script:

```sh
wc -c -l /opt/bin/glib-mkenums
head -3 /opt/bin/glib-mkenums
/opt/bin/perl -c /opt/bin/glib-mkenums
/opt/bin/perl -MSafe -e 'print "Safe.pm works\n"'
```

Test the generator against the actual libnice header:

```sh
cd /opt/src/libnice-0.1.17

/opt/bin/glib-mkenums \
  --vhead '#define NICE_TYPE_@ENUMSHORT@' \
  agent/agent.h | \
  grep NOMINATION
```

Expected output:

```text
#define NICE_TYPE_NOMINATION_MODE
```

### Configure and compile libnice

```sh
./configure \
  --prefix=/opt \
  --disable-static \
  --enable-shared \
  --with-crypto-library=openssl \
  --without-gstreamer \
  --without-gstreamer-0.10 \
  --disable-gupnp \
  --disable-introspection \
  --disable-gtk-doc
```

Remove any empty or incomplete enum files left by an earlier failed generator invocation:

```sh
rm -f \
  agent/agent-enum-types.h \
  agent/agent-enum-types.c
```

Compile conservatively:

```sh
make -j1 \
  GLIB_MKENUMS=/opt/bin/glib-mkenums \
  CPPFLAGS="-I/opt/include -DMB_LEN_MAX=16"
```

This compilation has succeeded on the target K1.

### Install and validate libnice

```sh
make install

ls -l /opt/lib/libnice.so*
find /opt/include -path '*/nice/agent.h'
find /opt/lib/pkgconfig -name nice.pc

pkg-config --modversion nice
pkg-config --cflags --libs nice
```

If `nice.pc` was not installed, create `/opt/lib/pkgconfig/nice.pc`:

```ini
prefix=/opt
libdir=${prefix}/lib
includedir=${prefix}/include

Name: libnice
Description: ICE library based on GLib
Version: 0.1.17
Requires: glib-2.0 gio-2.0 gobject-2.0
Requires.private: gthread-2.0 libcrypto
Libs: -L${libdir} -lnice
Cflags: -I${includedir}
```

Compile a functional test:

```sh
cat >/tmp/libnice-test.c <<'EOF'
#include <stdio.h>
#include <nice/agent.h>

int main(void)
{
    NiceAgent *agent = nice_agent_new(NULL, NICE_COMPATIBILITY_RFC5245);

    if (agent == NULL) {
        fprintf(stderr, "nice_agent_new failed\n");
        return 1;
    }

    puts("libnice initialised");
    g_object_unref(agent);
    return 0;
}
EOF

gcc /tmp/libnice-test.c \
  $(pkg-config --cflags --libs nice) \
  -DMB_LEN_MAX=16 \
  -Wl,-rpath,/opt/lib \
  -o /tmp/libnice-test

/tmp/libnice-test
```

Expected output:

```text
libnice initialised
```

The install supplied its own working `nice.pc` metadata. The library installation, pkg-config detection and functional `NiceAgent` test have all succeeded on the target K1.

## Build GNU Gengetopt

Janus 0.11.1 requires the `gengetopt` executable at configure time. It is not packaged by the Entware `mipselsf-k3.4` feed, so build GNU Gengetopt 2.23 natively. It is a build-time utility and is not a Janus runtime dependency.

Confirm that Entware GCC includes its C++ frontend:

```sh
command -v g++
g++ --version
```

Download from a fixed GNU mirror rather than the `ftpmirror.gnu.org` redirector, which returned HTTP 502 during the tested build:

```sh
cd /opt/src

/opt/bin/curl -fL \
  --retry 5 \
  --retry-delay 2 \
  https://ftp.gnu.org/gnu/gengetopt/gengetopt-2.23.tar.xz \
  -o gengetopt-2.23.tar.xz

/opt/bin/xz -t gengetopt-2.23.tar.xz
/opt/bin/tar -xJf gengetopt-2.23.tar.xz
cd gengetopt-2.23
```

Gengetopt 2.23 does not recognise `--disable-nls`; omit that option.

Entware Libtool incorrectly converts `-Wl,-rpath,/opt/lib` into a bare `-rpath` option passed directly to `g++`. Gengetopt does not need a custom rpath, so configure it with only the library search path:

```sh
export CPPFLAGS="-I/opt/include -DMB_LEN_MAX=16"
export CFLAGS="-Os"
export CXXFLAGS="-Os"
export LDFLAGS="-L/opt/lib"

./configure --prefix=/opt
```

Build the project:

```sh
make -j1 LDFLAGS="-L/opt/lib"
```

The top-level build may fail after successfully creating `src/gengetopt` because Entware does not package `makeinfo`. Setting `MAKEINFO=true` is not sufficient: the documentation rule subsequently expects a generated `gengetopt.htp` file and fails while renaming it.

Confirm the executable itself was built:

```sh
ls -l src/gengetopt
./src/gengetopt --version
```

Install only the program subtree, bypassing the irrelevant documentation directory:

```sh
make -C src install LDFLAGS="-L/opt/lib"

/opt/bin/gengetopt --version
command -v gengetopt
```

The selective `src` installation also installs Gengetopt's required support files beneath `/opt/share/gengetopt`. This build and installation have succeeded on the target K1.

## Build the Moonraker-Obico Janus fork

Moonraker-Obico uses its own Janus fork rather than an unmodified current upstream release. The pinned fork revision below identifies itself as Janus 0.11.1 and contains Obico-specific Streaming/DataChannel behavior.

```sh
cd /opt/src

/opt/bin/curl -fL \
  https://github.com/TheSpaghettiDetective/janus-gateway/archive/119d220c2e6324dd2233d8f93868fcaa1ae229fd.tar.gz \
  -o obico-janus-119d220c.tar.gz

/opt/bin/tar -xzf obico-janus-119d220c.tar.gz
cd janus-gateway-119d220c2e6324dd2233d8f93868fcaa1ae229fd

grep '^AC_INIT' configure.ac
./autogen.sh
```

Use the corrected equals-form rpath syntax for Entware Libtool projects:

```sh
export PATH=/opt/bin:/opt/sbin:$PATH
export LD_LIBRARY_PATH=/opt/lib
export PKG_CONFIG=/opt/bin/pkg-config
export PKG_CONFIG_PATH=/opt/lib/pkgconfig
export PKG_CONFIG_LIBDIR=/opt/lib/pkgconfig

export CPPFLAGS="-I/opt/include -DMB_LEN_MAX=16"
export CFLAGS="-Os -fPIC"
export LDFLAGS="-L/opt/lib -Wl,-rpath=/opt/lib -Wl,--export-dynamic"
```

Configure a minimal Obico-compatible build beneath its own prefix:

```sh
./configure \
  --prefix=/opt/obico-janus \
  --libdir=/opt/obico-janus/lib \
  --disable-docs \
  --enable-data-channels \
  --enable-libsrtp2 \
  --disable-turn-rest-api \
  --disable-all-plugins \
  --enable-plugin-streaming \
  --disable-all-transports \
  --enable-websockets \
  --disable-all-handlers \
  --disable-all-loggers \
  --disable-rest \
  --disable-unix-sockets \
  --disable-post-processing
```

The configure step has succeeded on the target K1. Its verified summary enables:

- libsrtp 2.x
- OpenSSL
- DataChannels
- WebSockets transport
- Streaming plugin

It disables REST, TURN REST API, recording post-processing, Unix sockets, all other plugins, all event handlers, external loggers and JavaScript modules. `DTLS set-timeout: not available` is expected for this OpenSSL build and is not an error.

Compile Janus conservatively while retaining the Entware-safe rpath form:

```sh
make -j1 \
  CPPFLAGS="-I/opt/include -DMB_LEN_MAX=16" \
  LDFLAGS="-L/opt/lib -Wl,-rpath=/opt/lib -Wl,--export-dynamic"
```

### Patch for libwebsockets 4.4

The Obico Janus fork predates libwebsockets 4.4 and refers to `LWS_CALLBACK_OPENSSL_CONTEXT_REQUIRES_PRIVATE_KEY`. Libwebsockets removed that callback value after its 4.3 series. Janus only uses it in functions that convert callback reason numbers into diagnostic strings; removing the obsolete `CASE_STR` entry does not alter WebSocket behavior.

If compilation fails on that identifier, patch all copies:

```sh
sed -i \
  '/CASE_STR(LWS_CALLBACK_OPENSSL_CONTEXT_REQUIRES_PRIVATE_KEY);/d' \
  transports/janus_websockets.c \
  events/janus_wsevh.c
```

Resume the build without cleaning:

```sh
make -j1 \
  CPPFLAGS="-I/opt/include -DMB_LEN_MAX=16" \
  LDFLAGS="-L/opt/lib -Wl,-rpath=/opt/lib -Wl,--export-dynamic"
```

The patched build has completed successfully on the target K1. The uninstalled `janus` executable reports version 0.11.1, and the build produced both `libjanus_streaming` and `libjanus_websockets` modules.

### Install Janus

```sh
make install \
  CPPFLAGS="-I/opt/include -DMB_LEN_MAX=16" \
  LDFLAGS="-L/opt/lib -Wl,-rpath=/opt/lib -Wl,--export-dynamic"
```

### Correct an executable linked without exported symbols

Supplying `LDFLAGS` on the `make` command line replaces the flags that Janus added during `configure`. The main executable must be linked with `--export-dynamic`, because dynamically loaded plugins and transports resolve Janus API functions from that executable.

If Janus reports errors such as these at startup:

```text
undefined symbol: janus_rtp_simulcasting_context_reset
undefined symbol: janus_transport_session_create
```

relink only the executable:

```sh
cd /opt/src/janus-gateway-119d220c2e6324dd2233d8f93868fcaa1ae229fd

mv janus janus.no-export

make -j1 janus \
  CPPFLAGS="-I/opt/include -DMB_LEN_MAX=16" \
  LDFLAGS="-L/opt/lib -Wl,-rpath=/opt/lib -Wl,--export-dynamic"

readelf --wide --dyn-syms ./janus | \
  grep -E 'janus_rtp_simulcasting_context_reset|janus_transport_session_create'

make install-binPROGRAMS
```

Both symbols must appear as `GLOBAL` entries in the dynamic symbol table. This relink and the subsequent plugin/transport load have succeeded on the target K1.

### Validate the installed Janus tree

```sh
/opt/obico-janus/bin/janus --version

find /opt/obico-janus/lib/janus -type f \
  \( -name 'libjanus_streaming.so' -o -name 'libjanus_websockets.so' \) \
  -print

readelf --wide --dyn-syms /opt/obico-janus/bin/janus | \
  grep -E 'janus_rtp_simulcasting_context_reset|janus_transport_session_create'

readelf -d /opt/obico-janus/bin/janus | grep NEEDED
readelf -d /opt/obico-janus/lib/janus/plugins/libjanus_streaming.so | grep NEEDED
readelf -d /opt/obico-janus/lib/janus/transports/libjanus_websockets.so | grep NEEDED
```

`ldd` may be used when available. The `NEEDED` entries are a useful fallback, but cannot detect missing exports from the host executable; that is why the dynamic-symbol check above is separate.

## Deploy the build into Moonraker-Obico

Moonraker-Obico generates Janus's runtime `.jcfg` files and starts Janus as a child process. Do not run a second standalone Janus service: it would conflict with Moonraker-Obico on WebSocket port `17730`.

Moonraker-Obico normally looks for a bundled precompiled Janus variant. Place symlinks to the K1 build in the exact variant directory that its Python code requests. The directory name `precomplied` is intentionally misspelled upstream.

```sh
cd /usr/data/moonraker-obico

VARIANT_DIR="$('/usr/data/moonraker-obico-env/bin/python3' - <<'PY'
import os
from moonraker_obico.janus_config_builder import (
    JANUS_ROOT_DIR,
    REQUESTED_PRECOMPILED_VARIANT,
)
print(os.path.join(
    JANUS_ROOT_DIR,
    'precomplied',
    REQUESTED_PRECOMPILED_VARIANT,
))
PY
)"

printf 'Moonraker-Obico Janus variant directory: %s\n' "$VARIANT_DIR"
mkdir -p "$VARIANT_DIR/bin"
ln -sf /opt/obico-janus/bin/janus "$VARIANT_DIR/bin/janus"

if [ -e "$VARIANT_DIR/lib" ] || [ -L "$VARIANT_DIR/lib" ]; then
    echo "STOP: $VARIANT_DIR/lib already exists; inspect it before replacing it"
else
    ln -s /opt/obico-janus/lib "$VARIANT_DIR/lib"
fi

ls -l "$VARIANT_DIR/bin/janus" "$VARIANT_DIR/lib"
```

If the Python import fails, include the application directory explicitly:

```sh
export PYTHONPATH=/usr/data/moonraker-obico
```

then repeat the `VARIANT_DIR` command.

This approach avoids maintaining a local Python source patch. On its next start, Moonraker-Obico selects the exact variant directory, uses the symlinked executable and module tree, sets `LD_LIBRARY_PATH` to the variant's `lib` symlink, and writes its generated configurations beneath:

```text
/usr/data/moonraker-obico/moonraker_obico/bin/janus/runtime/etc/janus/
```

## Configure the K1 camera for MJPEG-over-WebRTC

First validate the K1's direct camera endpoints:

```sh
/opt/bin/curl -fsS \
  'http://127.0.0.1:8080/?action=snapshot' \
  -o /tmp/k1-camera.jpg

wc -c /tmp/k1-camera.jpg
head -c 2 /tmp/k1-camera.jpg | od -An -tx1
```

A JPEG begins with `ff d8`. Also check that the stream responds:

```sh
/opt/bin/curl -fsS \
  --max-time 3 \
  'http://127.0.0.1:8080/?action=stream' \
  -o /dev/null
```

`curl` returning code 28 after the three-second limit is expected for a continuous stream. Other failures indicate that the direct endpoint is unavailable.

The K1 installer, Moonraker webcam discovery, or a previous manual setup may already have populated `/usr/data/printer_data/config/moonraker-obico.cfg`. Treat the existing file as authoritative: do not replace the whole file or disturb its server URL, authentication token, Moonraker connection, logging, tunnel, or additional webcam sections.

Back it up and inspect the existing primary webcam section first:

```sh
OBICO_CFG=/usr/data/printer_data/config/moonraker-obico.cfg
OBICO_BACKUP="${OBICO_CFG}.before-janus"

test -e "$OBICO_BACKUP" || cp -p "$OBICO_CFG" "$OBICO_BACKUP"
sed -n '/^\[webcam\]/,/^\[/p' "$OBICO_CFG"
```

If the existing `[webcam]` section already has the direct port `8080` URLs and `stream_mode = mjpeg_webrtc`, retain it. Only add or correct values that are absent or wrong. The resulting primary webcam section should be equivalent to:

```ini
[webcam]
disable_video_streaming = False
stream_mode = mjpeg_webrtc
snapshot_url = http://127.0.0.1:8080/?action=snapshot
stream_url = http://127.0.0.1:8080/?action=stream
target_fps = 10
aspect_ratio_169 = True
```

An existing `target_fps = 15` is reasonable to retain for the K1 when both direct camera endpoints have passed validation. Ten frames per second is a more conservative starting point for a new configuration. Obico Free accounts cap streaming at five FPS; self-hosted and Pro installations can use higher values. Check CPU, memory and network use before increasing it further.

If the direct port does not provide a usable snapshot or stream, test Fluidd's port `4408` proxy and substitute these URLs:

```ini
snapshot_url = http://127.0.0.1:4408/webcam/?action=snapshot
stream_url = http://127.0.0.1:4408/webcam/?action=stream
```

Prefer port `8080` when it works because it avoids an unnecessary proxy hop.

Moonraker-Obico will generate an MJPEG Streaming mount resembling:

```ini
mjpeg-2: {
        type = "rtp"
        id = 2
        description = "mjpeg-data"
        audio = false
        video = false
        data = true
        dataport = 17732
        datatype = "binary"
        dataiface = "127.0.0.1"
        databuffermsg = false
}
```

The exact UDP port is assigned at runtime. JPEG frames are carried over a WebRTC DataChannel; Janus does not decode or transcode them.

## Start on boot

The official K1 installer places `/etc/init.d/S99moonraker_obico` in the firmware boot sequence. Moonraker-Obico should remain the sole service owner of Janus.

Confirm the startup files:

```sh
ls -l \
  /etc/init.d/S99moonraker_obico \
  /usr/data/moonraker-obico/scripts/start_on_creality_k1.sh

chmod 755 \
  /etc/init.d/S99moonraker_obico \
  /usr/data/moonraker-obico/scripts/start_on_creality_k1.sh
```

Add the Entware environment and `exec` to the K1 launcher. Replace `/usr/data/moonraker-obico/scripts/start_on_creality_k1.sh` with:

```sh
#!/bin/sh

CONF_FILE=/usr/data/printer_data/config/moonraker-obico.cfg

export PATH=/opt/bin:/opt/sbin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
export LD_LIBRARY_PATH=/opt/lib
export PYTHONPATH=/usr/data/moonraker-obico

exec /usr/data/moonraker-obico-env/bin/python3 \
  -B -m moonraker_obico.app -c "$CONF_FILE"
```

Using `exec` makes the PID recorded by `start-stop-daemon` become the actual Python process rather than a waiting wrapper shell.

The stock K1 init script does not implement `stop`, so avoid using its `restart` action. For the first controlled start, check whether Obico is already running:

```sh
ps | grep '[m]oonraker_obico.app'
```

If there is no result, start it:

```sh
/etc/init.d/S99moonraker_obico start
```

If it is already running, rebooting is the cleanest way to validate both the updated launcher and automatic startup:

```sh
reboot
```

## Post-start validation

Moonraker-Obico starts at boot, but Janus may be started lazily when an Obico web or mobile session first requests streaming. Immediately after a reboot it is therefore valid to see Moonraker-Obico running without a Janus process or listener on port `17730`.

After starting or rebooting, allow approximately 30 seconds and first confirm Moonraker-Obico:

```sh
ps | grep -E '[m]oonraker_obico.app|[j]anus'

netstat -lnt 2>/dev/null | grep ':17730 '

tail -n 200 /usr/data/printer_data/logs/moonraker-obico.log | \
  grep -E 'janus_bin_path|JANUS:|Streaming|WebSocket|ERROR|Exception'
```

If Janus and port `17730` are absent, open the printer in Obico and request its live camera stream, then repeat the process and socket checks. This lazy start is expected; failure is only indicated if Janus still does not start or Obico continues to show a streaming error after the stream has been requested.

Janus subprocess lines are emitted through Moonraker-Obico's `DEBUG` logging. With the normal `INFO` level, the filtered log command may return no output even when streaming is working. In that case, validate the processes, socket and generated configuration directly.

If `netstat` is unavailable, use whichever socket utility the image provides:

```sh
ss -lnt 2>/dev/null | grep ':17730 '
```

Inspect the generated configuration and confirm it references the installed modules:

```sh
RUNTIME_ETC=/usr/data/moonraker-obico/moonraker_obico/bin/janus/runtime/etc/janus

find "$RUNTIME_ETC" -maxdepth 1 -type f -print
grep -R -E 'plugins_folder|transports_folder|ws_port|mjpeg-|dataport' "$RUNTIME_ETC"
```

Expected results include:

- Moonraker-Obico running automatically after boot.
- After requesting a live stream, Moonraker-Obico and the custom Janus process both running.
- After requesting a live stream, Janus WebSockets listening only on `127.0.0.1:17730`.
- Successful loading of `libjanus_streaming.so` and `libjanus_websockets.so`.
- An `mjpeg-2` mount with `video = false` and `data = true`.
- A live camera in the Obico web or mobile interface.

Check resource use while viewing the stream and again during a test print:

```sh
free -m
top
```

If load or bandwidth is excessive, reduce `target_fps` to `5` before changing any Janus settings.

## Rollback

To disable video without unlinking the printer, set:

```ini
[webcam]
disable_video_streaming = True
```

To remove only the custom Janus discovery shim, first print and inspect the exact directory again, then remove the `bin/janus` and `lib` symlinks inside that one variant directory. Do not remove `/opt` or the Moonraker-Obico tree.

## Verified final state

The complete build, installation and integration have been verified on the target K1. Moonraker-Obico starts automatically after reboot, discovers the custom Janus build, and starts Janus on demand when an Obico session requests the camera. The previous Obico streaming error is cleared and the `mjpeg_webrtc` feed operates through the generated DataChannel mount.

Longer-term resource measurements while viewing the stream during a print remain advisable, particularly at `target_fps = 15`.
