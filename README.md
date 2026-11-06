# OpenVPN Transport Obfuscation

UDP obfuscation, TCP Scrambler and legacy WebSocket-over-TLS (WSS) patches for
OpenVPN **2.7.9**. The OpenSSL backend is required.

Need a prebuilt client? A ready-to-use build is available at
[here](https://github.com/aw3r1se/ovpn-obfuscate).

## Apply patches

### Windows

Use a clean OpenVPN checkout. These commands assume this repository is at
`C:\src\ovpn-obfuscate`. Run them in an **x64 Native Tools Command Prompt**
(`cmd.exe`) for your installed Visual Studio.

```bat
git clone https://github.com/OpenVPN/openvpn.git C:\src\openvpn
cd /d C:\src\openvpn
git checkout --detach v2.7.9
git apply --check C:\src\ovpn-obfuscate\general\0001-transport-core.patch
git apply C:\src\ovpn-obfuscate\general\0001-transport-core.patch
```

Apply only the modules you need. All four patches apply in any order. Each
module's patch only adds source files; `general` is what teaches the build
system to detect and compile them, so a module without `general` applies
cleanly but stays out of the build.

UDP obfuscation:

```bat
git apply C:\src\ovpn-obfuscate\udp\0001-enable-udp.patch
```

TCP Scrambler:

```bat
git apply C:\src\ovpn-obfuscate\tcp\0001-enable-tcp.patch
```

WSS:

```bat
git apply C:\src\ovpn-obfuscate\wss\0001-enable-wss.patch
```

TCP Scrambler over WSS needs both `tcp` and `wss`. Apply all three modules for
a client supporting every mechanism. CMake and `configure` detect installed
modules from their source files, so unselected implementations stay out of the
build. Apply each patch once; stop if Git reports an error. After adding a
module, configure and build again so detection runs with the new files.

### Linux and macOS

Place this patch repository at `~/src/ovpn-obfuscate` and use a clean checkout:

```sh
mkdir -p ~/src
git clone https://github.com/OpenVPN/openvpn.git ~/src/openvpn
cd ~/src/openvpn
git checkout --detach v2.7.9
git apply --check ../ovpn-obfuscate/general/0001-transport-core.patch
git apply ../ovpn-obfuscate/general/0001-transport-core.patch
```

Run only the lines for the modules you need:

```sh
git apply ../ovpn-obfuscate/udp/0001-enable-udp.patch
git apply ../ovpn-obfuscate/tcp/0001-enable-tcp.patch
git apply ../ovpn-obfuscate/wss/0001-enable-wss.patch
```

## Build

### Windows

Install Git, Python 3, CMake 3.21+, and a Visual Studio C++ toolset
(2022 or 2026) with Spectre-mitigated libraries and a Windows SDK. Run the
commands below from that toolset's **x64 Native Tools Command Prompt** so the
SDK message compiler (`mc.exe`) is available. Builds are verified with the
VS 2022 generator (`win-amd64-release`, MSBuild 17.x); a VS 2026 toolset
builds the same presets.

```bat
git clone https://github.com/microsoft/vcpkg.git C:\src\vcpkg
C:\src\vcpkg\bootstrap-vcpkg.bat
cd /d C:\src\openvpn
cmake --preset win-amd64-release -DCMAKE_TOOLCHAIN_FILE=C:/src/vcpkg/scripts/buildsystems/vcpkg.cmake -DBUILD_TESTING=ON
cmake --build --preset win-amd64-release --parallel
ctest --test-dir out\build\win-amd64-release -C Release -R "^test_obfs_(udp|tcp|wss)$" --output-on-failure
```

The executable and runtime DLLs are in `out\build\win-amd64-release\Release`.
Keep those DLLs alongside `openvpn.exe`. Tests are added only for installed modules.

### Linux (Debian / Ubuntu)

Linux and macOS use the [upstream Autotools build](https://github.com/OpenVPN/openvpn/blob/v2.7.9/INSTALL).
Run these commands after applying the patches:

```sh
sudo apt-get update
sudo apt-get install build-essential autoconf automake libtool pkg-config libssl-dev liblzo2-dev liblz4-dev libcap-ng-dev libnl-3-dev libnl-genl-3-dev libcmocka-dev libpam0g-dev
cd ~/src/openvpn
autoreconf -fi
./configure --with-crypto-library=openssl --disable-dco
make -j4
make check RUN_SUDO="sudo -n"
./src/openvpn/openvpn --version
```

### macOS

Install Xcode Command Line Tools and [Homebrew](https://brew.sh), then build
against Homebrew OpenSSL 3. The paths below are for Apple Silicon:

```sh
xcode-select --install
brew install autoconf automake libtool pkgconf openssl@3 lzo lz4 cmocka
cd ~/src/openvpn
LIBTOOLIZE=glibtoolize autoreconf -fi
PKG_CONFIG_PATH=/opt/homebrew/opt/openssl@3/lib/pkgconfig:/opt/homebrew/lib/pkgconfig ./configure --with-crypto-library=openssl --disable-dco
make -j4
make check
./src/openvpn/openvpn --version
```

On Intel Macs, replace `/opt/homebrew` with `/usr/local`. The Unix executable
is `src/openvpn/openvpn`; installation is optional. `make check` runs the
upstream tests and the tests for installed transport modules.
Linux networking tests skip when noninteractive sudo is unavailable.

## Usage

[example.ovpn](example.ovpn) contains all 11 connection variants in one file.
Copy it to `client.ovpn`, uncomment one connection block, and replace the
example endpoint, paths and keys. Optional inline keys and cipher settings
are included at the end of the file.

Keep the endpoint, credentials, cipher and other options from your provider's
configuration. A common configuration looks like this:

```openvpn
client
dev tap
remote vpn.example.invalid 443
ca certs/ca.crt
cert certs/client.crt
key certs/client.key
remote-cert-tls server
auth-nocache
nobind
persist-key
persist-tun
disable-dco
verb 3
```

Replace the example address and paths. Windows `dev tap` needs a TAP-Windows
adapter. On macOS, native `utun` supports `dev tun`; a TAP configuration needs
a separate compatible TAP driver. Use a server profile matching the device
type: changing `dev` alone does not convert a TAP server to TUN. Keep
`disable-dco` explicit in shared profiles. The patched client also disables
DCO automatically when an obfuscation transport is configured.

### TCP Scrambler

```openvpn
proto tcp-client
obfuscate TCP_KEY
```

Replace `TCP_KEY` with the provider's exact key string; it is used as text.

### UDP obfuscation

```openvpn
proto udp
obfuscate-udp UDP_HEX_KEY MAX_PADDING_BYTES
```

`UDP_HEX_KEY` must contain 128 hexadecimal characters. Set `MAX_PADDING_BYTES`
to an integer from **0 to 1500**, for example `32`. It limits additional padding
in bytes; the actual amount varies by packet. Obfuscated datagrams are limited
to 1472 bytes. Change this setting in the configuration without rebuilding.
The encoded packet must be at least 32 bytes. If the padding limit cannot
reach that minimum, the client rejects the packet; `0` can therefore fail
with short control packets.

### WSS

```openvpn
proto tcp-client
wss-proxy front.example.invalid /provider-path
wss-proxy-sni front.example.invalid
```

`remote` is the connection address; `wss-proxy` sets the HTTP Host and URI.
Use `wss-proxy-sni` when the provider requires an outer TLS name. Certificate
verification checks that name when supplied, otherwise the HTTP Host. This
legacy protocol uses little-endian extended frame lengths.

`wss-proxy-allow-self-signed-cert` disables outer TLS certificate verification.
Use it only if required by the provider configuration. Inner OpenVPN certificate
verification is separate.

`wss-proxy-header` overrides a default Upgrade request header or adds a new one.
Repeat it up to 20 times:

```openvpn
wss-proxy-header User-Agent "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X)"
wss-proxy-header X-Custom-Header REPLACE_WITH_PROVIDER_VALUE
```

The first parameter is the header name; it cannot contain `:`, whitespace or a
line break, and is limited to 128 characters. The optional second parameter is
the value, limited to 1024 characters. Quote a value that contains spaces. A
name matching a default header replaces it; any other name appends a new line.
Omitting the value suppresses the header instead of sending it empty: the line
is left out of the request entirely, which is the way to drop a default header
such as `Origin`. Configured headers replace `Host`, `User-Agent`, `Origin` and
`Sec-WebSocket-Key` as well, so a custom `Sec-WebSocket-Key` also changes the
expected `Sec-WebSocket-Accept`.

### tls-crypt-v2

Copy the provider-issued **client key** into the configuration:

```openvpn
<tls-crypt-v2>
-----BEGIN OpenVPN tls-crypt-v2 client key-----
BASE64_CLIENT_KEY_DATA
-----END OpenVPN tls-crypt-v2 client key-----
</tls-crypt-v2>
```

Replace `BASE64_CLIENT_KEY_DATA` with the complete, unchanged key body. Keep
the BEGIN/END lines. Use the key issued for that server, not a server key or
a newly generated replacement.

Alternatively, save the same key, including its BEGIN/END lines, in a file:

```openvpn
tls-crypt-v2 tls-crypt-v2-client.key
```

Use either the inline block or the file directive, not both. The relative file
path resolves from the directory passed to `--cd`. Do not combine `tls-crypt-v2`
with `tls-auth` or `tls-crypt` in a client configuration.

This native OpenVPN option needs no patch by itself. It can also accompany
UDP obfuscation or WSS when required by the server, as listed below.

### Combinations

Add the options below to the corresponding TCP, UDP or WSS settings above.
For `tls-crypt`, use `tls-crypt tls-crypt.key` or the provider's existing
inline `<tls-crypt>` block. For `tls-crypt-v2`, use either form shown above.

| Connection | Options | Modules |
| --- | --- | --- |
| TCP + tls-crypt-v2 | `proto tcp-client`, `tls-crypt-v2` | None |
| TCP Scrambler | `obfuscate` | TCP |
| TCP Scrambler + tls-crypt | `obfuscate`, `tls-crypt` | TCP |
| TCP over WSS | WSS settings | WSS |
| TCP over WSS + tls-crypt | WSS settings, `tls-crypt` | WSS |
| TCP over WSS + tls-crypt-v2 | WSS settings, `tls-crypt-v2` | WSS |
| TCP Scrambler over WSS | WSS settings, `obfuscate` | TCP + WSS |
| UDP + tls-crypt-v2 | `proto udp`, `tls-crypt-v2` | None |
| UDP obfuscation | `obfuscate-udp` | UDP |
| UDP obfuscation + tls-crypt | `obfuscate-udp`, `tls-crypt` | UDP |
| UDP obfuscation + tls-crypt-v2 | `obfuscate-udp`, `tls-crypt-v2` | UDP |

Native `tls-crypt-v2` alone needs no patches. Each module listed above requires
`general` before building. TCP and UDP obfuscation cannot be used in the same
connection: the client rejects the combination at startup, and the same link
socket would otherwise need two different encodings.

### Connect

**Windows:** save the configuration as `C:\VPN\client.ovpn`. Relative certificate
and key paths resolve from `C:\VPN`. Open Command Prompt as Administrator:

```bat
cd /d C:\src\openvpn\out\build\win-amd64-release\Release
openvpn.exe --cd C:\VPN --config client.ovpn
```

**Linux / macOS:** save the configuration and its relative key/certificate
paths under `~/vpn`, then run the built executable:

```sh
sudo ~/src/openvpn/src/openvpn/openvpn --cd ~/vpn --config client.ovpn
```

Wait for `Initialization Sequence Completed`. Stop with Ctrl+C.

### Data cipher

Obfuscation does not select the data cipher. Keep the provider's settings or
choose AES-256-GCM, AES-128-GCM, AES-256-CBC, AES-128-CBC or BF-CBC if supported
by the server. For example, from the same `Release` directory:

```bat
openvpn.exe --cd C:\VPN --config client.ovpn --data-ciphers AES-128-CBC
```

For servers without cipher negotiation, also set `--data-ciphers-fallback` to
the same cipher. BF-CBC needs the OpenSSL legacy provider. On Linux/macOS,
add `providers legacy default` to the configuration. On Windows, set the
matching `legacy.dll` directory before starting the client:

```bat
set "OPENSSL_MODULES=C:\src\openvpn\out\build\win-amd64-release\vcpkg_installed\x64-windows-ovpn\bin"
openvpn.exe --cd C:\VPN --config client.ovpn --providers legacy default --data-ciphers BF-CBC --data-ciphers-fallback BF-CBC
```

## Checks and license

Build and test results for **2.7.9** on Windows x64:

- Patches applied cleanly in forward and reverse order.
- A patched tree is byte-identical to the reference tree.
- Module subsets build; unselected modules stay out of the binary.
- `openvpn.exe` exposes `--obfuscate`, `--obfuscate-udp` and the `--wss-proxy*`
  options only for the modules that were applied.
- All 21 upstream unit tests pass, including `test_obfs_udp`, `test_obfs_tcp`
  and `test_obfs_wss`.
- Live connections against provider servers succeeded for all four transports:
  UDP obfuscation, TCP Scrambler, WSS, and TCP Scrambler over WSS. Each brought
  the tunnel up and passed traffic.

Environment: OpenSSL 3.6.2, MSBuild 17.14 (VS 2022 generator),
`x64-windows-ovpn` triplet.

Older results from the earlier 2.7.4/2.7.8 patch series (Autotools builds on
Ubuntu WSL with `--enable-werror`, ASan/UBSan/leak runs on 2.7.4) are not
reproduced here: those patches have since been regenerated against 2.7.9.

Re-run the checks yourself after configuring with `-DBUILD_TESTING=ON`:

```bat
cmake --preset win-amd64-release -DCMAKE_TOOLCHAIN_FILE=<vcpkg>/scripts/buildsystems/vcpkg.cmake -DBUILD_TESTING=ON
cmake --build --preset win-amd64-release --parallel
ctest --test-dir out\build\win-amd64-release -C Release -R "^test_obfs_" --output-on-failure
```

macOS and Linux builds have not been tested against this patch series.

License terms: [GPL version 2](LICENSE), with upstream exceptions in [COPYING](COPYING).

## How it works

The transport code processes complete OpenVPN packets after OpenVPN encryption
on send, and before OpenVPN packet handling on receive. It does not replace
OpenVPN authentication or data-channel encryption. Native `tls-crypt` and
`tls-crypt-v2` protect control packets before transport obfuscation.

### UDP wire format

The 64-byte key contains two 32-byte halves: transmit, then receive. Each half
is an AES-128-XTS key. Its SHA-256 digest supplies the padding marker and the
32-bit cascade key. Sending applies four steps:

1. When padding is added, append random bytes, set the `0x80` padding flag,
   and encode the boundary with a keyed XOR marker derived from the final length.
2. Apply the 32-bit XOR cascade using multiplier `0xCC9E2D51`.
3. Apply AES-128-XTS. The final 16 bytes supply the tweak and are excluded from
   this step. Data packets with opcode 6 or 9 and length at least 48 encrypt
   only the first 16 bytes; other packets encrypt everything before the tweak.
4. Apply the 64-bit XOR cascade using multiplier `0x87C37B91114253D5`.

Receive reverses these steps and removes padding using the marker. Padding
fits within the configured limit and a maximum datagram size of 1472 bytes.

### TCP Scrambler

A random seed from 0 to 254 rotates the text key. MD5 digests of its 16-byte
chunks form a repeating XOR mask. Transmit and receive keep separate byte
counters across packets. The first byte of the first client packet is replaced
by the seed before the bit permutation. That byte carries opcode 7
(`P_CONTROL_HARD_RESET_CLIENT_V2`, `0x38`) or opcode 10
(`P_CONTROL_HARD_RESET_CLIENT_V3`, `0x50`), and the replacement destroys it, so
the client rejects any other first byte. Only the client's first packet is
affected: the decoder never looks for a seed, and the server's first byte
arrives intact. Each packet's alternating bits are redistributed across
half-packet offsets, then its byte order is reversed.

On plain TCP, the 16-bit network-order packet length becomes
`length + 5621 * k`, with random `k` from 0 to 6 when it fits. Receive removes
the length offset and reverses the packet transformation. State resets on
reconnection.

### WSS framing

The client establishes outer TLS and sends an HTTP WebSocket Upgrade request.
It checks status 101, the Upgrade and Connection headers, and Sec-WebSocket-Accept.
Each OpenVPN packet becomes a FIN binary frame with a fresh four-byte client
mask. Extended frame lengths use **little-endian** encoding, unlike standard
WebSockets. Server frames must be unmasked and unfragmented; ping/pong is handled.
OpenSSL memory BIOs connect TLS to OpenVPN's socket I/O. A bounded output
queue retains encrypted bytes across partial writes and socket backpressure.

With Scrambler enabled, the send path is:

```text
OpenVPN packet -> TCP Scrambler -> masked WebSocket frame -> TLS -> TCP
```

WSS carries packet boundaries, so the plain TCP length prefix and its masking
are omitted. Receive removes TLS and WebSocket framing before running the
Scrambler decoder.
