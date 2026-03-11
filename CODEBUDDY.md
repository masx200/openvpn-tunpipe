# CODEBUDDY.md

This file provides guidance to CodeBuddy Code when working with code in this repository.

## Project Overview

This is OpenVPN - a secure tunneling daemon that tunnels IP networks over a single TCP/UDP port with SSL/TLS-based session authentication, key exchange, packet encryption, authentication, and compression.

**Note:** This appears to be a fork named "openvpn-tunpipe".

## Build Commands

### Unix/Linux/macOS (Autotools)

From source repository checkout:
```bash
autoreconf -i -v -f
./configure
make
make install
```

From tarball:
```bash
./configure
make
make install
```

Build a tarball from repository:
```bash
autoreconf -i -v -f
./configure
make distcheck
```

### Windows (MSVC)

Use MSBuild with the provided solution file:
```bash
msbuild /m /p:Configuration=Release /p:Platform=x64 .
```

Supported platforms: ARM64, Win32 (x86), x64

### Cross-compilation (MinGW)

```bash
./configure --host=x86_64-w64-mingw32 --disable-lz4
make
```

## Testing

Run all tests:
```bash
make check
```

Unit tests require the cmocka framework. Tests are located in `tests/unit_tests/`.

Individual test drivers include:
- `crypto_testdriver` - Crypto functionality tests
- `packet_id_testdriver` - Packet ID tests
- `auth_token_testdriver` - Auth token tests
- `ncp_testdriver` - NCP tests
- `misc_testdriver` - Miscellaneous tests
- `tls_crypt_testdriver` - TLS crypt tests (requires ld wrap support)
- `argv_testdriver` - Argument parsing tests (requires ld wrap support)
- `buffer_testdriver` - Buffer tests (requires ld wrap support)

## Code Style

Code is formatted using Uncrustify. Configuration file: `dev-tools/uncrustify.conf`

Format all code:
```bash
./dev-tools/reformat-all.sh
```

Pre-commit hook available: `dev-tools/git-pre-commit-uncrustify.sh`

## Configure Options

Key configure options:
- `--with-crypto-library=openssl|mbedtls|wolfssl` - Choose crypto backend (default: openssl)
- `--disable-lzo` / `--disable-lz4` - Disable compression support
- `--enable-dco` - Enable data channel offload (Linux only)
- `--enable-iproute2` - Use iproute2 instead of ifconfig/route
- `--enable-systemd` - Enable systemd integration
- `--disable-management` - Disable management interface
- `--disable-plugins` - Disable plugin support
- `--enable-strict` - Enable strict compiler warnings
- `--enable-werror` - Treat warnings as errors

## Architecture

### Source Directory Structure

```
src/
├── openvpn/          # Main OpenVPN daemon (core VPN implementation)
├── openvpnserv/      # Windows service wrapper
├── openvpnmsica/     # Windows MSI custom actions
├── tapctl/           # Windows TAP adapter control utility
├── compat/           # Compatibility layer for missing functions
└── plugins/          # Plugin modules
    ├── auth-pam/     # PAM authentication plugin
    └── down-root/    # Root privilege dropping plugin
```

### Core Components (src/openvpn/)

**Event Loop & I/O:**
- `openvpn.c` - Main entry point and client (P2P) event loop
- `multi.c` - Server mode supporting multiple clients
- `forward.c` - Packet forwarding logic
- `event.c` - Event loop abstraction
- `io_wait()` pattern for async I/O

**Networking:**
- `socket.c` / `socket.h` - Socket handling
- `tun.c` / `tun.h` - TUN/TAP device interface
- `route.c` / `route.h` - Routing table management
- `networking_sitnl.c` - Linux internal networking API
- `networking_iproute2.c` - iproute2 backend

**Security Layer:**
- `ssl.c` / `ssl.h` - TLS control channel (crypto-agnostic)
- `ssl_openssl.c` / `ssl_mbedtls.c` - Backend implementations
- `ssl_verify.c` - Certificate verification
- `crypto.c` / `crypto.h` - Data channel encryption (crypto-agnostic)
- `crypto_openssl.c` / `crypto_mbedtls.c` - Backend implementations
- `tls_crypt.c` - TLS authentication wrapping

**Key Data Structures:**
- `buffer.c` / `buffer.h` - Buffer management (core data structure)
- `packet_id.c` - Anti-replay packet IDs
- `reliable.c` - Reliable transport for control channel

**Configuration:**
- `options.c` / `options.h` - Command-line option parsing
- `init.c` - Initialization and context setup

**Platform Abstraction:**
- `syshead.h` - System headers and platform detection
- `win32.c` / `win32.h` - Windows-specific code
- `compat/` - Compatibility shims for missing functions

### Crypto Backends

OpenVPN supports three crypto libraries via a backend abstraction:

1. **OpenSSL** (default) - `crypto_openssl.c`, `ssl_openssl.c`
2. **mbed TLS** - `crypto_mbedtls.c`, `ssl_mbedtls.c`
3. **wolfSSL** - Uses OpenSSL compatibility layer

Backend selection at compile time via `--with-crypto-library`. Header `crypto_backend.h` defines the backend interface.

### Plugin System

Plugins are shared libraries loaded at runtime. Plugin API defined in `include/openvpn-plugin.h.in`.

Built-in plugins:
- `auth-pam` - PAM authentication (Unix only)
- `down-root` - Execute down scripts with root privileges

### Windows Components

- `openvpnserv/` - Windows service that manages OpenVPN instances
- `tapctl/` - Utility for managing TAP adapter drivers
- `openvpnmsica/` - MSI installer custom actions
- `block_dns.c` - Windows DNS blocking for VPN

## Dependencies

Required:
- OpenSSL 1.0.2+ / mbed TLS 2.x / wolfSSL
- TUN/TAP driver

Optional:
- LZO / LZ4 - Compression
- libpkcs11-helper - PKCS#11 smart card support
- libpam - PAM authentication plugin
- libsystemd - systemd integration
- cmocka - Unit testing
