# vpp-unifi

Unmodified upstream [VPP](https://fd.io/) built and packaged for
Ubiquiti UniFi gateways.

## Why this exists

UniFi OS 4.x is Debian 11 (bullseye) on arm64 with glibc 2.31 — a
combination nobody publishes VPP packages for, at any version:

- fd.io publishes bullseye packages **amd64 only**; its arm64 builds
  are **Ubuntu only** (jammy/noble).
- Debian has never packaged VPP.
- Substituting the jammy arm64 deb fails outright: its symbol floor is
  GLIBC_2.34 against UniFi OS's 2.31, and bullseye's dpkg can't even
  unpack its zstd members.

Someone has to link VPP against the fleet's libc. This repo is where.

**We ship no patches. This is repackaging, not forking.** Every build
clones upstream `FDio/vpp` at a pinned release tag, asserts the working
tree is clean, and only passes build-config make variables (platform
selection, DPDK driver enable lists). The published manifest records
the exact upstream commit.

## Supported gateways

| Gateway | SoC | Platform | Pin | Status |
|---|---|---|---|---|
| EFG (Enterprise Fortress Gateway) | Marvell CN9670 (CN96xx D0) | `octeon9` | [`pins/efg.toml`](pins/efg.toml) | validated end to end on a reference EFG (full-table FIB sync + readback verify) |

Other UniFi gateways are candidates as need arises — add a pin file,
and the workflow does the rest. The EFG pin's comments carry the full
empirical record of why `platform = "octeon9"` is mandatory there
(short version: DPDK's cnxk PMD hard-fails on the CN96xx D0 stepping
and both DPDK PMD families require NPA mempool ops VPP doesn't
provide, so VPP's **native octeon driver** — built only under
`VPP_PLATFORM=octeon9` — is the only mainline path onto that NIC).

## Releases

Each build publishes under a tag of the form:

```
vpp-<vpp ref>-<platform>-<distro>-<arch>
```

e.g. `vpp-v26.06-octeon9-bullseye-arm64`. Tags are deliberately
gateway-agnostic: two gateways sharing a platform share one artifact.

Release assets:

- `vpp_*.deb` etc. — the packages, flat-named so `sha256sum -c
  SHA256SUMS` works after downloading into one directory
- `vpp-api-json.tar.gz` — the `.api.json` API-definition bundle, for
  binary-API codegen (consumed by
  [unredacted/packetframe](https://github.com/unredacted/packetframe)'s
  `vpp-offload` module); published separately so codegen never
  downloads ~100 MB of .debs
- `manifest.json` — exact upstream commit, platform, driver inventory,
  and the workflow run that produced it
- `SHA256SUMS`

A release only appears after CI has verified the packages in a clean
container of the target distro: glibc symbol floor within the pin's
`glibc_floor`, no unresolved shared libraries, and the Octeon driver
present in the installed tree. Publication ordering guarantees this —
the publish job runs after verification, never alongside it.

Historical note: builds up to 2026-08 were published on
unredacted/packetframe (tags `vpp-v26.06-octeon9-bullseye-arm64`,
`vpp-v22.02-generic-bullseye-arm64` — the AF↔VF mailbox-proof
artifact — and `vpp-v26.06-generic-bullseye-arm64`). Those releases
remain there for link stability; everything new publishes here.

## Installing on a gateway

```sh
gh release download vpp-v26.06-octeon9-bullseye-arm64 -R unredacted/vpp-unifi -D vpp/
sha256sum -c vpp/SHA256SUMS --ignore-missing
```

Four traps, every one of which has cost real time on a router:

```sh
# 1. Mask the service BEFORE install: the deb's postinst starts VPP.
#    (On a re-run, stop it first: systemctl stop vpp.service)
systemctl mask vpp.service

# 2. VPP_INSTALL_SKIP_SYSCTL=1 is MANDATORY. The deb writes
#    vm.nr_hugepages=1024 assuming 2 MB pages; on UniFi's 64K-page
#    kernel (512 MB default hugepages) that is a 512 GiB hugepage
#    request on a 64 GB box, applied at install time.
cd vpp && VPP_INSTALL_SKIP_SYSCTL=1 apt-get install -y ./*.deb

# 3. /tmp is noexec on UniFi OS — stage anything executable under
#    /root, not /tmp.

# 4. Package REMOVAL unbinds VFs from vfio-pci: re-bind after every
#    purge/upgrade before expecting VPP to attach the device.
```

Operational guidance for running VPP under PacketFrame's supervision
(steering, failover, health) lives in packetframe's
`docs/runbooks/vpp-offload.md`.

## Bumping a pin

1. Edit the pin file (e.g. `pins/efg.toml`): change `ref`, optionally
   set `commit` for a strict pin.
2. PR to `main`. The merge triggers `.github/workflows/build.yml`,
   which builds on a free native arm64 runner inside a container of
   the target distro (~45 min), verifies, and publishes the new tag.
3. The workflow skips entirely if the exact artifact is already
   published (the tag is a function of the pin), so accidental
   triggers cost seconds. `workflow_dispatch` offers `force` (rebuild
   anyway) and `publish=false` (build-and-verify only).
4. Consumers upgrade on their own schedule — that separation is the
   point: VPP artifacts change maybe twice a year, consumer releases
   monthly, and rollback must be independent in both directions.

## License

The build scripts and configuration in this repo are
GPL-3.0-or-later (see [LICENSE](LICENSE)). The **published artifacts
are unmodified upstream VPP**, which is licensed Apache-2.0 by the
fd.io project; this repo adds no code to them.
