# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an OpenVPN fork with custom modifications, primarily adding a "pipe to external program" device type. This feature allows tunneled traffic to be handled by an external program rather than a real tun/tap kernel device, enabling non-root users to connect to a VPN through a userland TCP/IP stack.

## Build Commands

### From Tarball
```bash
./configure && make && make install
```

### From Git Repository
```bash
autoreconf -i -v -f
./configure
make
make install
```

### Testing
```bash
make check          # Run all tests (stops on failure)
make -k check       # Run all tests regardless of errors
```

### Test Crypto
```bash
./openvpn --genkey secret key
./openvpn --test-crypto --secret key
```

### Test SSL/TLS (2 minutes)
```bash
# Terminal 1:
./openvpn --config sample/sample-config-files/loopback-client

# Terminal 2 (simultaneously):
./openvpn --config sample/sample-config-files/loopback-server
```

## Architecture

### Main Entry Point
- `src/openvpn/openvpn.c` - Contains `openvpn_main()` with the primary init-run-cleanup loop structure
  - Outer loop: runs at startup and once per SIGHUP
  - Inner loop: runs at startup and once per SIGUSR1
  - Two main tunnel modes: `MODE_POINT_TO_POINT` (tunnel_point_to_point) and `MODE_SERVER` (tunnel_server)

### Context Structure (src/openvpn/openvpn.h)
OpenVPN uses a three-level context hierarchy for state management:
- **context_0 (c0)**: Process-level state, initialized once at startup (PID, user, group, privileges)
- **context_1 (c1)**: State persisting across SIGUSR1 restarts (tun/tap, routes, keys, session state)
- **context_2 (c2)**: State reset on both SIGHUP and SIGUSR1 (event loops, timers, TLS state, buffers)

### Core Components

**Data Channel:**
- `src/openvpn/tun.c` - TUN/TAP device handling (including custom pipe device)
- `src/openvpn/forward.c` - Packet forwarding and I/O processing
- `src/openvpn/crypto.c` - Data channel encryption/decryption (with OpenSSL/mbedTLS backends)
- `src/openvpn/buffer.c` - Buffer management for packet data

**Control Channel (TLS):**
- `src/openvpn/ssl.c` - TLS state machine and key exchange
- `src/openvpn/ssl_openssl.c` / `ssl_mbedtls.c` - Crypto library backends
- `src/openvpn/ssl_verify.c` - Certificate verification
- `src/openvpn/tls_crypt.c` - Control channel authentication/encryption

**Event Loop:**
- `src/openvpn/event.c` - Event set management and I/O multiplexing
- `src/openvpn/socket.c` - Network socket operations (TCP/UDP)
- `src/openvpn/multi.c` - Multi-client server handling

**Configuration:**
- `src/openvpn/options.c` - Command-line and config file parsing (300K lines)
- `src/openvpn/init.c` - Initialization and context setup

### Custom Modifications

This fork includes a custom "pipe to external program" device:
- Enabled via `--dev-type pipe --dev <program>`
- Traffic is piped to/from an external program instead of a tun/tap device
- See `src/openvpn/tun.c` around `open_tun()` for the implementation
- Modified `CONNECTION_LIST_SIZE` increased from 64 to 512 (src/openvpn/options.h)

## Important Build-Time Options

Key configure options:
- `--disable-lzo` / `--disable-lz4` - Disable compression
- `--disable-plugins` - Disable plugin support
- `--enable-small` - Smaller executable size
- `--enable-strict` / `--enable-werror` - Stricter compiler warnings
- `--enable-pkcs11` - Enable PKCS#11 support

Crypto backends (one required):
- OpenSSL (default) - `ENABLE_CRYPTO_OPENSSL`
- mbedTLS - `ENABLE_CRYPTO_MBEDTLS`

## File Locations

- Source code: `src/openvpn/`
- Plugins: `src/plugins/`
- Unit tests: `tests/unit_tests/`
- Sample configs: `sample/sample-config-files/`
- Documentation: `doc/`

## Memory Management

OpenVPN uses garbage collection arenas (`gc_arena`) for scoped allocations:
- `struct context` has a `gc` field for context-scoped allocations
- `struct context_2` has its own `gc` for level-2 scoped allocations
- Use `gc_malloc()` / `gc_strdup()` for allocations that should be auto-freed

## Signal Handling

Three key signals:
- `SIGHUP` - Reload configuration (Level 1 cleanup/reinit)
- `SIGUSR1` - Restart tunnel (Level 2 cleanup/reinit, keeps connections)
- `SIGTERM` - Clean shutdown

See `src/openvpn/sig.c` for signal handling implementation.
