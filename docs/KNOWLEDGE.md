# PIVAS — Project Knowledge

## Project Overview

PIVAS (fork of КВАС/KVAS) is a VPN routing and DNS filtering package for Keenetic routers, distributed as an Entware `.ipk` package. It is written entirely in POSIX shell (~11,000 lines across 20+ files) with no compiled code.

## Build System

This is an OpenWrt/Entware package. Building requires the Entware toolchain — it is not built locally with standard shell tools.

**CI/CD (GitHub Actions):** Push to `main` triggers `.github/workflows/build-release.yml`, which:
1. Builds the Entware toolchain in Docker (Debian 11, `builder/Dockerfile`)
2. Compiles the `.ipk` package for `aarch64`
3. Creates a GitHub release with the `.ipk` artifact

**Local build** (requires Docker + Entware toolchain clone):
```sh
# See builder/builder script for the full orchestration
```

**Package version** is defined in `Makefile`:
```makefile
PKG_VERSION:=1.1.9_beta-10
PKG_RELEASE:=25
```

The version is injected into `/opt/etc/kvas.conf` at install time via `sed` in `Package/kvas/postinst`.

**There is no formal test suite.** Manual testing uses `kvas test` and `kvas debug` commands on the target router.

## Architecture

### Entry Point

`opt/bin/kvas` — the main CLI dispatcher. It sources all libraries on startup, checks setup completion, then dispatches to command handlers via a `case` statement.

### Directory Layout

```
opt/bin/
├── kvas          # Main CLI entry point (command dispatcher)
├── libs/         # Sourced libraries (never executed directly)
│   ├── main      # Core helpers: print_line, ready, when_ok, when_bad, get_config_value, error
│   ├── vpn       # VPN routing logic: iptables chains, ipset rules, interface management
│   ├── debug     # Debug output commands
│   ├── adblock   # Ad blocking integration
│   ├── check     # Health checks for DNS/VPN services
│   ├── route     # Static routing rules
│   ├── tags      # Domain tagging system
│   ├── vless     # VLESS/Xray protocol support
│   ├── update    # Self-update helpers
│   ├── hosts     # /etc/hosts management
│   ├── ndm_d     # Keenetic NDM daemon integration
│   └── keen_api  # Keenetic HTTP API client
└── main/         # Standalone scripts (executed via sh, not sourced)
    ├── setup     # Interactive first-run wizard
    ├── upgrade   # Package upgrade/rollback
    ├── adblock   # Ad block list management
    ├── ipset     # ipset table management
    ├── dnsmasq   # dnsmasq config generation
    ├── adguard   # AdGuardHome integration
    ├── check_vpn # VPN connectivity checker
    └── update    # Update command

opt/etc/
├── conf/         # Configuration templates
├── init.d/       # SysV init scripts (S96kvas, S97xray, S99adguard)
└── ndm/          # Keenetic NDM event hooks
    ├── fs.d/
    ├── ifstatechanged.d/
    ├── ifcreated.d/
    ├── ifdestroyed.d/
    ├── netfilter.d/
    ├── wan.d/
    └── iflayerchanged.d/
```

### Runtime Configuration

- `/opt/etc/kvas.conf` — main config (key=value), read via `get_config_value`
- `/opt/etc/kvas.list` — whitelist domains
- `/opt/etc/kvas.ovpn.list` — OpenVPN split-tunnel list
- `/opt/etc/dnsmasq.d/kvas.dnsmasq` — generated dnsmasq rules

### DNS Stack

Two mutually exclusive modes:
- **dnsmasq mode:** Port 9753 (dnsmasq) → port 9153 (dnscrypt-proxy2) → upstream
- **AdGuardHome mode:** Replaces dnsmasq as the DNS resolver

### VPN Routing

Traffic for domains in the whitelist is routed via VPN using:
- `ipset` tables (`KVAS_LIST`, `KVAS_OVPN_LIST`) populated from dnsmasq DNS responses
- `iptables` chains that redirect matched traffic to the VPN interface or Shadowsocks proxy

Supported VPN protocols: Shadowsocks (ss-redir), OpenVPN, VLESS/Xray, IKEv2.

### Keenetic NDM Integration

The package hooks into Keenetic's NDM event system via scripts in `/opt/etc/ndm/`. These reinstate iptables rules and ipset entries when the router's firewall is reset or interfaces change state.

## Code Conventions

**Function naming:**
- `cmd_*` — user-facing command implementations
- `get_*` — read/query (e.g., `get_config_value "KEY"`)
- `is_*` / `has_*` — boolean checks (return 0/1)
- `setup_*` — installation/configuration steps
- `update_*` — apply/refresh state (e.g., `update_iptables`)
- `ip4_*` — IPv4-specific operations

**Output helpers** (from `libs/main`):
```sh
ready "Starting operation"   # Pre-action status
when_ok "SUCCESS"            # Green success
when_bad "ERROR"             # Red failure
error "message" "exit_code"  # Red + optional exit
print_line                   # 68-char separator
```

**Color variables:** `RED`, `GREEN`, `BLUE`, `YELLOW`, `NOCL` — defined in `libs/main`.

**POSIX sh only** — no `[[`, no bash arrays, no process substitution. All conditionals use `[`.

**Variable scope:** ALL_CAPS for globals/config constants, lowercase_underscores for locals.

**Background jobs:**
```sh
/opt/apps/kvas/bin/main/ipset &>/dev/null &
[ $? = 0 ] && when_ok "OK" || when_bad "FAIL"
```
