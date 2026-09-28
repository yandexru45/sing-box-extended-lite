# sing-box-extended-lite

[![Build and Release Lite](https://github.com/yandexru45/sing-box-extended-lite/actions/workflows/build-and-release.yml/badge.svg)](https://github.com/yandexru45/sing-box-extended-lite/actions/workflows/build-and-release.yml)
[![GitHub Release](https://img.shields.io/github/v/release/yandexru45/sing-box-extended-lite?style=flat&color=3388ff)](https://github.com/yandexru45/sing-box-extended-lite/releases/latest)
[![License: GPL-3.0](https://img.shields.io/badge/License-GPL--3.0-blue.svg)](LICENSE)

Lightweight ("lite") builds of [shtorm-7/sing-box-extended](https://github.com/shtorm-7/sing-box-extended),
compiled automatically for every new stable upstream release. This is the
"third core" for **NetShift** (alongside stock sing-box and the full
extended build): a reduced build-tag set plus UPX compression gives a
**~9.5–12.7 MB** binary instead of the ~105 MB upstream monolith, targeting
routers with small flash.

There is **no installer in this repository** — NetShift's backend downloads
and installs these binaries itself. If you are not NetShift, just grab an
asset from [Releases](https://github.com/yandexru45/sing-box-extended-lite/releases/latest)
and unpack `sing-box` (mode 0755) wherever you need it.

---

## Version scheme / how to detect a lite build

For upstream tag `v1.14.1-extended-2.7.2` this repo publishes release
`v1.14.1-extended-2.7.2-lite`, and the binary reports:

```
$ sing-box version
sing-box version 1.14.1-extended-2.7.2-lite
...
```

The version string is the release tag without the leading `v`. **A trailing
`-lite` in `sing-box version` output is the marker of this build** — NetShift
relies on it.

## Assets

Asset names are **version-less** (stable download URLs):

| Asset | Contents | Typical size |
| :--- | :--- | ---: |
| `sing-box-extended-lite-linux-amd64.tar.gz` | pure ELF binary `sing-box` (0755) | ~17 MB |
| `sing-box-extended-lite-linux-amd64-compressed.tar.gz` | same binary, UPX-packed | ~12 MB |
| `sing-box-extended-lite-linux-arm64.tar.gz` | pure ELF | ~15 MB |
| `sing-box-extended-lite-linux-arm64-compressed.tar.gz` | UPX-packed | ~10 MB |
| `sing-box-extended-lite-linux-armv7.tar.gz` | pure ELF (GOARM=7) | ~16 MB |
| `sing-box-extended-lite-linux-armv7-compressed.tar.gz` | UPX-packed | ~10 MB |
| `sing-box-extended-lite-linux-mips-softfloat.tar.gz` | pure ELF (GOMIPS=softfloat) | ~15 MB |
| `sing-box-extended-lite-linux-mips-softfloat-compressed.tar.gz` | UPX-packed | ~9.5 MB |
| `sing-box-extended-lite-linux-mipsle-softfloat.tar.gz` | pure ELF (GOMIPS=softfloat) | ~15 MB |
| `sing-box-extended-lite-linux-mipsle-softfloat-compressed.tar.gz` | UPX-packed | ~9.5 MB |
| `sha256sums.txt` | SHA-256 of every asset above | — |

All binaries are static (`CGO_ENABLED=0`), `GOOS=linux` only.

## Build tags: what is kept / what is stripped

The full upstream build compiles a large set of enterprise/desktop features
into one monolithic binary (95–105 MB). This lite build keeps the
router/VPN core only:

### ✅ Kept

- **VLESS** incl. **VLESS Encryption** (`encryption: mlkem768x25519plus...`), Reality, XTLS Vision
- **XHTTP transport** (incl. extended `x_padding_bytes` options), vmess, Trojan, Shadowsocks
- **Hysteria2 / TUIC** (`with_quic`)
- **WireGuard** incl. AmneziaWG (`with_wireguard`)
- **uTLS fingerprinting** (`with_utls`)
- **Clash API** (`with_clash_api`) — required by NetShift/LuCI dashboards
- **DHCP** (`with_dhcp`)
- Core engine: TPROXY/TUN inbounds, RuleSet (`.srs`), GeoIP/GeoSite, FakeIP, DoH/DoT DNS

### ❌ Stripped (vs the full extended build)

| Tag | Why cut / approx. saving |
| :--- | :--- |
| `with_tailscale` | heavy Tailscale daemon + Derp client (~18 MB) |
| `with_call` | WebRTC voice/video engine (~5 MB) |
| `with_openvpn`, `with_openconnect` | native OpenWrt daemons do this better (~4 MB) |
| `with_masque`, `with_cloudflared`, `with_usbip` | Cloudflare tunnels & USB-over-IP (~5 MB) |
| `with_gvisor` | kernel TPROXY is faster and lighter (~4 MB) |
| `with_acme`, `with_admin_panel`, `with_manager` | built-in web dashboards / SSL issuance (~2 MB) |
| `with_mtproxy`, `with_ccm`, `with_ocm`, `with_trusttunnel`, `with_sudoku`, `with_naive_outbound` | rarely used on routers |

Exact tag list used (identical to the proven set of
[MANCrimSon/sing-box-extended-lite](https://github.com/MANCrimSon/sing-box-extended-lite)):

```
with_quic,with_dhcp,with_wireguard,with_utls,with_clash_api,badlinkname,tfogo_checklinkname0
```

## Pure ELF vs UPX: integration notes

- **Pure ELF** (`*.tar.gz`): zero decompression overhead, mmaps pages from
  flash on demand. Preferred when flash ≥ 64 MB or on x86/extroot.
- **UPX** (`*-compressed.tar.gz`): ~9.5–12.7 MB on flash, but every start
  (and every direct `sing-box version` invocation!) triggers a full in-memory
  unpack — a transient ~35–45 MB RAM spike. Recommended only for 16–32 MB
  flash devices with ≥ 256 MB RAM (or with zram-swap on 128 MB RAM boxes).
  If your tooling polls `sing-box version` frequently, cache the output.

## How the automation works

- Nightly (`cron 0 3 * * *`) and manual (`workflow_dispatch` with
  `force_build` / `custom_tag`) the workflow resolves the **latest stable**
  (non-draft, non-prerelease) release tag of `shtorm-7/sing-box-extended`,
  and skips the build if our repo already has the matching `-lite` release
  (unless `force_build=true`).
- Sources are checked out **unpatched** at the exact upstream tag; the Go
  toolchain version comes from that tag's `go.mod`.
- Release tag = upstream tag + `-lite`; version string inside the binary =
  that tag without the leading `v`.

## License & credits

This repository contains **no sing-box source code** — it only automates
builds of the upstream sources and ships the resulting binaries. The code is
licensed under **GPL-3.0** (see [LICENSE](LICENSE)); the corresponding
source for every release is the exact
[shtorm-7/sing-box-extended](https://github.com/shtorm-7/sing-box-extended)
tag named in the release notes.

- **[shtorm-7/sing-box-extended](https://github.com/shtorm-7/sing-box-extended)**
  — the upstream extended sing-box implementation this project builds.
- **[MANCrimSon/sing-box-extended-lite](https://github.com/MANCrimSon/sing-box-extended-lite)**
  — the original lite distribution: the idea and the reference
  build-and-release workflow (build tags, ldflags, UPX, arch matrix).
- **[EikeiDev](https://github.com/EikeiDev)** — author of
  [OpenWRT-sing-box-extended](https://github.com/EikeiDev/OpenWRT-sing-box-extended),
  whose version-cache pattern inspired the lite-version detection approach.
- **[SagerNet / nekohasekai](https://github.com/SagerNet/sing-box)** —
  sing-box itself.
